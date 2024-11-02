---
title: SQS - Lambda error hanlding
pageTitleSuffix: sqs-lambda-error-handling
---


### Retries
Lambda retries don't have any impact. If an error is thrown, the batch will be sent back to the queue. Then, depending on the visibility timeout configured in the queue, the messages of the batch will be reprocessed.

What controls the number of retries in this case is the configuration of a DLQ in the queue that sources the lambda. The field `maxReceiveCount` will determine how many times a message will be sent to the lambda. Once that number is reached, the message is sent to the DLQ.

CDK config example:
```typescript
const deadLetterQueue = new Queue(this, 'DeadLetterQueue');
const invocationQueue = new Queue(this, 'InvocationQueue', {
  deadLetterQueue: {
    queue: deadLetterQueue,
    maxReceiveCount: 3,
  }
});

const testFunction = new NodejsFunction(...);
testFunction.addEventSource(new SqsEventSource(invocationQueue, {
  batchSize: 10,
  maxBatchingWindow: Duration.seconds(5),// careful with SQS visibilityTimeout
}));
```

**The message in the DLQ will use the same `messageId`and it will contain the message body.
It would be necessary to review logs, since it doesn't show the response/error thrown.**

### Batch item failure
This configuration will allow to reprocess only those records that failed. To achieve this, configure the event source with `reportBatchItemFailures`.
```typescript
const testFunction = new NodejsFunction(...);
testFunction.addEventSource(new SqsEventSource(invocationQueue, {
  reportBatchItemFailures: true,
  batchSize: 10,
  maxBatchingWindow: Duration.seconds(5),// careful with SQS visibilityTimeout
}));
```
For this to succeed, the ids of the failed records need to be returned in the lambda response. This can be self-managed or delegated to [[#^507c6f|Lambda Powertools Batch]]. For a batch of five messages where two failed, the response should be something like this:
```json
{
  "batchItemFailures": [
    {
      "itemIdentifier": "2nd-item-identifier-uuid"
    },
    {
      "itemIdentifier": "3rd-item-identifier-uuid"
    }
  ]
}
```

![[sqs-item-batch-failures.png]]

---

A curiosity about Lambda Powertools Batch is that an error is thrown only in the situation of all messages failing. Otherwise, if only one of two messages fail, the error would be "silent" unless logged.

![[powertools-batch-logs.png]]

--- 
All the examples are based on processing all records in a batch at least once, even if some failed.
If using Lambda Powertools Batch, the processor `BatchProcessorSync` could be used to process the records sequentially. If self-managed, then loop through each record instead of using `Promise.all()`.
### Resources
- [Detailed explanation of SQS Lambda Batch Item Failure behaviour](https://sodkiewiczm.medium.com/sqs-partial-failures-4df63470506d)
- [Lambda powertools batch item failures](https://docs.powertools.aws.dev/lambda/typescript/latest/utilities/batch/) ^507c6f
- [Serverless Guru lambda integration retries table](https://www.serverlessguru.com/blog/lambda-retry-mechanisms)

