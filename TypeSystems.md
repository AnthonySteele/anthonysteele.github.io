## Type systems

> "I think Java and C# have done a reasonable job at hovering near the balance point. ... So, a little type safety, like a little salt, is a good thing. Too much, on the other hand, can have unfortunate consequences." http://blog.cleancoder.com/uncle-bob/2017/01/13/TypesAndTests.html

The C# type system is good but not perfect. I'd like to dsicuss some of the ways that I feel it is lacking.

The design of the C# type system, compiler and class library is a product of the best thinking and tradeoffs of a point in time, but times move on.

C#'s pradigm is of a primarily object-oriented, garbage collected language with extensive runtime metadata and a portable bytecode.
Even without changing this basic paradigm things could be different. 

The system has been improved over time, but there are limitations to the technique of improving a system by adding to it but not removing. All programming languages accumulate cruft. IMHO the evolution of C# has been relatively well-managed, but this just makes the process slower.

## Cruft

### Lists

Too many interfaces. Arrays can be elimintated or moved to the p/invoke ghetto.

Func recapitulate delegates.

There are 3 kinds of tuples.

## New thinking

For a new language in 2017 as opposed to 1997, I would expect more emphasis to be placed on values being *Immutable by default* and *Not null by default*. Allowing changes or nulls can be something that is opted into explicitly. [F#])(http://fsharp.org/) works this way. So does [Swift](https://swift.org/) and so does [Rust](https://www.rust-lang.org).

C# did a better than reasonable job in the initial design, and a better than reasonable job in managing the evolution, but time has passed and thinking has moved on. Best practice isn't what it was.
