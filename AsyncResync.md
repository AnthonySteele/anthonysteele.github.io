# Resynchronising async code

After writing "[Avoiding simple mistakes in async await](./AsyncBasicMistakes)" and revising it many times, 
it becomes inescapable that some of the uses and abuses of async code are not simple, particularly about breaking out of async and preventing deadlocks.

In order to understand async deadlocks, [you need to understand the Synchronisation Context](https://msdn.microsoft.com/en-us/magazine/gg598924.aspx) 
and how it differs in the different kinds of application. 
If your code has a synchronisation context and it runs only one thread at a time, then it can deadlock. 
This is true in Windows desktop GUI applications (Windows forms and WPF), and in ASP; 
but is false in a console app, a windows service or threadpool thread, and [false in ASP.NET Core](http://blog.stephencleary.com/2017/03/aspnetcore-synchronization-context.html). 

| Platform                                         | Has Sync Context | One at a time |
|--------------------------------------------------|------------------|---------------|
| WinForms                                         | Yes              | Yes           |
| WPF                                              | Yes              | Yes           |
| ASP.Net                                          | Yes              | Yes           |
| Default (Thread pool, Console apps, NUnit tests) | Yes              | No            |
| ASP.Net Core                                     | No               | No            |


### Best to stay async

The best option is *don't resynchronise*. Don't throw away the benefits of async. This may be a fair amount of work, adding `await` and `async Task` on many methods, but it should always be the first choice.

In general async code contains one or more `await` statements, but also lots of synchronous statements that are not awaited. Doing synchronous things in async code is generally safe. Doing asynchronous things in synchronous code and not awaiting it is generally dangerous, and should be avoided. Code that tries to embed async code within synchronous code often has synchronisation-related problems. Code that is never async or code that is always async tends to not have these.

## Know when you do need to re-sync

There is a short list of times when re-syncing is not avoidable.

- You can't use `async` in these language constructs: constructors, `Dispose` methods and inside `lock` statements. You should re-design around these limitations, i.e. move the async code elsewhere rather than doing resyncronisation.

- In 3rd party frameworks and libraries, for example ASP.NET filters and child actions must be synchronous. However, [in ASP.NET Core, the entire pipeline is fully asynchronous](http://blog.stephencleary.com/2017/03/aspnetcore-synchronization-context.html). There are no synchronous child actions in ASP.NET Core, so it is best to find another construct to use instead.

- The `Main` entry point of a console application must be synchronous. 

- [A windows service will also have a synchronous entry point](http://stackoverflow.com/questions/39656932/how-to-handle-async-start-errors-in-topshelf).

## Know how to re-synchronise

There are a few ways to re-synchronise:

* Just Wait
* `Task.Run`
* Denial of context

### Just Wait.

This is group of properties and methods calls,  `.Result`, `.GetAwaiter().GetResult()` and `.Wait()`. 

This is very risky and problematic. If you are running somewhere with a synchronisation context (Windows forms, WFP & ASP) or inside a library that can be used in those kinds of apps, you should absolutely avoid this at all costs. 
This is where deadlocks can occur as you will prevent any continuations inside the `Task` returning function from being able to be continued on the current (and now blocked) synchronisation context.

### Launch a task with `Task.Run`.

How does this avoid deadlocks? `Task.Run` executes on the threadpool, which can change the `SynchronizationContext`, at the heavy cost of a second thread.

### Denial of context

Set the current `SynchronizationContext` to null, so the code that you call is denied access to it.

```csharp
var context = SynchronizationContext.Current;
SynchronizationContext.SetSynchronizationContext(null);

try
{
	// do something async
}
finally
{
	SynchronizationContext.SetSynchronizationContext(context);
}
```

This has the advantage over `Task.Run` of not using an additional thread for the initial synchronous invocations, and applying this technique at multiple levels comes at no extra cost, whereas each `Task.Run` with cost yet another thread.

### Use a library

e.g. using [vs-threading](https://github.com/Microsoft/vs-threading/)

```csharp
var context = new JoinableTaskContext();
var jtf = new JoinableTaskFactory(context);
var result = jtf.Run(() => DoSomethingAsync());
```

### Console application example

For a console entry point, you can *just wait*, [as is discussed here](http://stackoverflow.com/questions/9208921/cant-specify-the-async-modifier-on-the-main-method-of-a-console-app):
 
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
It is not a concern that we are blocking on async here as we are in a console app or windows services where there will be no deadlocks, and an additional thread being used is acceptable. There is a [language proposal](https://github.com/dotnet/csharplang/blob/master/proposals/async-main.md) to take away this boilerplate code in the future.

`.GetAwaiter().GetResult()` is a little nicer than `.Result` in that it behaves the same, but you will get the first exceptions thrown, instead of an `AggregateException`.

## Links

* [About the ways of re-syncing](http://stackoverflow.com/questions/42223162/task-run-vs-null-synchronizationcontext/) and [here](http://stackoverflow.com/questions/25095243/set-synchronizationcontext-to-null-instead-of-using-configureawaitfalse/).

* [Understanding what `SynchronizationContext` does](http://stackoverflow.com/questions/18097471/what-does-synchronizationcontext-do).

* [Async code in ASP.NET Core and the lack of synchronisation context](http://blog.stephencleary.com/2017/03/aspnetcore-synchronization-context.html)

* [Async in C# - The Good, the Bad and the Ugly - slides from a talk by Stuart Lang](https://speakerdeck.com/slang25/async-in-c-number-the-good-the-bad-and-the-ugly)

* [The Microsoft.VisualStudio.Threading library](https://github.com/Microsoft/vs-threading/)