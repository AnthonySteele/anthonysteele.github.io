# AutoMapper considered harmful

TL:DR: You don't need it. Just write the mapping code.

## Why mapping

Consider a simple .NET data API. How many different kinds of [Data Transfer Object](https://martinfowler.com/eaaCatalog/dataTransferObject.html) do you need?  In a simple example, there has to be a class that maps to the data store - the "data model", and there has to be a class that is serialized as the HTTP response - the "view model".

I view the data model complexity as part of how complex the code needs to be. Right after the `File|New Project`, the data model and the view model might be the same type, call it `CustomerModel`. But as the project grows in size and complexity, it is very likely that you will want to separate these two concerns. e.g. When some fields on the data model are not exposed to the web, or some fields on the view model are computed or come from another source, and not stored in the data model. So at that point you split `CustomerDataModel` from `CustomerViewModel`, and now you have to map between them.

## Why AutoMapper

[AutoMapper](https://jimmybogard.com/tag/automapper/) automates this task of mapping. It is a popular tool in the .NET space.  I have seen pull request reviews where someone with the audacity to manually map four or five fields from one object to another is quickly told "_you must use AutoMapper for this!_". This common idea is IMHO not just wrong (using AutoMapper is a choice), it is harmful.

There are pros and cons to every design choice. People so often see the upsides of AutoMapper and not the downsides, but of course they do exist.

 With AutoMapper, the selling point is that you don't have to write boilerplate "boring mapping code".

 But now you have another NuGet package to keep up to date, the `IMapper` to configure and inject, so there are more moving parts. But these are side issues to the main objections.

### Mapping code is not worth abstracting

If your AutoMapper mappings are trivial, why not map with trivial code? It's dumb code, but I like dumb code for dumb tasks. It is easy to read, easy to test, hard to get wrong, and has no dependencies that need updates. The example is given in the `Example` section below.

If your AutoMapper mappings are not trivial - then that configuration in the AutoMapper DSL _is the mapping code_ and now it's just hidden and more complex due to extra layers of indirection - would you rather program in AutoMapper DSL or in plain old C#?

Let me repeat that:

* If the mapping is trivial, then you're better without AutoMapper because it's simple to do without it.
* If the mapping is _not_ trivial then you're better off without AutoMapper because of that complexity.

Mappings in dumb code is [Boring Technology](http://boringtechnology.club). AutoMapper is [Clever code](https://guifroes.com/clever-code-is-bad/). Clever code  doesn't impress me. Boring code with zero dependencies does.

### AutoMapper will attract hidden business logic

AutoMapper is supposedly transparent. It's pretty clear what it  _should_ be doing in e.g. `var viewModel = mapper.Map<CustomerViewModel>(dataModel);`.

Emphasis on _should_. It's completely hiding the details of what it _is actually_ doing in the mapping. I have run into confusion and complexity on occasion here. It would be nice if the mapping was straightforward, but often it isn't. And the mapping is "non-local" - i.e. not easy to step into or find from the call site.

AutoMapper configuration tends to attract business logic. I have seen significant facts about how the business logic works, embedded in AutoMapper field mappings, which is not the place for them.

When I have to maintain a "mature" codebase that has been worked on for several years, and I see that it uses AutoMapper, my heart sinks a bit because I know that there are almost always going to be difficulties hidden therein. I know that they _should_ not be difficulties of business logic in the mapping. And yet there usually are. It attracts trouble rather than being the "pit of success".

## Testing

Did you know that there is a method `AssertConfigurationIsValid` for [testing AutoMapper configurations](https://docs.automapper.org/en/stable/Configuration-validation.html)? It "checks to make sure that every single Destination type member has a corresponding type member on the source" which is one kind of error. I did not know about it either: I never see it used. Yet it's best practice that you should _always_ test AutoMapper configurations wity this. But here's nothing forcing use of it.

How should you test a class that uses an `IMapper`? The obvious idea is to mock the `IMapper` and the response from it. But this means that your test mocks are more complex, the test doesn't reflect an actual run, and the mapping logic does not get tested.

## Libraries to solve known problems

AutoMapper solves a problem. But it's a trivial problem that doesn't take a library to solve. I tend to prefer libraries (e.g. a JSON Serializer, or SQL mapper) when:

* I can't do it myself trivially, which typically means that there is complexity or special knowledge required to write it. I know I couldn't get every aspect if a Json Serializer or SQL mapper correct the first time around, so why bother with all that effort when the problem has already been solved.

* It is at the edges of the app. I'm much happier saying "here is where we hand off the response that has been built up to the serializer" or "here is where we hand it to the DB driver" than I am saying "it disappears over here and pops up again there with a different type and some new fields".

* It is going to be given data, not business logic. You can control what is serialized to some extent with attributes and options, but overusing these these for complex serialization would be an obvious code smell. That logic isn't generally injected into the serializer. Instead, the `CustomerViewModel` class definition is used as that specification of how the customer data is represented when serialized as JSON, because this is much more legible.  
This seems to be a case of "moving the complexity around" to a better place for it, rather than eliminating it: We shape `CustomerDataModel` and `CustomerViewModel` DTOs to capture the specifics in the data types and data values _before_ it is handed to the SQL and JSON code respectively, to serialize in a straightforward way. The necessary mapping complexity must still exist, but there are better and more readable places for it than the configuration of any library.

## When to Use AutoMapper

The author, Jimmy Bogard [says](https://jimmybogard.com/automappers-design-philosophy/):

> I set out to build a tool that:
>
> * Enforced a convention for destination types
> * Removed all those null reference exceptions
> * Made it super easy to test
>
> AutoMapper works because it enforces a convention. It assumes that your destination types are a subset of the source type. It assumes that everything on your destination type is meant to be mapped. It assumes that the destination member names follow the exact name of the source type.
> With AutoMapper, we could enforce our view model design philosophy. This is the true power of conventions - laying down a set of enforceable design rules that help you streamline development along the way.

So there is a better case for AutoMapper when the mapping is is "wide but shallow" i.e. lots of fields to map, but the names map exactly without complex configuration. The value add seems to be in object flattening and null handling - but this might not be so important any more, since modern c# has several new tricks to deal with nulls.

It also is key if you can enforce the convention that view model member names match. I am in favour of names matching as much as possible anyway, simply because it is less unnecessary complexity to keep in mind. But when they do easily match, there is a better case for AutoMapper. Lets consider when that's easy and when it is not:

* If your view model is consumed by a razor page or other templating engine, for server-side rendering in the same application then you have complete freedom to name the view model field anything that you want, and you can easily enforce the convention.
* If your view model is served on an API, and consumed by an associated front-end such as a "Single-Page App" then it's is quite easy to change the view model, if the corresponding front-end change is made and shipped at the same time.
* If your view model is served on an internal API, and consumed by more than one in-house client, then it is harder to change view model layout, but you can probably get impacted parties to make the necessary changes, given time.
* If your view model is served on a public API, and consumed by many parties outside of the organization, then it is a public contract. As a rule you never make breaking changes to it without a very good reason and lots of time for client to migrate. Additive changes are more common, as they are not breaking. e.g. there is often a "new" and "old" version of an endpoint running in parallel for a long time. Expect to send reminders to help some clients with the migration.
* If your model is sent to a third-party API then you have no choice, they define the contract.

> If you find yourself hating a tool, it's important to ask - for what problems was this tool designed to solve? And if those problems are different than yours, perhaps that tool isn't a good fit.

I don't disagree with that, but I think it's a corollary the cases at which AutoMapper excels are a small subset of the real-world uses that it is put to. And that the required discipline to make the usage of it transparent by sorting out the underlying mismatches is seldom present, because there's nothing enforcing that discipline other than higher maintenance costs later on.

## Example

You should be able to code and test like this code below:

```csharp
public class CustomerDataModel
{
    public string Name { get; set; }
    public DateTimeOffset Start { get; set; }
}

public class CustomerViewModel
{
    public string Name { get; set; }
    public DateTimeOffset Start { get; set; }
}

public static class CustomerMapper
{
    public static CustomerViewModel Map(CustomerDataModel dataModel)
    {
        return new CustomerViewModel
        {
            Name = dataModel.Name,
            Start = dataModel.Start
        };
    }
}
```

`CustomerMapper.Map` is a "[Pure function](https://en.wikipedia.org/wiki/Pure_function)", i.e output depends only on input, and there is no "side effect", no other state either accessed or altered by it. So it is easily testable, if you feel the need.

You can make `CustomerMapper.Map` an extension method as `CustomerViewModel Map(this CustomerDataModel dataModel)` if you want. You can pull it out to a separate namespace to both the Data Model and View Model if you want. It can be coupled or decoupled as you need.

## Horses for courses

So you disagree, and like AutoMapper, and we can do different things, what's the problem? The issue comes when I have to maintain code damaged by AutoMapper. Or when lack of AutoMapper is assumed to be a defect and not a virtue.

Or when a Junior developer says

> I've seen a lot of logic stuffed into AutoMapper. It becomes this weird magic thing that is very hard to understand. Especially as a junior dev.

The next person to review a Pull Request with the suggestion that "_you must use AutoMapper for this!_" will be directed to this essay, in the hope that they might learn something. AutoMapper is very much optional.

## Links

* [AutoMapper's Design Philosophy - Jimmy Bogard](https://jimmybogard.com/automappers-design-philosophy/)
* Cezary Piątek: [The reasons behind why I don't use AutoMapper](https://cezarypiatek.github.io/post/why-i-dont-use-automapper/) and [how to generate the mapping code](https://cezarypiatek.github.io/post/generate-mapping-code-with-roslyn/)
* StackExchange question: [Am I wrong in thinking that needing something like AutoMapper is an indication of poor design?](https://softwareengineering.stackexchange.com/questions/167250/am-i-wrong-in-thinking-that-needing-something-like-automapper-is-an-indication-o)
