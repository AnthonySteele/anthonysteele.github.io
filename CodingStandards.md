# On Coding Standards

So you're doing a coding standard. Here are my thoughts on how to go about it.
It's light on actual coding style rules, mostly concerned with how to derive and present these rules.

## Alignment

A standard is about aligning code into common patterns, to make it more readable to a new viewer, and easier for anyone else to pick up.
And further, it should spread good practices, resulting in code with desirable characteristics, e.g. robust and performant.
You can warn of pitfalls and capture institutional knowledge of problems overcome.

Just as a _team_ style is preferred over than a _personal_ style when working with others, an _organization_ style is better still,
and an _industry standard_ best of all. Therefore, align with the industry standards by default.

## Use Prior Art

There is plenty of prior art now. Use it, instead of re-inventing it.
IMHO, the style guide should link to the standards instead of paraphrasing them. e.g. start with
"We follow these industry standards", and then after the links, "with the following exceptions, clarifications and extensions..."

It is usually a waste of time to write your own standards from scratch, in many ways:
the effort to create it, and the effort for new hires to learn how it differs from their prior industry experience, and the effort to get them to follow it.

In the .NET world I think of these documents:

* [C# Coding Conventions](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/inside-a-program/coding-conventions)
* [.NET Framework Design Guidelines](https://docs.microsoft.com/en-us/dotnet/standard/design-guidelines/)
* [C# Programming Guide](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/)
* [Philips Healthcare C# Coding standard](https://tics.tiobe.com/viewerCS/index.php?CSTD=General)
* [IDesign Standards](https://idesign.net/)

And books such as the _Effective C#_ series by Bill Wagner.

## Familiarity

Don't underestimate the role that [mere familiarity](https://en.wikipedia.org/wiki/Mere-exposure_effect) plays.
For instance, you can still find people trying to make the case that [Imperial measurements](https://en.wikipedia.org/wiki/Imperial_units) in Miles, Feet and Inches is a "better system".
Invariably, they were raised on it since early childhood and the case rests on the premise that it "just feels right".
Well, it does, _to them_. It's subjective, although it might not feel like it.
When two different styles are equivalent (or sometimes even when they are not), the one that we are familiar with will be perceived as intuitively "better" and more usable.

Just understand that sometimes a choice has to be made, to aid regularity, even when neither option is superior in any objectively quantified way.
Even though there may be no objective truth to the choice, it is still worth getting a common familiar style.

Sometimes you just have to use the unfamiliar style that you don't like for a while, and gain familiarity, before your opinion of it changes.

## Minimize Accidental Complexity

Sometimes you can measure: **Minimize [accidental complexity](https://en.wikipedia.org/wiki/No_Silver_Bullet)**.
When there is free choice, pick the style where the rules are more simple and consistent. e.g. The simple rule that "`if` statements are always followed by a block with braces" in this style:

```csharp
if (cond)
{
    statements;
}
```

The rule has the advantage that there is no "part two" to it. There are no other layouts, special cases, decisions or thought required to apply it.

This is one of the reasons why, despite initial resistance, use of `var` has become pervasive in C# code: it is the simplest possible rule, and you don't have to think about it.

## Automate

Look for the automated tools: Let the computer do the rote work. Automation is far faster and more consistent than eyeball inspection and manual fixing.

If you are fortunate, your modern language comes with a standard formatter and a standard style included from day one, such as [this for Rust](https://github.com/rust-lang/rustfmt) or [this for Golang](https://blog.golang.org/gofmt).
If not (and this is the case in the .NET world) then you will likely find all the pieces that you want if you look around.
There may be emerging standards and well-known tools.

In .NET [there is `dotnet format`](https://github.com/dotnet/format) for spacing, editor support for [the `.editorconfig` file](https://editorconfig.org/).
And the [Roslyn Analysers](https://github.com/dotnet/roslyn-analyzers) for linting. Even fixing the compiler warnings and turning on "warnings as errors" counts.

## Levels of Rule Strength

Use the levels: **Must**, **Should**, **Should Not**, and **Must not** as per [RFC 2119: Key words to Indicate Requirement Levels](https://tools.ietf.org/html/rfc2119) to describe rules.
Equivalently, these are called **Do**, **Consider**, **Avoid**, and **Do Not**.

Some rules are "Always" and "Never" cases, but the majority will be "Should" / "Should not" strength:
it says "Prefer to do it this way, unless you have a good reason to do otherwise, as this is a sensible default."
It instructs novices who have no preference yet, without blocking experts from making exceptions where there is a good reason to.

## Rules Should Have Reasons

A rule should have a reason.
If you do not consider the reasons for rules, you might end up carrying cargo-cult rules such as [Single Return](./TheSingleReturnLaw), [Yoda conditions](https://en.wikipedia.org/wiki/Yoda_conditions)
or "constants in `ALL_CAPS`", in languages where they are no longer needed.

Languages have different coding conventions, and some of that is by chance and culture e.g. [Use of K&R brace style](https://en.wikipedia.org/wiki/Indentation_style#K&R_style).
But some of it is down to how the language works, and that can vary: the examples given above are useful in `C`, but do not make sense in `C#` or many recent languages.
So the experience gained in other languages and ecosystems is valuable, but is not infallible.

Consider writing the rule with a reason, e.g. as "You **should** do _some action_ so that _desired outcome_." and see if it still makes sense.

## Group and Progression

A coding style is not just a bag of unrelated items.
Consider structuring it from low level to high level, e.g. from spacing to syntax to compiler warnings to [S.R.P.](https://en.wikipedia.org/wiki/Single-responsibility_principle) to library usage,
to building and testing practices, to architecture and good practices for more specialized topics such as interacting with Azure, AWS or other platforms.

The place to innovate and add value is higher up, where existing style guides don't cover your specific cases.

You might find that you end up with several distinct documents, covering different but related topics, with different linked literature and different reasons to change.

Beware of vague statements. e.g. Writing "good design is better" is something that we can all agree with, but fails to educate anyone on what constitutes good design or in which way it is better and under what circumstances.
Do capture specifics of lessons learned along the way.
