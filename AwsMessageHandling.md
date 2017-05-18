## Message Handling

Have a look at [the Amazon SQS documentation on how to read a message from an SQS queue](http://docs.aws.amazon.com/sdk-for-net/v3/developer-guide/how-to/sqs/ReceiveMessage.html).

Yes, this tells you how to read messages from a queue. But you are going to want to do much more than that.
for starters, if your system is anything like the ones that I have seen, you need a persistent message handler that sits there waiting for messages, something like

```csharp

public async Task Handler()
{
	while (true)
	{
	  await CheckForMessages();
	}
}

private async Task CheckForMessages()
{
  var receiveMessageRequest = new ReceiveMessageRequest();
  receiveMessageRequest.QueueUrl = myQueueURL;
  var receiveMessageResponse = await sqsClient.ReceiveMessageAsync(receiveMessageRequest);

  foreach (var message in result.Messages)
  {
    await ProcessMessage(message);
  }
}
```

So now you have a long-lived process that you can wrap up in a commandline executable or windows service - this option is less portable but is quite popular anyway. But things have just begun.

You will reach a scale where one EC2 instance can't handle all of the messages. You will have peaks in load. You will want to roll out changes without downtime.
You might find out the wrong way that a single EC2 instance only promises 99.95% uptime and need resilience to keep running though instance issues. 
All of these suggest that you want to run multiple EC2 machines side by side in an autoscaling group, and start new ones and terminate old ones without anything going badly wrong. 
The good news is that will basically work, this arrangement allows machines to share a queue of work items.  The load-spreading doesn't have to be exactly level, but it should be reasonably level.

I would add to that code: 
- A health check http endpoint for the auto scaling group. A simple `return OK;` is fine. If the application fails, this will stop working. 
- Configuration of a suitable maximum number of messages to retrieve at a time. This prevents extreme disparities where one handling machine gets there first and takes all the messages while other machines sit idle.
- a [CancellationTokenSource ](https://msdn.microsoft.com/en-us/library/system.threading.cancellationtokensource) to exit the loop gracefully when a shutdown is received.
- Stats on number of messages received and time taken to process them
- Logging of any errors when processing messages. A `try...catch` block around any message handling, and treat exception as failure.

Errors will lead you on to considering retry policy (how many times and with what interval) and how to configure error queues when this is exhausted, and what to do errors in those queues.

Consider if you want one queue for all messages or one queue per message type. 
The first option does the routing of specific message to code that handles it inside the app, and the second does it in AWS infrastructure. The second requires that the handling program run multiple handing loops side-by-side on different threads.

This leads to how to leverage the multi-threaded advantage that "heavyweight" languages like C# and Java have. Otherwise you night as well use lambdas, free-floating functions - to handle these messages. [As Ian Cooper notes](https://medium.com/altdotnet/on-the-need-for-a-c-renaissance-634078d4e865) "Python, Ruby, and especially Node.JS work well with I/O bound workloads. But all use the process model to scale. If your workload is CPU bound there can be advantages to using a vm like the JVM (or the CLR) to scale."

In the example above, if for example the handler retrieved 10 messages from AWS, why should it use only one thread, and wait until the first 9 are processed before starting processing the tenth one. Maybe the last message is important, unrelated to the other ones, really easy to process quickly but is held up by the others taking longer than usual. After all, using the thread pool is part of the power of the platform, as the joke goes "my threads used to run in [apartments](https://www.codeproject.com/Articles/9190/Understanding-The-COM-Single-Threaded-Apartment-Pa), but now they run in [containers](https://www.docker.com/what-docker)". We can do better than insisting on *n* execution environments to have *n* threads.

But thread management is tricky to get right. Throttling when requests take longer than usual is an issue. There will be configuration parameters such as "max worker threads", "max messages per poll" with no one "right" value, just values suitable to your workload. 

You may be tempted to give up and just use lambdas. Lambdas seem to be simpler and cheaper for some situations, but not all - where the load can be high and response time must be reliable, I would favour running full handler apps instead.

Ordering and idempotency come up, as SQS does not guarantee "exactly once" delivery on most queues. 
