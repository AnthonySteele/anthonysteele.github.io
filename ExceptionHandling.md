# On the subtleties of Exception handling in `C#`

Or, why `throw new Exception(ex);` is an act of vandalism.

## Links

* [Vexing Exceptions - Eric Lippert](https://learn.microsoft.com/en-us/archive/blogs/ericlippert/vexing-exceptions)
* [Exceptions for flow control](https://enterprisecraftsmanship.com/posts/exceptions-for-flow-control/)
* [Serilog.Exceptions](https://github.com/RehanSaeed/Serilog.Exceptions)

## Good practices

People have different ideas about exceptions handling in C# code.
And they often have simple patterns that work well enough, but could be improved upon.

One example is the suggestion of wrapping all exceptions in `throw new Exception()` at a certain point in a library. In my opinion, this is an act of vandalism. Let me explain. It is said that "you have to know the rules before you decide when to break them" (sometime attributed to Pablo Picasso).

The rules in this case are: [CA2201: do not throw the base exception](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca2201) and [CA1031: do not catch all exceptions](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1031). This is discussed further at the links.  And you should know these rules and why they exist before deciding if it’s appropriate to break them today.

Catching all exceptions is not ideal: when you wrap e.g. a database operation in `catch(Exception ex)` you’re going to catch several kinds of exception - [See Eric Lippert's taxonomy](https://learn.microsoft.com/en-us/archive/blogs/ericlippert/vexing-exceptions):

* Exception types that you shouldn't catch at all, as you can’t handle them, e.g. `OutOfMemoryException`. These are often **fatal** exceptions.
* Common exception types that you did not anticipate, but are your own **boneheaded** fault.  e.g. a `NullReferenceException`. Humility is a virtue for coders, as we all make boneheaded mistakes from time to time. These should instead be fixed not by catching the error, but with code changes to prevent the code from throwing at all. Iteratively fixing your stuff is another virtue.
* The type that you actually needed to catch. These are often **exogenous** - arising due to processes outside of your control, and where you just have to deal with them e.g. code may need to catch a `SqlException` or it may need to catch a `SocketException`. But likely not both.

Exception handling is often done poorly in this way:
Catch blocks should happen "[for a very limited range of exception types that you know you can safely handle](https://enterprisecraftsmanship.com/posts/exceptions-for-flow-control/)" - this means using specific exception types. Central to this is the idea that the **Exception types matter**, that they are not all the same type, and that the type is there to be used as a filter.  

Throwing a specific exception type does not prevent `catch (Exception ex)`, if the caller chooses to do that; but `throw new Exception()` prevents calling code from doing anything else.

Exception handling is also often done poorly in another way:  Poorly targeted as to the code inside the `try...catch` block.

Catch blocks should often wrap a specific block that has known failure modes, rather than as a blanket around lots of diverse code. Rather than think of as "wrap this whole method in a catch all exceptions because it can fail" rather as "wrap this database operation in a catch `DbException` block because failures of that kind can happen in that block, and are `exogenous` as they happen at or en route to the database server, beyond my control.

Exclude the request building code before it or the response parsing code after it from this catch block as this code _cannot_ throw `DbException`.
This ensures that you're not sweeping up other "unexpected failures" or **boneheaded** mistakes.

This is IMHO more readable, more specific and expresses the understanding of known failure modes of the code in the try block.

[This suggestion here of wrapping all Exceptions](https://timwise.co.uk/2014/05/10/throw-vs-throw-ex-vs-wrap-and-throw-in/) is very bad advice. It breaks the rules, and not in the expert way, but in the "novice didn't know any better" way. It makes the caller's life harder. We should know better now.

## Considerations

I sometimes write code that catches all exceptions, or leave existing code alone. Sometimes this is because very generic common code cannot rightly say what exception types matter, but sometimes that is because there is no choice, due to the first rule not being followed well.

`System.Exception` is the base class, and maybe it should have been an abstract base type, but we're stuck with it as is.
When throwing, erasing the exception type information with a  `throw new Exception(inner)` wrapper is an act of vandalism against the type system. It takes away the choice to filter exceptions from the caller.

In Library code, there is benefit to:

* Not catching all exceptions, so that typed exceptions come through as-is from underlying layers, so that e.g. If CosmosDb permission are not there, we get a `CosmosDbException`
* Ensuring that all deliberately thrown exceptions are of a type (or have a base type) defined by that library, so that they can be caught separately to all other exceptions, so that e.g. `FooClientLibrary` defines a `FooClientException`.

## Logging

I am going to be recommending the approach of [Serilog.Exceptions](https://github.com/RehanSaeed/Serilog.Exceptions) for a long time. Any logging system that receives an exception should behave like that.

* An exception is not a string, nor a couple of strings for "type, message, stacktrace". An exception is a _structured_ object with named fields with typed values.  
* An exception is a _recursively_ structured object due to `InnerException` or `InnerExceptions`.  And if your actual issue is wrapped in a `System.Exception` then accurately recording inner data becomes even more important.
* An exception can have _different properties_ due to differing exception types in the framework.  (e.g. if you have a [`ValidationException`](https://learn.microsoft.com/en-us/dotnet/api/system.componentmodel.dataannotations.validationexception), then the [`ValidationResult`](https://learn.microsoft.com/en-us/dotnet/api/system.componentmodel.dataannotations.validationexception.validationresult) and properties of that, are important).  
* An exception can have _custom properties_ due to custom exception types declared in the application not the framework.

 Structured logging techniques apply: Different data items should be stored in different fields of the exception, and written to different fields of the logs.

If e.g. an `UserException`  has a property `public Guid UserId { get; }`. Then, do I want `exception.UserId` to be logged a separate field every time that logging or telemetry records the exception? _Yes, of course, that's exactly what it's there for!_

The logging or telemetry system should handle this for any exception type.

Is this inefficient and will need reflection to build such a map of public properties?  Is this a performance issue? Maybe, but consider that:

* Efficiency in terms of time taken and data stored, is not the primary goal of exception handling. Reliability come before Efficiency.  It should be infrequent (indeed, exceptional!) to log exceptions. But when an exception does occur, performance is not the primary concern at all, complete and accurate recording is primary, so that the error can be mitigated to occur less frequently in future.
* Even un-optimized, these times are actually quite small compared to e.g. Database server queries or HTTP round-trips.
* Many frameworks (e.g. Dependency Injection tools, JSON serialization libraries) have succeeded in doing both performance and flexibility:
  * They can handle many types, even user-defined types that have not been seen before.
  * Performance is more than good enough over the lifetime of the application, due to e.g. using code generation to build efficient code for that type, and not doing reflection from scratch every time.
* Applications tend to throw a small set of exceptions regularly, so the same types will be encountered again, therefor caching techniques will work well.
