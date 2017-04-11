# Avoiding simple mistakes in async await

I see a lot of code lately that makes some simple mistakes using the `async ... await` construct in C# code.

## TL;DR

 * A Task is a [promise](https://en.wikipedia.org/wiki/Futures_and_promises) of a value.
 * Most of the time the wait is for network I/O, and [there is no thread waiting](http://blog.stephencleary.com/2013/11/there-is-no-thread.html).
 * Avoid `.Result` as much as possible. Let the async flow.
 * Avoid `Task.Run`. You almost always don't need it.
 * Know when you do need to re-sync.
 * Avoid `async void` methods.
 * In async code, you should await wherever possible.
 
Async can't make your code wait faster, but it can free up threads for greater throughput. However some common antipatterns prevent that from happening.

More and more libraries are going to support async, and in some cases support _only_ async. So we need to get used to doing async, and get used to doing it competently.
 
## A task is a promise
 
 Think of `Task<Order>` as a promise of an order later, i.e. "There will be an order. If I don't have it now, I'll have it at some point in the future". [The Computer Science term is "future" or "promise"](https://en.wikipedia.org/wiki/Futures_and_promises).
 
 Since this is C#, there are a few other things that can happen besides the promised order arriving: The value can arrive, and be `null` instead of an order. Or an exception can be thrown instead.
 
`Async ... await` is basically about allowing you to write code in the familiar linear style, but under the hood, it is reconfigured so that the code after the `await` completes is put in a callback that is executed later. Without `async ... await` we could still write equivalent code using callbacks, but it was just much harder. 

This is not unique to C#. Look at how [this Javascript promises library](https://www.promisejs.org/) works with chained callbacks. And [futures and promises in Java](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html).
 
## There is no thread

A Task - the promise of a value later - is a general construct which says nothing about how the value is being generated. It's usually network I/O, and [there is no thread waiting](http://blog.stephencleary.com/2013/11/there-is-no-thread.html) as there would be for computation.
 
## Avoid taking the result
 
 Most of the time you should not need to use `.Result`. Let the async flow. This means that lots of methods have to be sprinkled with `async`, `Task` and `await`: the caller, the caller's caller and so on up the chain. This is just the cost of it, so get used to it. The model is not so much an "some async methods" in an app as "an async app". 

 Calling `.Result` forces the code to wait at that point, losing the main advantage that you can hand off whole blocks of code to be executed later when results are available.

I heard Kathleen Dollard compare the async call stack to a [Light tube](https://en.wikipedia.org/wiki/Light_tube): Any blockage between the basement and roof will prevent light getting through. And with async, any blocking code will prevent the task from getting through.

 Liberal use of `.Result` is red flag. Try moving the `.Result` up to the calling method. I generally do this while refactoring out `.Result` calls as it's a step towards the goal. It will eventually become apparent if your whole operation can be async or not.
 
 Ideally, you hand the async Task off to your framework. You can do this in ASP MVC and WebApi as they allow [async methods on controllers](http://stackoverflow.com/questions/31185072/how-to-effectively-use-async-await-on-asp-net-web-api), you do this [in NUnit as tests can be async](http://stackoverflow.com/a/21617400/5599), [in NancyFx](https://github.com/NancyFx/Nancy/wiki/Async), OpenRasta, etc. 

 
## Avoid Task.Run
 
Some people have the idea that `Task.Run` is necessary or even generally good for using `async` and `await`. It is not.
  
In async code, you do not need `Task.Run` to start a task since
[The task returned by an async method will be "hot"](http://stackoverflow.com/a/11707546/5599). i.e. already started. The very heavyweight `Task.Run` construct ties up threads and adds nothing of value.

There are times when it can deadlock, and times when it won't. I have seen cases where the calling code was not async, and using `.Result` to exit the async code caused a deadlock. Then `Task.Run` was used and worked. 

But don't mistake this for the right thing - it is an ugly hack with a large performance penalty, and you should far rather look at propagating the `Task` up the call stack to where the framework can handle it.  


If you get a deadlock in code that does follow the correct async patterns, it probably means that you have other problems lurking. Code that is never async or code that is always async tends to not have these problems. Code that tries to be both in different places often does.


## Know when you do need to re-sync

In general async code contains one or more `await` statements, but also lots of synchronous statements that are not awaited. Doing synchronous things in async code is generally safe. Doing asynchronous things in synchronous code is generally dangerous, and should be avoided. 

There is a short list of times when re-syncing is not avoidable.

- ASP.Net filters and child actions must be synchronous. However, [in ASP.NET Core, the entire pipeline is fully asynchronous](http://blog.stephencleary.com/2017/03/aspnetcore-synchronization-context.html). There are no synchronous child actions, so it is best to find another construct to use.

- The `Main` entry point of a console application must be synchronous. You have to do [something like this](http://stackoverflow.com/questions/9208921/cant-specify-the-async-modifier-on-the-main-method-of-a-console-app):
 
```csharp
class Program
{
	static void Main(string[] args)
	{
		var task = AsyncMain();
		task.GetAwaiter().GetResult();
		Console.ReadLine();
	}

	private static async Task AsyncMain()
	{
	  // await the rest of the code
	}
 }  
```


##  Avoid async void methods

The type `async void` is only there for compatibility reasons. When you can, use e.g. `public async Task Foo()` instead of `public async void Foo()`.

There are patterns for converting code to async. e.g. `private bool SomeTest()` becomes `private async Task<bool> SomeTest()`. `bool` becomes `Task<bool>` and `void` becomes `Task` - and where there is no return value we still need a task so that the caller can know when the async code has completed without a return value. You only drop the `Task` when it is necessary for a calling framework that specifically needs a `void` return.

## In async code, you should await wherever possible

I asked the stupid question so that you don't have to: [In server code, if you use one `await` for a slow operation, should you use a second `await` for a fast operation?](http://stackoverflow.com/questions/38118051/should-i-make-a-fast-operation-async-if-the-method-is-already-async). 
And the answer is "yes". If you have one `await` in your code, then you might as well use as many as you can to chop the operation up into small parts so that operations can flow through better. 
So if your code *can* trivially  `await` an async method instead of a calling the sync version, then you should do so. There is no additional performance penalty to this.

[In UI code, there is a guideline that "operations that could take over 50ms should be async" so as to not lock the UI thread](http://blog.stephencleary.com/2013/04/ui-guidelines-for-async.html). But it's clear that this does not apply to web server code where it's all thread pool threads already. 
Even quicker operations should also be awaited in order to break code into smaller chunks and so increase throughput. 

## Further reading

* [Best Practices in Asynchronous Programming](https://msdn.microsoft.com/en-us/magazine/jj991977.aspx)
* [Async Await Best Practices Cheat Sheet](https://jonlabelle.com/snippets/view/markdown/async-await-best-practices-cheat-sheet)
* [There is no thread](http://blog.stephencleary.com/2013/11/there-is-no-thread.html).
* [What is the promise pattern](https://www.quora.com/What-is-the-promise-pattern)

## The catastrophe

This is actual production code:

```csharp
 protected T ExecWithLoggingAndTiming<T>(Func<T> func)
 {
        return Task.Run(async () => await ExecWithLoggingAndTimingAsync(() => Task.FromResult(func()))).Result;
 }
```

As far as I can see:

 * There is a method `protected async Task<T> ExecWithLoggingAndTimingAsync<T>(Func<Task<T>> action)` which wraps the `action` code in timer metrics and error logging.
 * Someone needed a sync version of `ExecWithLoggingAndTimingAsync`, and decided that a sync wrapper was the way to do it. First mistake.
 * When it didn't compile, they  just continued piling on code constructs until it compiled, and called it done. 
 * OK, it probably passed some unit tests as well. 
 * But it's still a catastrophe. People have learned from this "pattern" that if it doesn't compile, just add `Task.Run` and `.Result` until it does, and applied it elsewhere. Unreadable code is poor code. And there is significant unnecessary performance overhead from the heavyweight constructs. 
 * The best way to do this is to actually write a `ExecWithLoggingAndTiming` method that does the same `try...catch` and timers as `ExecWithLoggingAndTimingAsync` but without the Task and awaits. Sometimes the sync code is best as just a wrapper around the async code, but there are simpler wrappers than the code above, and this is not one of those times.

Best refactor:

```csharp
 // The sync version of the code to ExecWithLoggingAndTimingAsync, 
 // i.e. without the awaits 
 // this avoids the overhead and confusion of creating tasks and syncing them up again
 protected T ExecWithLoggingAndTiming<T>(Func<T> func)
 {
   T result = default(T);
   var timer = Stopwatch.StartNew();
   try
   {
     result = func();
     
     timer.Stop;
     LogElapsedTime(timer.Elapsed);
     
     return result;
   }
   catch (Exception ex)
   {
     _logger.Error("ExecWithLoggingAndTiming failed", ex);
     throw;
   }
 }
```
