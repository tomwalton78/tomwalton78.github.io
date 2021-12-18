---
layout: post
title: AWS Lambda-SQS Integration - Scaling Surprises
tags: AWS
---

Using [SQS queues](https://aws.amazon.com/sqs/) with [Lambda functions](https://aws.amazon.com/lambda/) is a very 
commonly used pattern, and for good reason. It's a highly horizontally scalable way of processing messages, allowing 
services to stay decoupled from each other and scale independently. For convenience, [back in 2018](https://aws.amazon.com/blogs/aws/aws-lambda-adds-amazon-simple-queue-service-to-supported-event-sources/) AWS added SQS to 
the list of supported triggers for Lambda, allowing Lambda functions to automatically consume messages 
from SQS queues. However, some aspects of this 
integration don't necessarily 
work how you might expect, which can lead to scaling issues if you're not careful.



## How does it work?

![AWS Lambda-SQS integration diagram](/images/figures/2021-09-21-aws-lambda-sqs-integration-diagram.png)


Lambda polls your SQS queue periodically, calling the `ReceiveMessage` API, in order to check for 
available messages. Whenever some messages are found by the Lambda pollers, a batch of messages (up to 10 in 
size) is sent to a particular Lambda instance for processing. This is done by calling the `Invoke` API, thereby 
synchronously invoking the Lambda service.

Once the Lambda function instance has finished processing the batch of messages, 
Lambda deletes the messages from the SQS queue. If an error is thrown by your Lambda function, then the 
messages are not deleted, and can be picked up again later by the Lambda pollers, once their visibility timeout has 
expired. If a [dead-letter queue](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) (DLQ) is configured on the SQS queue, then these messages will be retried up to a 
maximum number of times, before being sent to the DLQ.


## How does it scale up?

Let's consider what happens when a large batch of SQS messages suddenly 
gets added to the queue. To begin with, there are only a few Lambda poller processes reading from your queue. When 
Lambda finds that there are still messages available even after sending some of them to your Lambda functions, it 
will increase the number of processes reading from your queue, adding up to 60 more instances per minute. This 
continues up until 1000 batches are being processed simultaneously.

At the same time, Lambda functions are being spun up to deal with all the `Invoke` requests being sent to them. Lambda 
functions have an initial burst capacity of up to 3000 concurrent instances (varies by region), and can then add 500 
instances each 
minute. This only occurs up to the relevant concurrency limit, either at the account or Lambda function level.

These two scaling processes are actually independent, and so if the Lambda SQS pollers call `Invoke` on your Lambda 
function, but there are no instances available, and you have reached the configured concurrency limit, then Lambda 
will actually throttle the requests, returning a 429 error. To the Lambda pollers, this looks just like any other 
exception your Lambda function might throw, and the messages are not deleted from the queue. If this happens several 
times to the 
same message, it can end up in your DLQ. 

## When would throttling occur in practice?

It may seem unlikely that the same message would be throttled by Lambda multiple times, but it can easily happen if 
your function takes more than a second or so to process a batch of messages. Alternatively, if you are using 
reserved concurrency and have a lower limit set, throttling can occur very easily. This can catch people out, as 
they assume that they can simply lower the Lambda concurrency limit for their function, and have messages processed 
at a slower rate, whereas in reality this will simply result in lots of DLQ messages.

Even if you have no limits set on 
your function's concurrency, remember the account limit (1000 by default) applies across all Lambda functions at once.
So if you have multiple functions processing messages from multiple queues, it's the total workload that matters, 
not just one queue.

## How to mitigate the throttling issue?

Unfortunately, there isn't a simple solution to mitigate this is problem. You can increase the SQS message 
visibility timeout, so there's a longer gap between message retries; [AWS recommends 6 times the Lambda function 
timeout](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html#events-sqs-queueconfig). Similarly, you could 
increase 
the SQS 
maximum 
receives for a message, so it can be retried more times before being sent to the DLQ; each time Lambda polls your 
message, this counts as 1 receive. Or, you can ask AWS support to increase the Lambda concurrency limit on your 
account.

However, all these solutions simply move the threshold for encountering throttling higher, but don't truly solve the 
issue. To get around this, you'd likely need to stop using the Lambda-SQS trigger. Instead, you could move your 
compute to something like ECS, where you can choose how quickly to pull messages from the SQS queue.


## Conclusion

The SQS Lambda trigger is a welcome addition that makes life much easier for developers. It can horizontally scale 
to a large number of messages per second, and do so very quickly. However, it's not without its 
pitfalls. If your workload might approach the thresholds where throttling can occur, it's worth load testing your 
application. Or, if you need to consume messages from an SQS queue at a controlled rate, then the Lambda-SQS 
trigger likely isn't for you.





## References

- [AWS Docs: Using Lambda with Amazon SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [AWS Docs: Lambda Function Scaling](https://docs.aws.amazon.com/lambda/latest/dg/invocation-scaling.html)


