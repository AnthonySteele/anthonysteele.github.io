# Avoiding advanced mistakes in async await

After writing "Avoiding simple mistakes in async await" and revising it many times, 
it becomes inescapable that some of the uses and abuses of async code are not simple.

If you are confused, read that first. This part will cover getting out of async, i.e. re-synchronising. The first option is *don't resynchonise*. Avoid doing it, stay async.

## Know when you do need to re-sync, and how to do it.

In general async code contains one or more `await` statements, but also lots of synchronous statements that are not awaited. Doing synchronous things in async code is generally safe. Doing asynchronous things in synchronous code is generally dangerous, and should be avoided. 

If you get a deadlock in code that does follow the correct async patterns, it probably means that you have other problems lurking. Code that is never async or code that is always async tends to not have these problems. Code that tries to be both in different places often does.

There is a short list of times when re-syncing is not avoidable.


- You can't use `async` in these language constructs: constructors, `Dispose` methods and inside `lock` statements. You should re-design around these limitations, i.e. move the async code elsewhere rather than doing resyncronisation.

- ASP.Net filters and child actions must be synchronous. However, [in ASP.NET Core, the entire pipeline is fully asynchronous](http://blog.stephencleary.com/2017/03/aspnetcore-synchronization-context.html). There are no synchronous child actions in ASP.NET Core, so it is best to find another construct to use instead.

- The `Main` entry point of a console application must be synchronous. 

### How to re-sync

For a console entry point, you have to do [something like this](http://stackoverflow.com/questions/9208921/cant-specify-the-async-modifier-on-the-main-method-of-a-console-app):
 
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


Elsewhere, you need `Task.Run` or setting the current `SynchronizationContext`.

How does this avoid deadlocks? `Task.Run` executes on the threadpool, which can change the `SynchronizationContext`, at the heavy cost of a second thread.

[Discussing the two ways](http://stackoverflow.com/questions/42223162/task-run-vs-null-synchronizationcontext/) and [here](http://stackoverflow.com/questions/25095243/set-synchronizationcontext-to-null-instead-of-using-configureawaitfalse/).

[understanding what `SynchronizationContext` does](http://stackoverflow.com/questions/18097471/what-does-synchronizationcontext-do).



* [Async code in ASP.NET Core and the lack of synchronisation context](http://blog.stephencleary.com/2017/03/aspnetcore-synchronization-context.html)
