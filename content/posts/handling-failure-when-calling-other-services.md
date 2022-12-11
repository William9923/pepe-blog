---
title: "Solve communication failures between services"
date: 2022-12-11T11:32:38+07:00
draft: false
---

Have you ever found this issue?

```
dial tcp <IPAddress>: cannot assign requested address
```

or any kind of issue due to failures when calling other services? If this issue is what you currently experiencing / curious about, this article is for you...

**TLDR;** Here are the code solution [link (github)](https://github.com/William9923/ping-pong-retryable)

Based on this [stackoverflow](https://stackoverflow.com/a/31810116), 

> The error occurs when we can't assign the requested address during a Dial because it was running out of local ephemeral ports to use for the client connection. 

## Why does it happen?
The reason that we were running out of available ports, is simply that we were making **too many** connections **too fast** (usually happen on high traffic system / highly concurrent system)

When the connection rate is too fast, due to caching on the TCP connection on `http.Get`, it will open another connection (thus eating away another available port). Also, there is a limit on the number of idle connections to a given host; the default is 2. If the current process/session is more than 2 (default), it means that it will **regularly discard connections and open new ones**. Each closed connection will be in `TIME_WAIT` state for two minutes, tying up that connection, and limitting the available port for TCP connection.

Not only that, failures during communication between multiple services / system can come from a variety of factors.
## So, how to solve / mitigate it? 
There are actually quite a lot of solutions out there to solve/mitigate this issue (or any related issue regarding failure when calling other systems/services), we can split it into 2 parts based on my knowledge:

**Receiver:**
- Scale up to be able to handle more request (and faster) 
- etc... (haven't research more)

**Caller(client):**
- Build more resilient client system (with retry and backoff) 
- Use optimized connection pool configuration

In this blog, we will focus on modifying the **Client** side, instead of the receiver, because usually when developing a system that calls to the 3rd party / already released system, we can only change "our" system (caller / client) and are unable to change "their" system (receiver)

### Retrying The Request

Often, trying the same request again causes the request to succeed. This happens because the types of systems that we build don't often fail as a single unit. Rather, they suffer partial or intermittent failures. Retries allow clients to try again and **avoid those partial or intermittent failures**.

But, it's not always safe to retry. A retry can **increase the load** on the system being called (receiver), especially if the system is already failing due to overloading request. To avoid this, we can implement the clients system to use **backoff**. This increases the time between subsequent retries, which keeps the load on the backend even.

Retry with backoff is very useful especially when the traffic doesn't arrive into the receiver services at a constant rate, but has large bursts. These bursts can be caused by client behavior, failure recovery, and even by something simple as a periodic cron job. If errors are caused by load, retries can be **ineffective** if all clients retry at the same time. To avoid this problem, we employ **jitter**. This is a random amount of time before making or retrying a request to help prevent large bursts by spreading out the arrival rate. 

One of the good golang pkg out there that I found is [hashicorp/go-retryablehttp](https://github.com/hashicorp/go-retryablehttp). We can configure easily all important parts for retrying a request. If you are curious, check the code below...
```go
	retryableClient := retryablehttp.NewClient()
	retryableClient.RetryMax = 5
	retryableClient.Backoff = retryablehttp.LinearJitterBackoff
	retryableClient.RetryWaitMin = 1 * time.Second
	retryableClient.RetryWaitMax = 20 * time.Second

	httpClient := retryableClient.StandardClient()
```

Done! Setting up the retryable http client is never more easier. You could check more of the example in the [documentation](https://pkg.go.dev/github.com/hashicorp/go-retryablehttp?utm_source=godoc)

While some of the configuration names are self-explanatory, you might be wondering how to use your own custom backoff algorithm. Don't worry, it's been sorted out by the pkg. Let's say you want to implement a exponential backoff algorithm, you could implement it like this:
```go
// NOTE: Backoff algorithm experimentation
func exponentialBackoff(min, max time.Duration, attemptNum int, resp *http.Response) time.Duration {
	// attemptNum always starts at zero but we want to start at 1 for multiplication
	attemptNum++

	if max <= min {
		// Unclear what to do here, or they are the same, so return min * attemptNum
		return min * time.Duration(attemptNum)
	}

	return time.Duration(math.Pow10(attemptNum-1)) * min
}

retryableClient.Backoff = exponentialBackoff
```

## Retry Alternative (configuring http client connections)
Besides retry, we can also do other ways to avoid connection problems when communicating with other services / system. In Go, we have an interface called `RoundTripper` that executes a single HTTP transaction, returning a Response for the provided Request. The low level implementation that was available out of the box were `&http.Transport{}` (see [godoc](https://pkg.go.dev/net/http#RoundTripper)).

There are 2 configuration that can help (need custom adjustment):
1. MaxConnsPerHost 
2. MaxIdleConnsPerHost

Based on godoc:
```go
// MaxConnsPerHost optionally limits the total number of
// connections per host, including connections in the dialing,
// active, and idle states. On limit violation, dials will block.
//
// Zero means no limit.
MaxConnsPerHost int

// DefaultMaxIdleConnsPerHost is the default value of Transport's
// MaxIdleConnsPerHost.
const DefaultMaxIdleConnsPerHost = 2

// MaxIdleConnsPerHost, if non-zero, controls the maximum idle
// (keep-alive) connections to keep per-host. If zero,
// DefaultMaxIdleConnsPerHost is used.
MaxIdleConnsPerHost int
```

If the receiver has difficulty receiving a large number of requests at once, we can limit the value of `MaxConnsPerHost` so that at the same time, the maximum number of connections don't exceed that values (similar analogy to rate limit).

However, as stated by this [Github Issue](https://github.com/golang/go/issues/16012#issuecomment-224948823), if the receiver often failure because we regularly discarding connections and opening new ones, then we should increase the `MaxIdleConnsPerHost` configuration. Let's say that we had 10 worker and each worker use 10 goroutine (10* 10 = 100). We should set the idle conn to 100 so there are no blocking connection because of `TIME_WAIT` state that tying the connection.

Confused on how to configure it? Simple, just add Transport configuration in the http client, example below:
```go
transportOpts := &http.Transport{
  MaxConnsPerHost:     0,
  MaxIdleConnsPerHost: 5,
}
client := &http.Client{
  Transport: transportOpts,
}
```

## Additional Notes
All code for benchmarking can be see at this [repo](https://github.com/William9923/ping-pong-retryable). It is a simple ping-pong application to explain communication between 2 sevices, and you can look at it's README on how to use it to benchmark the retryable http and how to configure http client connection.

## Reference:
- [AWS Timeout, Retry, and Jitter Guide](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- [Exponential Backoff API Retry](https://docs.aws.amazon.com/general/latest/gr/api-retries.html)
- [Exponential Backoff & Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
