---
layout: post
title: A Brief Summary of Retry Strategies
tags: API, Retry Strategy, Exponential Backoff, Token Bucket, Linear Backoff, Reliability
---

The goal of this post is to help you pick the right retry strategy for your use case. This will enable you to overcome transient API request failures, without causing wider system failure. We will first consider what happens with no retry strategy, compared with using a simple linear backoff strategy. Then we will see how adding jitter to the exponential backoff strategy can solve the thundering herd problem. Finally, we will solve the retry storm problem by using the token bucket retry condition.


## Introduction

Useful software systems are often complex. To get work done, they need to talk to other services. This typically occurs as an API request over a network, where all sorts of things can go wrong. Servers crash, network packets get lost, compute resources are depleted, and so on. While we try to minimise the occurrence of errors as much as possible, it's impractical to eliminate them all. This means we must design our systems to handle errors gracefully. Fortunately, many errors are transient, and so we can improve the reliability of our service by retrying failed requests. Such errors include HTTP 408 (request timeout), 429 (too many requests) and 5XX (server error) response codes.

However, we should only retry API requests if we are certain it is safe to do so. This means that any API request with side effects, such as resource creation, should be [idempotent](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/). This guarantees that the side effects only happen once, regardless of how many times the API is called.

Furthermore, retries are selfish. They increase the chance that a given request succeeds, but at the cost of increasing the load on both your own service, and the service you are calling. They also increase the response time of the operation you are performing. If a downstream service has returned an error because it is struggling to deal with the current load, sending lots of retry requests will only make things worse. If a user is waiting on a request, it doesn't matter if the request succeeds on the 5th retry 10 seconds later, because the user won't be there anymore. For these reasons it is critical to consider the context in which the service is used when deciding upon the most suitable retry strategy.

## Strategy 1: Don't Retry

To start with let's keep things simple and not use any retry strategy. Transient failures in dependencies will be returned as errors in our own service. This could mean a message goes to a dead-letter queue, or a user sees an error message on a UI. One benefit is that we reduce resource usage in both our service, and the service we are calling, with no chance of retries overwhelming the downstream service. If retries are rare, this simple approach may be adequate. However, in most systems this reduced reliability would cause a poor user experience, and wasted time spent debugging transient issues.

## Strategy 2: Linear Backoff

A simple strategy is to wait a fixed amount of time before trying a request again. We can do this multiple times up to a set number of retries, or a timeout threshold. This greatly increases reliability, but it can be difficult to tune the wait time appropriately. Too small and the transient error is less likely to be resolved, and the burst of traffic could overwhelm downstream services. Too long and the total response time of the operation increases. Limited server resources such as memory, threads and connections are held for longer, reducing the total throughput of the system.

## Strategy 3: Exponential Backoff

We want to resolve transient issues as quickly as possible, without overwhelming the downstream API. Therefore it would be ideal if we could have a very short wait time between retries to begin with. And, if this first retry fails, we can use progressively longer gaps between retries. Eventually, after a certain time threshold, we would give up. This is exactly what exponential backoff gives us. Considering the exponential formula: `y = a * b^n`, we can tune the initial retry wait time, *a*, and the scaling factor, *b*, to change the retry wait time, *y*, for a given attempt, *n*. For example, Figure 1 shows us how the wait time grows if we wait 10 ms at first, and then quadruple the wait time on each attempt.

![Exponential Backoff Graph](/images/figures/2022-03-01-exponential-backoff-graph.png)
*Figure 1 - Exponential backoff function for a=10, b=4*

### The Thundering Herd Problem

Exponential backoff is all well and good when we're only worrying about one client. However, in practice we likely have hundreds of threads making API calls. Furthermore, traffic often comes in bursts, rather than being evenly distributed over time. This could be due to an upstream batch process, a scheduled job or a news event that triggers an influx of users. As Figure 2 illustrates, this leads to lots of requests at certain intervals, and then large gaps between these intervals. This is a very inefficient use of resources.

![Thundering Herd Problem Graph](/images/figures/2022-03-01-thundering-herd-problem-graph.png)

*Figure 2 - Distribution of requests when using exponential backoff retry strategy. 100 requests, a=10, b=4, 1 % randomness factor applied to wait times to simulate real-world variability*. Bucketing in 20 ms increments.

### Adding Jitter

A common solution is to [add jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) to the wait time. This adds randomness to the wait times, to spread the retries out more evenly over time. The underlying exponential behaviour is still there, but sharp peaks from bursts of traffic are smoothed out. For example, in Figure 3 we apply a random scaling factor between 0.5 and 1.5 to the wait time. For the final retry attempt, this reduces the maximum concurrent requests from 35 in Figure 2, to 5 in Figure 3.

