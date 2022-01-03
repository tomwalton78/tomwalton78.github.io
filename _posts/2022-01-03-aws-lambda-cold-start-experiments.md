---
layout: post
title: Experimenting with AWS Lambda Cold Starts
tags: AWS, Lambda, Java
---


AWS Lambda is a great serverless compute platform. It is extremely elastic, scaling up and down to meet your workload as required, so you only pay for the resources you actually need. However, as a consequence of this, it can suffer from slow function start up times: cold starts. In this post you will learn what you can do about this problem, especially in the context of Java applications. You will see how, with a few simple changes, cold start time can be halved, without an increase in costs. It can even be eliminated altogether, if you're willing to accept some limitations.


## What is a cold start?

When a Lambda function is invoked, one of two things can happen. If there is an existing execution environment available, it can simply be re-used and the function can be executed straight away; this is a *warm start*. However, if no such environment is available, then the Lambda service has to spin up a new environment for you. This initialisation involves bootstrapping the runtime and executing any static code you may have, including invoking the constructor in your handler class. Relatively speaking, this takes much longer, and is known as a *cold start*. An execution environment might not be available because there have been no other recent Lambda invocations, because all of your existing environments are busy handling other requests or because you have just updated the code/configuration of your Lambda function.

The figures below show how much longer a cold start may take compared to a warm start for a [sample Java application](https://github.com/tomwalton78/aws-lambda-cold-start-experiments): 15 times longer. The Lambda code simply writes to S3 and DynamoDB, and uses common Java libraries. The trace shows how the cold start time is divided between the initialisation and invocation phases, with the latter being the time taken to actually invoke the handler method for the given request.

![Cold start X-ray example](/images/figures/2022-01-03-cold-start-x-ray-example.png)
*Figure 1 - X-Ray trace for Lambda cold start*

![Warm start X-ray example](/images/figures/2022-01-03-warm-start-x-ray-example.png)
*Figure 2 - X-Ray trace for Lambda warm start*


## When do cold starts matter?

You might be thinking: cold starts only account for a small fraction of my Lambda invocations, so why should I care about optimizing them? And that's fair enough, in some situations. If your Lambda is used for an asynchronous, event-based task, such as processing messages from a queue, it likely doesn't matter if some messages take an extra few seconds to process. However, if your Lambda function backs a synchronous API, with customers waiting on the response, you may care quite a bit if some of them have to wait 10 seconds rather than 100 ms. Alternatively, if your Lambda function experiences a sudden burst of traffic, the majority of Lambda invocations may be cold starts. Let's assume your cold starts are 10 s, warm starts are 100 ms and Lambda concurrency limit is at the default 1000: with all warm starts you'd have 10 K TPS, but with all cold starts only 100 TPS. You may be able to work around this with retries and backoff, but wouldn't it be nice if you didn't have to.


## Why focus on Java?

Java has a particularly hard time with Lambda cold starts, and can be over 10 times slower when compared to other languages such as Python, JavaScript or Rust ([source](https://filia-aleks.medium.com/aws-lambda-battle-2021-performance-comparison-for-all-languages-c1b441005fd1)). This is largely due to loading lots of classes, especially as you add dependencies, resulting in lots of I/O operations. The JVM initialises classes lazily, only when they are first called in the application, resulting in higher latencies during the first invocation within an execution environment. Furthermore, use of reflection prevents certain JVM optimisations, thereby slowing down performance. However, given Java's enduring popularity, and it's strong ecosystem of libraries, it's imperative to find ways to make it work well in Lambda compute environments. As the below experiments will demonstrate, there's a lot you can do to achieve acceptable cold start times for your use case.

## Experiments

The sample Java application used in these experiments, CDK infrastructure code, results, analysis and testing scripts can be found in the GitHub repository: [tomwalton78/aws-lambda-cold-start-experiments](https://github.com/tomwalton78/aws-lambda-cold-start-experiments). The below experiments all use this configuration, unless otherwise stated.


### Experiment 1 - Lambda memory configuration

Perhaps unintuitively, increasing the configured memory for your Lambda function actually increases CPU and network resources proportionally. Lambda memory can be set between 128 MB (default setting) and 10 GB, and so you can get significant improvements in cold start time by simply increasing this parameter, as shown in Figure 3.

![Experiment 1 - Lambda memory configuration graph](/images/figures/2022-01-03-experiment-1-memory-graph.png)
*Figure 3 - Effect of Lambda configured memory on cold start time*

Figure 3 shows that the time taken to execute the invocation phase decreases linearly as memory increases, but only up until just under 4 GB; when memory is doubled, time taken halves. Since you only pay for the invocation phase, not the initialisation phase, and Lambda bills you for the GB-seconds used, this actually means that the extra performance doesn't cost you anything! Therefore, when considering cold starts only, for the sample application it makes no sense to configure anything less than around 4 GB. Many Lambda functions needlessly stick to lower memory configurations, with the mistaken belief that this will always be cheaper. Of course, to properly evaluate the costs you also need to take warm starts into account, with the end result depending on how CPU/memory constrained your code is. [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) is a great tool for performing this cost-benefit analysis automatically for warm starts.

![Experiment 1 - Lambda memory configuration by cost graph](/images/figures/2022-01-03-experiment-1-memory-graph-by-cost.png)
*Figure 4 - Cost per 1M Lambda cold start invocations*

Not only do you not pay for time spent in the initialisation phase, but AWS actually gives you a burst of CPU capacity while in this phase ([source](https://aws.amazon.com/premiumsupport/knowledge-center/lambda-improve-java-function-performance/)). Looking at Figure 3, it's clear that the amount of CPU burst actually increases with higher memory configurations. Although the effects of this on my sample application level off after 4 GB memory, either due to the burst capacity no longer increasing, or simply because the application stops being CPU/memory bound after that point.


### Experiment 2 - Leveraging the CPU burst

Given we get a free burst of CPU power in the initialisation phase, could we move more of the setup work to this phase in order to lower our cold start times? One particularly resource-hungry step in our application is initialisation of classes, via the Guice framework. Sometimes this is done lazily, as part of the handler code. However, Figure 5 shows that if we make sure to initialise all of our classes in the constructor of our Lambda, and so as part of Lambda's initialisation phase, we can shave 5.8 seconds from our cold start times, a 40 % improvement.


![Experiment 2 - Effect of Guice initialisation location on Lambda cold start times graph](/images/figures/2022-01-03-experiment-2-guice-init-location.png)
*Figure 5 - Effect of Guice initialisation location on Lambda cold start times*

At this point, when we profile our Java Lambda cold start with [CodeGuru Profiler](https://docs.aws.amazon.com/codeguru/latest/profiler-ug/what-is-codeguru-profiler.html) (Figure 6), two operations stick out within the invocation phase: calling S3 and DynamoDB. As Stefano Buliani discusses in his excellent [AWS re:Invent talk](https://www.youtube.com/watch?v=ddg1u5HLwg8), the AWS SDK itself lazily loads certain resources, such as request/response marshallers. This means that, although we've initialized our AWS service clients within the constructor, there's still more work that's happening outside of the initialisation phase's CPU burst. One simple, albeit hacky, way of forcing this work to occur within the initialisation phase is to actually make a call to the AWS service from within the constructor, and this is exactly what I've done in Figure 7.

![Amazon CodeGuru profile of Lambda application](/images/figures/2022-01-03-code-guru-profling.png)
*Figure 6 - Amazon CodeGuru profile of Lambda application*

![Experiment 2 - Pre-calling AWS services](/images/figures/2022-01-03-experiment-2-pre-calling-aws-services.png)
*Figure 7 - Effect of pre-calling AWS services on Lambda cold start time*


In order to "pre-call" S3 and DynamoDB, all I did was duplicate the work of writing dummy data to each service, doing it in the constructor in addition to the function handler. With this simple change, which actually increases the total amount of work we need to do, we can shave a further 2.8 seconds off of our cold start times, another 40 % improvement. In practice, simply reading data from a service, even if it doesn't exist, should suffice to "warm up" the necessary code paths. However, don't go too crazy with this trick, as AWS limits the initialisation phase to 10 seconds; if this threshold is exceeded, the entire initialisation phase will be retried, negating any benefits from cold start optimisations ([source](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-context.html)).


### Experiment 3 - Deployment package size

It's all too easy to end up with large JAR files after adding a few dependencies to your Java application. For example, if you were to add the entire AWS Java SDK, your deployment package size would increase by 240 MB; even adding Log4J to your Lambda adds 2 MB. In this experiment I observe the effect of increasing the deployment package size on the Lambda cold start time. I do this by including a single, large file containing random data in the JAR. This file is never read, and so we're only observing the effect of the increased deployment package size, and not the effect of loading more classes at startup.

![Experiment 3 - Deployment package size](/images/figures/2022-01-03-experiment-3-deployment-package-size.png)
*Figure 8 - Effect of deployment package size on Lambda cold start time*

 In an X-Ray trace, time spent pulling the deployment package appears before the initialisation phase, although isn't represented by a specific segment. In the trace in Figure 1 this stage takes less than 0.1 ms, however, as shown by the "pre-init" phase in Figure 8, pulling the deployment package can have a significant impact on cold start time, adding almost 2 seconds once we reach Lambda's 250 MB unzipped package size limit. There are other costs to adding more, large dependencies to your application, namely complexity and maintenance overhead, so it's wise to think carefully before including a new dependency in your project.


### Putting it all together

Now that we've seen some ways to significantly reduce cold start time, let's combine them. Compared to our original Lambda function, let's increase our configured memory from 512 MB to 4096 MB, and also pre-call both S3 and DynamoDB. The original implementation already initialises classes with Guice in the constructor, and has a deployment package size of only 13 MB. As Figure 9 shows, we've gone from 7.5 to 3.2 seconds, a 60 % improvement, with only a marginal increase in cost. In particular, we've slashed the invocation phase from 3.9 seconds to less than 0.1 seconds. 

![Putting it all together graph](/images/figures/2022-01-03-putting-it-all-together-graph.png)
*Figure 9 - Effect of combining all tested optimisations together on Lambda cold start time*

### A note on provisioned concurrency

Lambda's [Provisioned Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html) feature lets us keep a certain number of Lambda functions always warm and ready to respond, in an effort to overcome the cold start problem. However, it is not a silver bullet, otherwise this article would be much shorter, and comes with some important caveats. Firstly, it can cost more, depending on your usage, as you now have to pay for each provisioned instance being kept ready, regardless of whether or not it is being used. Having to predict the correct number of provisioned instances for your application detracts from the core goals of serverless architecture, although auto scaling can reduce this burden.

![Putting it all together with provisioned concurrency graph](/images/figures/2022-01-03-putting-it-all-together-with-provisioned-concurrency-graph.png)
*Figure 10 - Effect of enabling provisioned concurrency on Lambda cold start time*

Figure 10 re-runs the experiments shown in Figure 9, but with provisioned concurrency enabled. Comparing the two figures, we can see that all provisioned concurrency does is eliminate the initialisation phase of the cold start, but does nothing to improve the invocation phase. Therefore even when using provisioned concurrency, you still need to ensure code paths are pre-called as part of the initialisation phase to fully eliminate cold starts. And for any invocations that exceed your provisioned concurrency threshold, you'll still have to deal with the initialisation phase slowing things down.


## Conclusion

This series of experiments has shown that Java Lambda functions can have low cold start times, without throwing more money at the problem, by making a few careful optimisations. In particular, for this sample application, ensuring that all objects that will be needed are constructed as part of Lambda's initialisation phase, and setting an appropriate memory configuration, slashed the cold start time by 3.3 seconds. Applying provisioned concurrency on top of this further decreased the cold start time to just 160 ms.

This post only explored a few ways to improve cold start times. For other directions to explore, I highly recommend watching Stefano Buliani's [AWS re:Invent talk](https://www.youtube.com/watch?v=ddg1u5HLwg8), checking out AWS's own recommendations ([one](https://aws.amazon.com/premiumsupport/knowledge-center/lambda-improve-java-function-performance/), [two](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-1/)) and trying out your own experiments, to see what woks best for your particular application.


## References

- [AWS Lambda execution environment](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-context.html)
- [Operating Lambda: Performance optimization - Part 1](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-1/)
- [Operating Lambda: Performance optimization - Part 2](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-2/)
- [GitHub tomwalton78/aws-lambda-cold-start-experiments](https://github.com/tomwalton78/aws-lambda-cold-start-experiments)
- [GitHub webdevwilson/aws-lambda-maven-cdk](https://github.com/webdevwilson/aws-lambda-maven-cdk)
- [AWS Lambda battle 2021, Aleksandr Filichkin](https://filia-aleks.medium.com/aws-lambda-battle-2021-performance-comparison-for-all-languages-c1b441005fd1)
- [AWS Knowledge Center: lambda-improve-java-function-performance](https://aws.amazon.com/premiumsupport/knowledge-center/lambda-improve-java-function-performance/)
- [AWS Lambda Docs: Memory and computing power](https://docs.aws.amazon.com/lambda/latest/operatorguide/computing-power.html)
- [AWS Lambda Power Tuning, Alex Casalboni](https://github.com/alexcasalboni/aws-lambda-power-tuning)
- [Amazon CodeGuru Profiler](https://docs.aws.amazon.com/codeguru/latest/profiler-ug/what-is-codeguru-profiler.html)
- [AWS re:Invent 2019: Best practices for AWS Lambda and Java](https://www.youtube.com/watch?v=ddg1u5HLwg8)
- [AWS Lambda Docs: Managing Lambda provisioned concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html)

