---
title: "My biggest mistake as junior engineer"
date: 2022-11-08T19:08:50+07:00
draft: false
---

I just completed my probation period in my first full-time job as a software engineer. A lot of things happened during those time. To celebrate it, I would like to share a few mistakes I did from past year and what I would like to do in the future to prevent those things.

## 1. Scared to take on challenging task
These mistakes happen during my internship period during my 6-month internship in my last year as a CS student. As someone who change from a data science role to a software engineer role, I was scared to take on the meaty task. I am scared that I cannot finish the job as well as they expected during my first 3 months, especially after I made many stupid mistakes on my first few tasks. I was feeling really bad about myself and kept thinking that I don't have the correct skillset for the job. Thankfully, I think my manager notice this issue, and he keeps finding ways to make me feel good about myself. He consistently encourage me for the remaining 3 months I was there. I grow from that newbie engineer to one of the top performer engineers in the team (at least that's what I think of myself :D). To this day, I am still very thankful for my senior & mentor (Mr. Agung) because he keeps encouraging me to always learn from my mistakes.

Nowadays, of course, I still feel scared to take on challenging tasks because deadlines, expectations, and changes in the middle of a project will still exist no matter what. But, the goal is to learn from your past self and become better for your future self. That mindset allows me to keep taking on the most challenging project in the team and contribute more to the team and my future self.

## 2. Misunderstood Project Requirements
I think this is the most typical issue with me, along with a ton of other software engineers. As a junior, I like to code. The feeling of creating something is always good. But a lot of the time, I forget that the most important thing is to build a product that `solves` a problem.

Requirements are given to us so we understand what we are trying to solve. Sadly, I usually don't read it as carefully as I should. Looking back, I think a ton of bugs and issues probably could be prevented simply just by reading more carefully about product requirements. Not only that but a ton of changes in the middle of a project could also be prevented if we ask the correct question during the planning phase for the requirements.

I will and still trying to increase my capability to understand project requirements better. As I keep working on more and more projects, I feel like this skill will keep growing in time. 

## 3. Not understanding the technlogy that we use in the project correctly 
During my internship, I am working with Golang & NSQ to make an automated report generator that sends a report of our product line weekly to our partner. And during those time, we had an end-of-year holiday that make us not really notice the metrics. Suddenly, I received a call from the SRE team about how the NSQ message is overflowing. I was shocked by the number of messages being retried because we forgot to whitelist certain service IPs from our microservices. By misunderstanding the nature of the message queue, I make a simple mistake and not preventing the `retry storm` that might occurs due to no max retry attempt in the message queue configuration, causing certain services to be down. Thankfully, the microservice is still new and not affecting other services so no financial loss occurs in that incident (just some complaints from our clients that were waiting on those reports). From that mistake, I swear on myself to learn the concept & idea of message queues in-depth and how to prevent those kinds of things.

Also, I made a lot of mistakes due to my recent projects because I misunderstood the go language feature. As someone who usually uses Python and Javascript, `goroutine` and `channel` concepts are really new to me. I'm not a pro on **distributed system** topics in general, but because the data that need to be processed are quite a lot and it requires me to use those concepts. While there is no fatal bug occurs in production, I think understanding those concepts earlier before starting the project could allow me to make much more readable and maintainable code. From those mistakes, I think understanding language concepts, limitations, and best practices that we use in the project might be a good thing before jumping into project implementation.

## 4. Not finding other hobbies other than coding
When I first started into this industry, I felt a lot of pressure to keep up with the newest tech and to continue learning new things. However, I burnt out quickly because that was all I did.

I took a few steps back and decided to explore doing things I enjoy that weren’t coding-related. I starting learning how to play use vim, I like to read manga & light novel, and I started going to the exercise more.

<br>

Well, I’ve learned a lot during this past year (especially on this last 3 month(s) during my probation period), and hopefully you’ve taken away some lessons from my experiences. I’m thinking to make this kind of post as a yearly routine to reflect on myself. Until then, good bye!