![Thundering Herd Solution Graph](/images/figures/2022-03-01-thundering-herd-solution-graph.png)

*Figure 3 - Distribution of requests when using exponential backoff retry strategy. 100 requests, a=10, b=4, 50 % randomness factor applied to wait times to simulate jitter. Bucketing in 20 ms increments.*

## Retry Storms

Exponential backoff with jitter works fine when considering a single service in isolation. However, often a single API call made by a user results in a chain of API calls downstream. Let's think about what happens if all these services implement retries, and the final service in the chain is starting to fail due to a temporary spike in load.

![Retry Storm problem](/images/figures/2022-03-01-retry-storm-problem.png)
*Figure 4 - Four service calls in a chain*

When service E fails, service D will retry the request up to its retry limit. If all these retries fail, service C will retry the request to service D up to its limit, and so on. It's clear that the retry behaviour is exponential in nature: `y = x^n`, where the total number of calls, *y*, is the number of calls in each service, *x*, raised to the power of the number of service calls in the chain, *n*. Note that *x* is equal to the number of retries + 1. For example, with 2 retries per service call, and 4 service calls in the chain, that's 81 API calls to service E. And remember, this is per request to service A. So if there's a spike of even 1000 requests to service A, service E will get bombarded with 81,000 requests. This is the difference between service E throttling a few incoming requests, then dealing with them a short while later; and total failure of service E.

### Retry Throttling

We have seen how retries can be useful when failures are rare, but they can wreak havoc when failures are common. To solve this problem, it would be good to have some sort of bi-modal behaviour: use retries freely when failures are infrequent, but aggressively limit retries when failures become frequent. This is what the Token Bucket Retry Condition gives us, [used in the AWS SDK](https://sdk.amazonaws.com/java/api/latest/software/amazon/awssdk/core/retry/conditions/TokenBucketRetryCondition.html). This is how it works:

1. We start with a bucket containing, for example, 10 tokens.
2. A successful retry does not consume any tokens from the bucket.
3. A failed retry attempt consumes 1 token.
4. When the bucket is empty, we cannot perform any retries. However, we can still perform the initial request.
5. Only successful requests may refill the bucket. For example, we might refill 1 token for every 10 successful requests, up to the bucket's maximum capacity.

![Retry Storm problem](/images/figures/2022-03-01-token-bucket-retry-condition.png)
*Figure 5 - Illustration of how failed and successful requests remove and add tokens to bucket when using Token Bucket Retry Condition*

This means that when the bucket is empty, only the first API request is sent out, but no retries. This solves the retry storm problem, as the equation for the total number of retries becomes `y = 1^n`, which is always 1, regardless of the number of services in the chain.

It's important to note that the retry behaviour is now stateful, thereby increasing the complexity. Let's say we have 10 servers in our service, and we store the token bucket state in memory on each one. It will take 100 retry failures across the service to empty all buckets; with 20 servers, 200 failures. Therefore you should take the number of compute environments into account when configuring the parameters for the bucket. This is especially true when using Function-as-a-Service compute environments such as AWS Lambda. Here, you can have thousands of concurrent compute environments, and there are no guarantees on how long the state in each one will be retained for.

Another related solution is the Circuit Breaker pattern, [as explained by Microsoft](https://docs.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker). The key difference is that it blocks *all* API requests during its failure mode, rather than retries only. This can extend the time to recovery.

## Conclusion

We have seen how a simple linear backoff retry strategy helps us to recover from transient issues, but has its limitations. The exponential backoff with jitter strategy lets us recover quickly, while avoiding the thundering herd problem. Adding a token bucket retry condition can then mitigate the retry storm problem. These last retry strategies are certainly more complex, but they offer a strong return on investment. The key point is that you should choose the right retry strategy for your particular use case. It's worth emphasising that retries are selfish. You are trading your limited system resources for increased reliability and a better user experience.


## References

- [Amazon Builder's Library: Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- [Software Reliability, Jiantao Pan](https://users.ece.cmu.edu/~koopman/des_s99/sw_reliability/)
- [Amazon Builder's Library: Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- [Google Cloud Docs: Retry Strategy](https://cloud.google.com/storage/docs/retry-strategy)
- [AWS Architecture Blog: Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Wikipedia: Thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem)
- [AWS Developer Tools Blog: Introducing Retry Throttling](https://aws.amazon.com/blogs/developer/introducing-retry-throttling/)
- [AWS SDK: TokenBucketRetryCondition](https://sdk.amazonaws.com/java/api/latest/software/amazon/awssdk/core/retry/conditions/TokenBucketRetryCondition.html)
- [Microsoft Azure Docs: Circuit Breaker pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
