# On the subtleties of Exception handling in C#

or, Why `throw new Exception(inner);` is an act of vandalism,

Links:

* [Vexing Exceptions - Eric Lippert](https://learn.microsoft.com/en-us/archive/blogs/ericlippert/vexing-exceptions)
* [Exceptions for flow control](https://enterprisecraftsmanship.com/posts/exceptions-for-flow-control/)
* [Serilog.Exceptions](https://github.com/RehanSaeed/Serilog.Exceptions)

People have different ideas about exceptions handling in C# code.
And they often have simple patterns that work well enough, but could be improved upon.

One example is the suggestion of wrapping all exceptions in `throw new Exception()` at a certain point in a library. In my opinion, this is an act of vandalism.

Let me explain. It is said that "you have to know the rules before you decide when to break them". The rules in this case are:

[CA2201: do not throw the base exception](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca2201) and [CA1031: do not catch all exceptions](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/quality-rules/ca1031). And you should know these rules and why they exist before deciding if it’s appropriate to break them today.

Catching all exceptions is not ideal: when you wrap e.g. a database operation in `catch(Exception ex)` you’re going to catch several kinds of exception - [See Eric Lippert's taxonomy](https://learn.microsoft.com/en-us/archive/blogs/ericlippert/vexing-exceptions):

* Exception types that you shouldn't catch at all, as you can’t handle them, e.g. `OutOfMemoryException`. These are often **fatal**.
* common Exception types that you did not expect, and are your own **boneheaded** fault.  e.g. a `NullReferenceException`. Humility is a virtue for coders, as we all make boneheaded mistakes from time to time. These should instead be fixed with code changes to prevent the exception being thrown at all. Iteratively fixing your stuff is another virtue.
* The type that you wanted to catch. These are often **exogenous** - arising due to processes outside of your control, and where you can and should deal with them e.g. a  `SqlException`.

Catch blocks should happen for "a very limited range of exception types that you know you can safely handle" - this means using specific exception types. Central to this is the idea that the Exception types matter, that they are not all the same type, and that the type is used as a filter.  

Exception handling is often done poorly - don't think of as "wrap this method in a blanket catch all exceptions because it can fail" rather as "wrap this database operation (but not the request building code before it or the response parsing code after it)" in a catch `DbException` block because failures of that kind can happen in that block, and are `exogenous` as they happen on the database server, beyond my control. This is IMHO more readable and expresses the understanding of known failure modes of the code in the try block.

Rather than "wrap the method in `catch (Exception)` as a blanket, wrap the operation in `catch (DbException)`. This reduces the scope to only the exception types that indicate that this operation failed. Exclude the code before and after that e.g. initializes objects and parses results, which cannot e.g. throw a DbException - this ensures that you're not sweeping up other "unexpected failures" such as e.g. a `NullReferenceException` in your code
Catch blocks should wrap a specific block that has known failure modes, rather than as a blanket around the whole thing.

I sometimes write code that catches all exceptions, or leave existing code alone, but sometimes that is because there is no choice, due to the first rule not being followed well. this pattern won't work if the called code is doing `throw new Exception()`, which it really should not do.

`System.Exception` is the base class, and maybe it should have been an abstract base type, but we’re stuck with it as is
When throwing, erasing the exception type information with a  `throw new Exception(inner)` wrapper is an act of vandalism against the type system. It takes away the choice to filter exceptions from the caller.

In Library code, there is benefit to:

* Not catching all exceptions, so that typed exceptions come through as-is from underlying layers, so that e.g. If CosmosDb permission are not there, we get a `CosmosDbException`
* Ensuring that all deliberately thrown exceptions are of a type (or have a base type) defined by that library, so that they can be caught separately to all other exceptions, so that e.g. `FooClientLibrary` defines a `FooClientException`.

And related, any logging system that receives an exception should behave like [Serilog.Exceptions](https://github.com/RehanSaeed/Serilog.Exceptions)  and record all public properties of the actual exception type. An exception is structured data, and due to inner exceptions it is also recursively structured data. Structured logging techniques apply. The logging or telemetry system should handle this for any exception type.

Is this inefficient and will need reflection to build such a map of public properties?  IS this a performance issue? Maybe, but consider that:

* Applications tend to throw a small set of exceptions regularly, and a map can be built when the exception type is first encountered, then re-used.
* Efficiency in terms of time taken and data stored, is not the primary goal of exception handling. It should be infrequent (indeed, exceptional!)  When an exception does occur, performance is not the primary concern at all, complete and accurate recording is primary, so that the error can be mitigated to occur less frequently.

[This suggestion here of wrapping all Exceptions](https://timwise.co.uk/2014/05/10/throw-vs-throw-ex-vs-wrap-and-throw-in/) is very bad advice. It breaks the rules, and not in the expert way, but in the "didn’t know any better" way. We should know better.
