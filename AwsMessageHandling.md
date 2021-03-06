## AWS Message Handling

## The Basics

Have a look at [the Amazon SQS documentation on how to read a message from an SQS queue](http://docs.aws.amazon.com/sdk-for-net/v3/developer-guide/how-to/sqs/ReceiveMessage.html).

Yes, this tells you how to read messages from a queue by calling `sqsClient.ReceiveMessage` or the async version, once. But the gap between that example and a running system like the ones that I have seen is large. This article is all about that gap. 

## An Application that listens

For starters, you need a persistent message handler that runs all the time, listens  for messages and handles them when they arrive. This is something like:

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
  var request = MakeReceiveMessageRequest();
  var response = await sqsClient.ReceiveMessageAsync(request);

  foreach (var message in response.Messages)
  {
    await ProcessMessage(message);
  }
}
```

So now you have a long-lived process that you can wrap up in a commandline executable or a windows service - the second option is less portable but is easy to manage, [often using TopShelf](http://topshelf-project.com/). Run it in AWS and messages will be processed all the time. But things have just begun.

## Scaling Out

You might reach a scale where one EC2 instance can't handle all of the messages. You will have peaks in load. You will want to deploy changes without downtime.
You might find out the hard way that a single EC2 instance only promises 99.95% uptime and so you need resilience to keep running though the inevitable issues. 

All of these suggest that you want to run multiple EC2 machines side by side in an [Auto Scaling Group](http://docs.aws.amazon.com/autoscaling/latest/userguide/AutoScalingGroup.html), and start new ones and terminate old ones without anything going badly wrong. 

The good news is that if you run the code above on multiple machines, it will basically work, and it allows machines to share a queue of messages and do more-or-less equal amounts of work: e.g. if 25 messages arrive all at once, and there are 3 machines each asking for 10 messages each in rapid succession, then the first and second machine will get 10 messages each, and the third machine will pick up the last 5. 

The load-spreading doesn't have to be exactly level, but it should be reasonably even. In that example, what we don't want is the first machine configured to pick up all 25 messages and then spend 25 seconds working through them while the other machines are idle.

I would add to that code: 
- [A health check HTTP endpoint](https://skeltonthatcher.com/blog/http-healthchecks-for-a-resilient-platform/) for the Auto Scaling Group. A simple `return OK;` is fine. If the application cannot start or fails entirely, then the ASG will notice.
 
- Configuration of a suitable maximum number of messages to retrieve at a time. This prevents extreme disparities where one handling machine gets there first and takes all the messages while other machines sit idle.

- a [CancellationTokenSource ](https://msdn.microsoft.com/en-us/library/system.threading.cancellationtokensource) and `CancellationToken` used to exit the loop gracefully when an application shutdown happens. i.e. the while loop becomes `while (! ct.IsCancellationRequested)` and [the same token is passed on to ReceiveMessageAsync ](http://docs.aws.amazon.com/sdkfornet/v3/apidocs/items/SQS/MSQSSQSReceiveMessageAsyncReceiveMessageRequestCancellationToken.html)

- Send statistics on number of messages received and time taken to process them.

- Logging of any errors when processing messages. A `try...catch` block around any message handling, and treat exception as failure.

Errors will lead you on to considering retry policy (how many times should it retry, and with what interval) and how to configure error queues when the retry is exhausted, and what to do errors in those queues.

## Architecture and Queue Topology

Consider if you want one queue for all messages or one queue per message type. 

The "one queue for all" option does the routing of specific message to code that handles it inside the application, and the second does it in AWS infrastructure. 

The first means that there is only one message-receiving loop. And a shared error queue. In practice, it makes issues with messages in the error queue harder to diagnose, as this error queue can accumulate different kinds of messages.

The "queue per message type" option requires more configuration, and that the handling program run multiple handing loops side-by-side on different threads.  Since they all share the same thread pool, if you want to limit messages in progress by keeping track of threads, this should be done once across all message loops.

## Queue Topology and Threads

If you have 10 messages to process, why process them one at a time when you can use the thread pool to launch 10 worker threads and process them in parallel, while immediately going back to listening for more messages? When do you stop asking for more messages?

You might think that you want to run at most "one thread per core" but it depends on what the message handlers actually do. If they retrieve values from HTTP apis or databases, as is typical, using `async` might result in much lighter load per handler if they spend a significant fraction of the time waiting for responses.

This leads on to how to leverage the multi-threaded advantage that "heavyweight" languages like C# and Java have. Otherwise you night as well use lambdas to handle these messages. These are free-floating functions where effectively AWS is handling the scaling and threading issues.

But, [as Ian Cooper notes](https://medium.com/altdotnet/on-the-need-for-a-c-renaissance-634078d4e865) "Python, Ruby, and especially Node.JS work well with I/O bound workloads. But all use the process model to scale. If your workload is CPU bound there can be advantages to using a vm like the JVM (or the CLR) to scale."

In the code above, if for example the handler retrieved 10 messages from AWS, it only uses one thread, and will wait until the first 9 are processed before starting processing the tenth one. Maybe the last message is important, unrelated to the other ones, really easy to process quickly but is held up by the others taking longer than usual. After all, using the thread pool is part of the power of the platform, as the joke goes "my threads used to run in [apartments](https://www.codeproject.com/Articles/9190/Understanding-The-COM-Single-Threaded-Apartment-Pa), but now they run in [containers](https://www.docker.com/what-docker)". We can do better than insisting on *n* execution environments in order to have *n* threads.

You may be tempted to give up and just use lambdas. Lambdas seem to be simpler and cheaper for some situations, but not all: Where the load can be high and response time must be reliable, I would favour running full handler apps instead.

Ordering and idempotency come up, as SQS does not guarantee "exactly once" delivery on most queues. 

But the thread management issues in an application are tricky to get right.  There will be configuration parameters such as "max worker threads", "max messages per poll" with no one "right" value, just values suitable to your workload.  

Throttling when requests take longer than usual is an issue. There are similarities between a doubling in the number of incoming messages and a doubling in the time taken to process a message - both will result in doubling in the average number of messages being processed at any one time. 

I don't believe that there is a definitive, general and robust library for this yet. But these are the issues that I have come across.
