---
title: "Transaction in Hexagonal Architecture"
date: 2022-12-04T21:39:03+07:00
draft: false
---

**TLDR;** Here are the code example [link (github)](https://github.com/william9923/clean-transaction)

One of the applications of **clean architecture** is [Hexagonal Architecture](https://netflixtechblog.com/ready-for-changes-with-hexagonal-architecture-b315ec967749), an approach that explicitly distinguishes layers, adapters, and so on. This approach has gained love among Go developers because it does not require complex abstractions or intricate patterns and does not contradict complicated language idiom - the so-called Go way. Have you ever used **transactions** in a hexagonal architecture using Go? How do you keep the application layer and database adapter layer separate? Have you occurred a code similar to the like below? 

```go
import (
	"context"
	"database/sql"
)

type UserTxDAO interface {
	GetUser(ctx context.Context, userID uint64, tx *sql.Tx) (model.User, error)
	UpdateUser(ctx context.Context, user model.User, tx *sql.Tx) error
	InsertUser(ctx context.Context, user model.User, tx *sql.Tx) error
}
```

While it works perfectly, it has drawbacks. A transaction will need to be opened and closed with specific commands (BEGIN, COMMIT, or ROLLBACK in SQL) and has a binding to the generated entity - the transaction object (*sql.Tx). Transaction object itself is usually not hovering in the clouds of the global program scoop but explicitly bound to the database connection session over the TCP connection. When opening a transaction we have transaction object that we need to pass to the adapter to perform database operations exactly within this transaction. The information on those transaction objects needs to be known by the database adapter function. 

The easiest solution would be initiate the transaction in the business code and simply passing the transaction object from the business code to the database adapter, but it breaks some rules regarding no infrastructure (database adapter) code in the application layer. This is the most frequent problem that I often see on a lot of codebases that implement database transactions on hexagonal architecture.

If you read till this point, it is safe to say that you probably had faced with similar issue and were interested in tackling this issue. Based on reading through a few StackOverflow, blogs, and Reddit posts, here is the solution that I find the most elegant...

<br>
<br>
<center><h2><strong>CONTEXT</strong></h2></center>
<br>
<br>

Okay, don't get angry just yet. I know that the 1 word above has made a lot of people shake their heads with this post. But, please read it first. At least until the code part.

Firstly, why context? As explained in the Go documentation.
> Contexts should not be stored inside a struct type, but instead passed to each function that needs it.

> At Google, we require that Go programmers pass a Context parameter as the first argument to every function on the call path between incoming and outgoing requests.

While storing data in a context.Context, or as I refer to it - using context values, is one of the most controversy design patterns in Go. Storing values in a context appears to be fine with everyone, but what specifically should be stored as a context value receives a lot of heated discussion in the Go community.

First of all why is context.Value() needed? The short answer to that by using context values, we can easily create both reusable and interchangeable middleware functions that can pass the value that was used in that middleware only. For example, on an HTTP request, we can attach the request UUID and request start time to calculate the latency of those requests.

Back to the topic, so what is the connection between context and transaction? How does it solve the problem stated before? Well after a lot of digging around, I found a pretty good solution...

```go
import (
	"context"
	"database/sql"
)

type txKey struct{}

func InjectTx(ctx context.Context, tx *sql.Tx) context.Context {
	return context.WithValue(ctx, txKey{}, tx)
}

func ExtractTx(ctx context.Context) *sql.Tx {
	if tx, ok := ctx.Value(txKey{}).(*sql.Tx); ok {
		return tx
	}
	return nil
}
```

So, instead of passing around the transaction object around from business logic to the adapter code (thus polutting the business layer with adapter code), we inject the transaction into the context. Here the things, the context that were injected with transaction then could be used in others database adapter code. Look at below code on how I use the transaction injected in the context for the database adapter code.

```go
func (repo userRepo) InsertUser(ctx context.Context, user model.User) error {

	tx := internal_mysql.ExtractTx(ctx)
	now := internal_time.Now().UnixMilli()
	queryValues := []interface{}{
		user.Name,
		user.Balance,
		now,
		now,
	}

	var errDB error
	if tx == nil {
		_, errDB = repo.db.ExecContext(ctx, queryInsertUser, queryValues...)
	} else {
		_, errDB = tx.ExecContext(ctx, queryInsertUser, queryValues...)
	}

	if errDB != nil {
		if errMySQL, ok := errDB.(*mysql.MySQLError); ok {
			return internal_mysql.GetMysqlSpecificError(int(errMySQL.Number), errDB)
		}
		return errDB
	}

	return nil
}
```

Notice that we don't need to pass around the transaction object to the database adapter function because it already passed in the context.

While that solves the issue of separation between business logic and database adapter layer, we still had a lot of code in the business logic that need to call **BEGIN**, **COMMIT**, and/or **ROLLBACK** to start / end the transaction. An example for that could be seen in below code.

```go
func (s *transferService) Transfer(ctx context.Context, param DoTransferParam) error {

	var needRollback bool = false

	ctxWithTrx, err := s.transactionManager.Begin(ctx)
	if err != nil {
		return err
	}
	defer func() {
		if needRollback {
			s.transactionManager.Rollback(ctxWithTrx)
		}
	}()

	users, err := s.userRepo.GetUsersInTransfer(ctxWithTrx, [2]uint64{param.FromUserID, param.ToUserID})
	if err != nil {
		needRollback = true
		return err
	}

	var fromUser model.User
	var toUser model.User
	for _, user := range users {
		if user.UserID == param.FromUserID {
			fromUser = user
		}

		if user.UserID == param.ToUserID {
			toUser = user
		}
	}

	err = s.transferLogsRepo.CreateTransferLogs(ctxWithTrx, fromUser, toUser, param.Amount)
	if err != nil {
		needRollback = true
		return err
	}

	if err = s.depositUserBalance(ctxWithTrx, toUser, param.Amount); err != nil {
		needRollback = true
		return err
	}

	if err = s.withdrawUserBalance(ctxWithTrx, fromUser, param.Amount); err != nil {
		needRollback = true
		return err
	}

	if err := s.transactionManager.Commit(ctxWithTrx); err != nil {
		needRollback = true
		return err
	}

	return nil
}
```

It kinds of messy. Could we simplify it in Go? We could! We can wrap it with a wrapper func that called those command separately, like in the below code.

```go
func (repo transactionManager) WithinTransaction(ctx context.Context, fn dao.TransactionFn) error {
	var needRollback bool = false

	ctxWithTrx, err := repo.Begin(ctx)
	if err != nil {
		return err
	}

	defer func() {
		if needRollback {
			repo.Rollback(ctxWithTrx)
		}
	}()

	if err := fn(ctxWithTrx); err != nil {
		needRollback = true
		return err
	}

	if err := repo.Commit(ctxWithTrx); err != nil {
		needRollback = true
		return err
	}

	return nil
}
```

Now we can define the business logic without the need to think about transaction, like code below...
```go
func (s *transferService) TransferV2(ctx context.Context, param DoTransferParam) error {
	return s.transactionManager.WithinTransaction(ctx, func(trxCtx context.Context) error {
		return s.transfer(trxCtx, param)
	})
}

func (s *transferService) transfer(ctx context.Context, param DoTransferParam) error {
	users, err := s.userRepo.GetUsersInTransfer(ctx, [2]uint64{param.FromUserID, param.ToUserID})
	if err != nil {
		return err
	}

	var fromUser model.User
	var toUser model.User
	for _, user := range users {
		if user.UserID == param.FromUserID {
			fromUser = user
		}

		if user.UserID == param.ToUserID {
			toUser = user
		}
	}

	err = s.transferLogsRepo.CreateTransferLogs(ctx, fromUser, toUser, param.Amount)
	if err != nil {
		return err
	}

	if err = s.depositUserBalance(ctx, toUser, param.Amount); err != nil {
		return err
	}

	if err = s.withdrawUserBalance(ctx, fromUser, param.Amount); err != nil {
		return err
	}
	return nil
}
```

Now, we got:
- simple mechanism for transaction execution in terms of the business logic layer
- isolation of levels, abstractions do not leak
- no reflection, all transaction work is typed and fault-tolerant
- clean repository methods, no need to add a transaction to a signature
- query methods are transaction agnostic - if there is a transaction, they are executed within it, if not - directly on the database
- commit and rollback are executed automatically according to the function execution result. No deferring
- in case of panic, rollback will be executed inside tx.Close()

But, I still try to make this clear in my examples, but despite that I want to explicitly state that context.Value() should **NEVER** be used for values that are not created and destroyed during the lifetime of the request. You shouldn’t store a logger there if it isn’t created specifically to be scoped to this request, and likewise you shouldn’t store a generic database connection in a context value. In this specific example, this could be done because the context lifetime only used during the lifetime of the business unit of work (see below code). The context should not be used or passed for any other business logic.

You could checkout out my [Github](https://github.com/william9923/clean-transaction) to find more example. Also, if you are interested on where I found the idea from, you could use these links below. I got a ton of inspiration from those reference:
- [Hexagonal Architecture](https://netflixtechblog.com/ready-for-changes-with-hexagonal-architecture-b315ec967749)
- [SQL Transaction](https://www.sohamkamani.com/golang/sql-transactions/)
- [Clean Transaction in Hexagonal Architecture](https://www.kaznacheev.me/posts/en/clean-transactions-in-hexagon/)
- [Using Context in Go](https://www.calhoun.io/pitfalls-of-context-values-and-how-to-avoid-or-mitigate-them/)
