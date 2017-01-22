## Type systems

> "I think Java and C# have done a reasonable job at hovering near the balance point. ... So, a little type safety, like a little salt, is a good thing. Too much, on the other hand, can have unfortunate consequences." http://blog.cleancoder.com/uncle-bob/2017/01/13/TypesAndTests.html

The C# type system is good but not perfect. I'd like to dsicuss some of the ways that I feel it is lacking.

The design of the C# type system, compiler and class library is a product of the best thinking and tradeoffs of a point in time, but times move on.

C#'s paradigm is of a primarily object-oriented, garbage collected language with extensive runtime metadata and a portable bytecode.
Even without changing this basic paradigm things could be different. 

The system has been improved over time, but there are limitations to the technique of improving a system by adding to it but not removing. All programming languages accumulate cruft. IMHO the evolution of C# has been relatively well-managed, but this just makes the process slower.

## Cruft

### Lists

Consider an everyday `List<Order>`. This inherits from  `IList<Order>, ICollection<Order>, IEnumerable<Order>, IReadOnlyList<Order>, IReadOnlyCollection<Order>`, and the non-generic versions: `IList, ICollection, IEnumerable`.

You can use this or an array of `Order`, which inherits from `System.Array` and from the generic and non-generic `IList, ICollection, IEnumerable`.

If there were no legacy concerns, we would eliminate the non-generic versions of these interfaces: in the rare case that you want a list of objects, you can still type `List<object>`, the remove a few of the other interfaces, and get rid of arrays, or if they are still needed for interop with system code, move them to a P/Invoke ghetto.

Func recapitulate delegates.
Try doing `var x = y => y + 1;`. The error is "Cannot assign lambda expression to an implicitly-typed variable". The compiler doesn't know if you want the type of `x` to be a delegate or a `Func<int, int>`. Because they're the same, only not.

There are 3 kinds of tuples. There is `var value = new Tuple<int, string>(1, "hello");1, there is `var value = new { Count = 1, Message = "hello" };` and there are [new C#7 value tuples](https://www.kenneth-truyers.net/2016/01/20/new-features-in-c-sharp-7/). They are all different, all have uses, all filled a need at the time. But three kinds of tuple is *at least* one too many.

## New thinking

For a new language in 2017 as opposed to 1997, I would expect more emphasis to be placed on values being *Immutable by default* and *Not null by default*. Allowing changes or nulls can be something that is opted into explicitly. [F#])(http://fsharp.org/) works this way. So does [Swift](https://swift.org/) and so does [Rust](https://www.rust-lang.org).

C# has gained some small-scale functional features, but it is not a "functional-first" language. Along those lines, there is a convenience in F' and in java that is lacking in C#, and that is that the type `Func<TIn, TOut>` could be trivially convertible to or from a matching interface `interface IDoSomething { TOut TheMethod(TIn input); }`.

C# did a better than reasonable job in the initial design, and a better than reasonable job in managing the evolution, but time has passed and thinking has moved on. Best practice isn't what it was.
