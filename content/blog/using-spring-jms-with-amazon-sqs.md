+++
date = "2021-05-04"
title = "Using Spring JMS with Amazon SQS"
type = "post"
ogtype = "article"
+++

In this blog post, I'll try to explain what you need to be careful about when using Spring JMS for consuming messages from [Amazon SQS](https://aws.amazon.com/sqs/).

Even though there are many blog posts about this topic, I noticed that most of them don't even mention how to handle failures properly.

I'm not sure if it's because blog posts are poor-quality, or the consumers of them are just very lazy that they're trying to build production systems based on random blog posts with zero questionings. Anyway...

Alright. So, you have an SQS queue that you want to consume and process messages. This message processing might involve some external systems so you should be prepared for failures during this message processing. SQS provides some nice features for you to handle these failures.

[**Visibility Timeout**](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html): SQS has out-of-the-box support for retries with fixed delays. If you don't delete the message you consumed within this period, that message will be visible again to the consumers. Hence, retry mechanism with a fixed delay. You can configure the maximum number of retries with [Redrive Policy](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SetQueueAttributes.html).   
What happens if you still can't process message successfully and can't delete the message from the queue after all these retries?

[**Dead-letter Queues**](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html): The messages that you couldn't process within `(maxReceiveCount * visibilityTimeout)` seconds, will be sent to its dead-letter queue automatically by SQS, so that you can later on examine these messages you could not process and try to find out the root cause for the failures.

"Great!" you think, SQS already does a lot of stuff for handling failures, you just need to integrate Spring JMS with Amazon SQS, and then you can start writing message processing logic.

So you found [this](https://aws.amazon.com/blogs/developer/using-amazon-sqs-with-spring-boot-and-spring-jms/) blog post for this integration and realize that SQS even has its own [JMS library](https://github.com/awslabs/amazon-sqs-java-messaging-lib) which you can integrate with Spring JMS, nice! 

When you apply the very same config in the blog post above, if you have an error and your JMS Listener throws an exception during message processing, you'll see that visibility timeout is not respected at all, and retry kicks in immediately. In [this GitHub issue](https://github.com/awslabs/amazon-sqs-java-messaging-lib/issues/75), you can find why that's happening.

Basically, when you use `CLIENT_ACKNOWLEDGE` mode, if your listener method throws an exception, SQS JMS library changes that message's visibility timeout to 0, so that message becomes visible immediately. This is not what we want, we all know that immediate retries often trouble rather than a cure for any problem.

It's clear that we need to choose another acknowledge mode.

### JMS Acknowledge Modes and SQS Implementations

These acknowledge modes come from the original JMS spec and client libraries (SQS JMS library, in this case) are responsible to implement them. Let's have a look at our options and implementations of it.

There are 3 acknowledge modes in the [JMS spec](https://github.com/javaee/jms-spec/blob/master/jms1.0.1a/src/share/javax/jms/Session.java#L116): `AUTO_ACKNOWLEDGE`, `CLIENT_ACKNOWLEDGE`, `DUPS_OK_ACKNOWLEDGE`

And there's 1 acknowledge mode SQS JMS library provides: [`UNORDERED_ACKNOWLEDGE`](https://github.com/awslabs/amazon-sqs-java-messaging-lib/blob/master/src/main/java/com/amazon/sqs/javamessaging/SQSSession.java#L112)

To find out their SQS implementations, we need to look at [SQSConnection](https://github.com/awslabs/amazon-sqs-java-messaging-lib/blob/master/src/main/java/com/amazon/sqs/javamessaging/SQSConnection.java#L181) class:

{{< gist sedooe edaa8e3938e0e3fda2c4a83d35817e4d >}}

Let's see these implementations one by one:

1. [AutoAcknowledger](https://github.com/awslabs/amazon-sqs-java-messaging-lib/blob/master/src/main/java/com/amazon/sqs/javamessaging/acknowledge/AutoAcknowledger.java): The implementation of `AUTO_ACKNOWLEDGE`.

{{< gist sedooe e36564813b0fc9240b92d369a50d1556 >}}

As explained both in AutoAcknowledger and `AUTO_ACKNOWLEDGE` JavaDoc, this acknowledges the message (deletes it from the queue) when it first received it. So, even though your processing ends up with an exception, you'll never get that message again from SQS. If you failed the process of this message, goodbye, it's lost. Thus, if you can't tolerate losing messages and want to leverage SQS' visibility timeout, do not use this acknowledge mode.

2. [RangedAcknowledger](https://github.com/awslabs/amazon-sqs-java-messaging-lib/blob/master/src/main/java/com/amazon/sqs/javamessaging/acknowledge/RangedAcknowledger.java): The implementation of `CLIENT_ACKNOWLEDGE` and `DUPS_OK_ACKNOWLEDGE`.

{{< gist sedooe 89ea89eb45994a069c2b850289d0e3c5 >}}

In this implementation, `acknowledge` deletes all messages within the `unAckMessages` queue it holds. Invocation of this `acknowledge` is done by Spring JMS from its `commitIfNecessary` method, if message listener doesn't throw any exception. You need to be careful when using this acknowledge mode. If you use the solution above to not change visibility timeout to 0, you'll end up removing `rollbackOnExceptionIfNecessary` implementation. Doing that may cause failed messages to be acknowledged. So, if you want to use fixed visibility timeout on the SQS queue you set and you can not tolerate losing messages, you should not use `CLIENT_ACKNOWLEDGE` mode either.

3. [UnorderedAcknowledger](https://github.com/awslabs/amazon-sqs-java-messaging-lib/blob/master/src/main/java/com/amazon/sqs/javamessaging/acknowledge/UnorderedAcknowledger.java): The implementation of `UNORDERED_ACKNOWLEDGE`.

{{< gist sedooe 4647dfb6e9e5370a3fbfad63fd9df751 >}}

As you see, this also deletes a single message just like `AutoAcknowledger`, except `acknowledge` method is not called by Spring JMS or SQS JMS libraries. If you're using this mode, you're the one who is responsible for calling the `acknowledge`.

All of these acknowledge modes do things very differently and you should choose the one that works for your use case. There is a number of things to consider, some of them:

- Can you tolerate losing messages?
- Is your message processing idempotent? Does the order of messages important for you?
- What's the volume of messages you consume? Since SQS charges you per call, you may end up with 10x more costs if you don't use `CLIENT_ACKNOWLEDGE`. (hint: take a look at [AmazonSQSBufferedAsyncClient](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/sqs/buffered/AmazonSQSBufferedAsyncClient.html))
- Is your message processing asynchronous? If it is, you should be in charge of the calling `acknowledge` method, so you better not use anything other than `UNORDERED_ACKNOWLEDGE`.
 
 My suggestion is, if you want to benefit from visibility timeout and dead-letter queues, go with `UNORDERED_ACKNOWLEDGE` by making sure that you `acknowledge` the message manually. 
 
 Nevertheless, as with anything in software development, there's no silver bullet in this case too. You should know your tools/libraries well and be aware of the trade-offs you face.
