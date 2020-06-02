# AutoMapper considered harmful

## Why mapping

Consider a simple dotnet data API. How many different kinds of [Data Transfer Object](https://martinfowler.com/eaaCatalog/dataTransferObject.html) do you need? I view it as a function of how complex the code is. In a simple example, there has to be a class that maps to the data store - the "data model", and there has to be a class that is serialised as the http response - the "view model".

Right after the `File|New Project`, these might be the same type, call it `Foo`. But as the project grows in size and complexity, it is very likely that you will want to separate these concerns. e.g. When some fields on the data model are not exposed to the web, or some fields on the view model are computed and not stored.

So then you split `FooDataModel` from `FooViewModel`, and now you have to map between them.

## Why AutoMapper

[AutoMapper](https://jimmybogard.com/tag/automapper/) automates this task of mapping.

I am one of the people that doesn't "get" AutoMapper. In summary: What can I do with it that I could not do before? What hard problem does it solve, and how is your code better with it? Is it worth the inevitable costs?

If your AutoMapper mappings are trivial, why not map with trivial code?

If your AutoMapper mappings are not trivial - that _is_ mapping code, just hidden and more complex due to extra layers of indirection - would you rather program in AutoMapper DSL or in plain old dumb code? I say "dumb code" but I like dumb code for dumb tasks. It is easy to read, easy to test, hard to get wrong, and has no dependencies that need updates.

Yes there are pros to AutoMapper, you don't have to write "boring mapping code" but boring mapping code is dead simple to write and easy to test so it's not a huge win. But if you use AutoMapper instead you now have "magic" meaning it's not obvious what's being mapped, so you might want the tests anyway, and another NuGet package in your project to maintain, mappers to inject so more moving parts, etc.

People so often see the upsides of AutoMapper and not the downsides. They do exist. Maybe AutoMapper is "clever" but honestly that doesn't impress me like simple code with zero dependencies does.

###

> `CreateMap<FooDataModel, FooViewModel>();`  is pretty clear in what it _should_ be doing.

Emphasis added. And it's completely hiding the details of what it _is actually_ doing. Yes I have run into ... complexities on occasion here.

> I know I have multiple view models with 20+ properties

Ah, I typically don't. YMMV. 

> It is obnoxious to map those to and from domain models by hand

Why? You write the map, and it works, trivially. Done. I guess you're saying that you don't like it. Fair enough, but that's not an objective thing, just a matter of taste.

Fewer lines of code? OK, so you have 5 datamodels with 20 properties each, that's 100 lines of mapping code that you have saved. This is small potatoes in a large project.

> there is an obvious 'problem' it attempts to solve.

Yes, a _trivial_ problem.

> AutoMapper works because it enforces a convention. It assumes that your destination types are a subset of the source type. It assumes that everything on your destination type is meant to be mapped. It assumes that the destination member names follow the exact name of the source type. 

AutoMapper usage that I have seen would be _much_ nicer if it was all like that, or if you could reasonably assume that about opaque mappings that you come across in the wild. Instead, it tends to attract business mapping code.

>  If you find yourself hating a tool, it's important to ask - for what problems was this tool designed to solve? And if those problems are different than yours, perhaps that tool isn't a good fit.

Agreed 100%

Typically my beef is with the last guy to "maintain" the code, and only indirectly with the tools that they chose to abuse to that end.

PS: I still owe you a beer next time you're in town, whenever that is gonna be now :(

## Libraries to solve known problems

I tend to prefer libraries (e.g. JSON Serialiser, or SQL driver) when

a) I can't do it myself trivially, which typically means that there is complexity or special knowledge required to write it. I know I couldn't get every aspect if a Json parser or SQL mapper correct the first time around, so why bother.

b) It's not part of the business problem domain. Thing is, I have seen significant facts about how the business logic works, embedded in AutoMapper field mappings, which IMHO is not the place for them

c) At the edges of the app. I'm much happier saying "here is where we hand off the response that has been built up to the serializer" or "here is where we hand it to the DB driver" than I am saying "it disappears over here and pops up again there with a different type"

## what to do

It's optional.

If you use AutoMapper in your project or team, fine. Maybe you disagree on this, maybe you have special requirements that I haven't seen before that make it more worthwhile?

If I work on that project then I'll deal with the automapper.

If you review my project and start with "OMG WTF you must use AutoMapper!" then you're going to be told to go away, you don't have the insight that you think you do.

You should be able to code and test like this code below:

```csharp
public class FooDataModel
{
    public int Count { get; set; }
}

public class FooViewModel
{
    public int Count { get; set; }

}

public static class FooMapper
{
    public FooViewModel Map(FooDataModel dataModel)
    {
        return new FooViewModel
        {
            Count = dataModel.Count
        };
    }
}
```

## Links

Jimmy Bogart: AutoMapper's Design Philosophy: https://jimmybogard.com/automappers-design-philosophy/

In which Jimmy talks about when to use AutoMapper and when not to. I agree, but sadly, when I open up an existing codebase and find that it uses AutoMapper, I always find these bad uses present. Always.

Cezary Piątek: The reasons behind why I don't use AutoMapper. https://cezarypiatek.github.io/post/why-i-dont-use-automapper/

Stackexchange question: https://softwareengineering.stackexchange.com/questions/167250/am-i-wrong-in-thinking-that-needing-something-like-automapper-is-an-indication-o