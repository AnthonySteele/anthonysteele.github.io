## Crufty Type systems

C#'s paradigm is of a primarily object-oriented, garbage collected language with extensive runtime metadata and a portable bytecode. I'm not going to venture out of that today, as even without changing this basic paradigm things could be tweaked. I'd like to discuss some of the ways that this is noticeable in regular use.

The design of the C# type system, compiler and class library is a product of the best thinking and tradeoffs of a point in time, but times move on. C# 1.0 came out in January 2002, and things got interesting with C# 2.0 and generics in November 2005.

It has been improved over time, but there are limitations to the technique of improving a system by adding to it but not removing. All programming languages accumulate [cruft](https://en.wikipedia.org/wiki/Cruft). IMHO the evolution of C# has been relatively well-managed, but this can only slow the decay.

One of the things that newcomers to .Net say these days is that there is extra effort to uncover which features they should use, and which they should not. They hear experienced team-members say "no, don't use that, it's obsolete, use this instead" all the time. This applies to language features, classes in the library, and whole subsystems, e.g. [ASP.Net Web Forms](https://www.asp.net/web-forms).

In this sense, cruft is [accidental complexity](https://en.wikipedia.org/wiki/No_Silver_Bullet) is avoidable [cognitive load](https://en.wikipedia.org/wiki/Cognitive_load) and it hinders us.

## Cruft

### Lists

Consider an everyday `List<Order>`. This inherits from  `IList<Order>, ICollection<Order>, IReadOnlyList<Order>, IReadOnlyCollection<Order>, IEnumerable<Order>`, and the non-generic versions: `IList, ICollection, IEnumerable`.

Another option is an array of `Order[]`, which is similar but not identical to a list. It inherits from `System.Array` and from the generic and non-generic `IList, ICollection, IEnumerable`. And has [odd covariance rules](http://stackoverflow.com/q/4317459/5599).

If there were no legacy concerns, we would eliminate unnecessary duplication: the non-generic versions of these types and interfaces can go. In the rare case that you want a list of objects, you can still type `List<object>`. 

I would also get rid of arrays, or if they are still needed for interop with system code, move them to a P/Invoke ghetto and not allow them to be used for anything else.

## Funcs and delegates

Delegates were the way to attach handlers to button click events since C# 1.0, before generics. And now there are lambda functions as well.

Try the code `var x = y => y + 1;`. The error is "*Cannot assign lambda expression to an implicitly-typed variable*". The compiler doesn't know if you want the type of `x` to be a delegate or a `Func<int, int>`. 

Because these two types are the same, only not. If one was designing the "delegates" system today, it would be based upon `Func`, not separate to it. 

## Tuples and tuples

There are 3 kinds of tuples or similar types. 

 * There is `var value = new Tuple<int, string>(1, "hello");`.
 * There is an anonymous object: `var value = new { Count = 1, Message = "hello" };`. 
 * And there are new C#7 value tuples, e.g. `public (double lat, double lng) GetLatLong()`. 

They are all different, all have uses, all filled a need at the time they were designed. But three kinds of tuple is *at least* one too many.

## conditional keywords

One of the main uses of the `async` keyword is to signal that in the code that follows, `await` cannot be a variable name, it must be a keyword. If `await` cannot be a variable name, then the compiler could largely infer that `async` should be there from the presence of an `await` and a returned `Task`. But instead much method annotation is needed.

Similarly, C# 7 has new pattern matching, but this is constrained by being added to the `switch` statement rather than adding a new keyword. This has consequences. Some features were not included - since it's a statement and not an expression, a switch will not occur to the right of a `var result =`. Nevertheless, the two variants of the `switch` syntax are going to add to the learning curve of next year's crop of junior c# coders. 

## New thinking

For a new language in 2017 as opposed to 2002, I would expect more emphasis to be placed on values being **Immutable by default** and **Not null by default**. Allowing changes or nulls can be something that is only allowed when opted into explicitly. [F#](http://fsharp.org/) works this way. So does [Swift](https://swift.org/) and so does [Rust](https://www.rust-lang.org).

I would also keep an eye on Rust for new thinking about eliminating data races in parallel code by entirely avoiding shared mutable state.

C# has gained some small-scale functional features, but it is not a "functional-first" language. And that's fine. But along those lines, there is a convenience in F# and in java that is lacking in C#: the type `Func<TIn, TOut>` could be trivially convertible to or from a matching interface such as `interface IDoSomething { TOut TheMethod(TIn input); }`.

Sum types with pattern matching like in F#, Swift or Rust would be good too.

In C# generics, many people have run up against the constraint that they can't accept "any type that knows how to use the `+` operator". This is called a [Type Class](https://en.wikipedia.org/wiki/Type_class).

C# did a better than reasonable job in the initial design, and a better than reasonable job in managing the evolution, but time has passed and thinking has moved on. Best practice isn't what it was.

## Objections

>  Nullable types: The question is: Whose job is it to manage the nulls. The language? Or the programmer?
- Robert C Martin, [The Dark Path, January 2017](http://blog.cleancoder.com/uncle-bob/2017/01/11/TheDarkPath.html)

Have a look at the [Proposal for C# Non-Nullable Reference Types](https://gist.github.com/olmobrutall/31d2abafe0b21b017d56). This shows that the c# team are pretty careful at managing addition to the language, and it happens only after much debate, thought to backward compatibility and weighing of pros and cons. But there's only so much that you can do with a mature language that has "zillions of lines  ... assuming reference types are nullable by nature".

Javascript has the same issue - how do you remove `var` from the language now that there's `let` and `const` , as there's zillions of lines of Javascript that use `var`? Should you? Can you prevent `var` in new code only?


> "I think Java and C# have done a reasonable job at hovering near the balance point. ... So, a little type safety, like a little salt, is a good thing. Too much, on the other hand, can have unfortunate consequences."
- Robert C Martin, [Types and Tests, January 2017](http://blog.cleancoder.com/uncle-bob/2017/01/13/TypesAndTests.html)

I can't really agree with Uncle Bob this time. As far as type safety goes, I'd like to lean as far in the direction of type safety as the expressiveness of the type system allows. I'd like to take on board proven best practices from research and from functional languages.
To add value, I'd prefer more type expressiveness over less type safety. Yes, types and tests are different tools towards reliability, and I'd like both. 

The point that "adding to the type system means added language complexity" is of course valid; and I've been talking about complexity at length above - all other thing being equal, less complexity is better. But we deal with large and complex class libraries already, that's the price of entry. Shifting some of that complexity onto the compiler is not necessarily bad: it might be bad if it's done badly, or it might be great if done well. 


