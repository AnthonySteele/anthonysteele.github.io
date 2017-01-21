# Avoiding simple mistakes in async await

I see a lot of code lately that makes some simple mistakes using the `async ... await` construct in C# code.

## TL;DR

 * A Task is a [promise](https://en.wikipedia.org/wiki/Futures_and_promises) of a value.
 * Most of the time the wait is for network I/O, and [there is no thread waiting](http://blog.stephencleary.com/2013/11/there-is-no-thread.html).
 * Avoid `.Result` as much as possible. Let the async flow.
 * Avoid `Task.Run`. You don't need it.
 * Avoid `async void` methods
 
Async can't make your code wait faster, but it can free up threads for greater throughput. However some common antipatterns prevent that from happening.
 
## A task is a promise
 
 Think of `Task<Order>` as a promise of an order later, i.e. "There will be an order. If I don't have it now, I'll have it at some point in the future". [The Computer Science term is "future" or "promise"](https://en.wikipedia.org/wiki/Futures_and_promises).
 
 Since this is C#, there are a few other things that can happen besides the promised order arriving: The value can arrive, and be `null` instead of an order. Or an exception can be thrown instead.
 
`Async ... await` is basically about allowing you to write code in the familiar linear style, but under the hood, it is reconfigured so that the code after the `await` completes is put in a callback that is executed later. Without `async ... await` we could still write equivalent code using callbacks, but it was just much harder. 

This is not unique to C#. Look at how [this Javascript promises library](https://www.promisejs.org/) works with chained callbacks. 
 
## There is no thread

A Task - the promise of a value later - is a general construct which says nothing about how the value is being generated. It's usually network I/O, and [there is no thread waiting](http://blog.stephencleary.com/2013/11/there-is-no-thread.html) as there would be for computation.
 
## Avoid taking the result
 
 Most of the time you should not need to use `.Result`. Let the async flow. This means that lots of methods have to be sprinkled with `async`, `Task` and `await`: the caller, the caller's caller and so on up the chain. This is just the cost of it, so get used to it. It's not so much an "some async methods" in an app as "an async app". 

 
 If you have to use `.Result`, use it as few times as high up the call stack as possible.
 
 Ideally, you hand the async Task off to your framework. You can do this in ASP MVC and WebApi as they allow [async methods on controllers](http://stackoverflow.com/questions/31185072/how-to-effectively-use-async-await-on-asp-net-web-api), you do this [in NUnit as tests can be async](http://stackoverflow.com/a/21617400/5599), [in NancyFx](https://github.com/NancyFx/Nancy/wiki/Async), etc. 
Calling `.Result` forces the code to wait at that point, losing the main advantage that you can hand off whole blocks of code to be executed later when results are available.

I heard Kathleen Dollard compare the async call stack to a [Light tube](https://en.wikipedia.org/wiki/Light_tube): Unless the tube from the basement is connected to the sunlight on the roof in an uninterupted way, no light will get through. And unless the task from the inner method is handed back to the framework, the app is not async.
 
 The most common exception is for commandline apps, where since `Main` cannot be async, you have to do something like this:
 
```csharp
    class Program
    {
        static void Main(string[] args)
        {

            var task = AsyncMain();
            task.Wait();
            Console.ReadLine();
        }

        private static async Task AsyncMain()
        {
          // await the rest of the code
        }
     }  
```
 
## Avoid Task.Run
 
You really don't need this in most cases (unless you're writing a scheduling engine like the one in [JustSaying](https://github.com/justeat/JustSaying)). 
[The task returned by an async method will be "hot"](http://stackoverflow.com/a/11707546/5599). i.e. already started and the very heavyweight `Task.Run` construct ties up threads and adds nothing of value.


##  Avoid async void methods

This type is only there for compatibility reasons. When you can, use e.g. `public async Task Foo()` instead of `public async void Foo()`.

There are patterns for converting code to async. e.g. `private bool SomeTest()` becomes `private async Task<bool> SomeTest()`. `bool` becomes `Task<bool>` and `void` becomes `Task` - and where there is no return value we still need a task so that the caller can know when the async code has completed without a return value. You only drop the `Task` when it is necessary for a calling framework that specifically needs a `void` return.

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
        return Task.Run(async () => await DoTheThingAsync(() => Task.FromResult(func()))).Result;
 }
```

As far as I can see, 
* There is a method `protected async Task<T> DoTheThingAsync<T>(Func<Task<T>> action)` which wraps the `action` code in timer metrics and error logging.
* Someone needed a sync version of `DoTheThingAsync`, and decided that a sync wrapper was the way to do it. First mistake.
* When it didn't compile, they  just continued piling on code constructs until it compiled, and called it done. 
* OK, it probably passed some unit tests as well. 
* But it's still a catastrophe. People have learned from this "pattern" that if it doesn't compile, just add `Task.Run` and `.Result` until it does, and applied it elsewhere. Unreadable code is poor code. And there is significant unnecessary performance overhead from the heavyweight constructs. 
* The best way to do this is to actually write a `DoTheThing` method that does the same `try...catch` and timers as `DoTheThingAsync` but without the Task and awaits. Sometimes the sync code is best as just a wrapper around the async code, but there are simpler wrappers than the code above, and this is not one of those times.

Best refactor:

```csharp
 // The sync the code to DoTheThingAsync, 
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
     _logger.Error("DoTheThing failed", ex);
     throw;
   }
 }
```
