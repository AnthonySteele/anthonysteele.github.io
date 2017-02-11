# More patterns for web services in Nancy

I wrote this in March 2014. It is hosted here again for reference, with some edits to remove ideas that I have later decided are bad, lest the reader be lead astray.

You are of course still welcome to to cut, paste and use it "as is" or to alter it; or just take the idea and implement something similar to meet your own requirements.

## Original text of part 2

## The Boilerplate Problem

In [the previous blog post]((./NancyWebServicePatterns)) I showed the code used to validate input and return a `400 Bad Request` if it fails. In the cakeshop example, this works if I request the url `/cake/0` which fails validation in the [fluent validator](https://github.com/JeremySkinner/FluentValidation) due to the `RuleFor(cr => cr.Id).GreaterThan(0);` rule. But if I request `/cake/chocolate` then I sadly get a `ModelBindingException` in a `500 Internal Server Error` since "chocolate" can't be turned into an `int` at all. This should also be a `400` response.

So to fix this, the boilerplate becomes:

```csharp
CakeRequest model;
try
{
	model = module.BindAndValidate<CakeRequest>();
	if (!module.ModelValidationResult.IsValid)
	{
		return module.Negotiate.RespondWithValidationFailure(module.ModelValidationResult);
	}
}
catch (ModelBindingException)
{
	return module.Negotiate.RespondWithValidationFailure("Model binding failed");
}
return GetCake(model);
```

I initially put in code so that I could have an exception class, and somewhere deeper in the code I could do `throw HttpException.BadRequest();` and have this result in that response code.  Nancy's Pipeline handlers are supposed to be the right way to deal with exceptions and other cross-cutting concerns, but I didn't find a way to do `module.Negotiate.With...` in a Pipeline handler, or any simple equivalent. And I _always_ want to do content negotiation.


But we decided that this was a bad idea - it is [flow control by exception](http://softwareengineering.stackexchange.com/questions/189222/are-exceptions-as-control-flow-considered-a-serious-antipattern-if-so-why), and was not in line with the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single_responsibility_principle) -  there are more details about this change under "Nancy handler functions revisited" below.

With each additional thing, the code in the handler becomes even more complex. At this point I felt that the boilerplate that I was copying around was getting out of hand and I wanted to extract the pattern. And it's a well-defined pattern - the only things that differ are the type of the input model ( call it `TIn`), the response type (`TOut`) and the response generation function such as `GetCake` which can be typed as `Func<TIn, TOut>`. Then there's a variation when there are no request parameters and thus no request model, and the response generation is just `Func<TOut>`.

But the result can be a DTO or also HTTP status response, e.g. `return HttpStatusCode.BadRequest;` so to cover these two cases `TOut` must always be `object` - there are more details about this change under "Nancy handler functions revisited" below.

## Refactor Number Four: Shimming Nancy with functions.

So I found a way with a little generic functional code. It's a function that we pass the response generation function into, and which runs it with the boilerplate around it. A Nancy handler is actually a `Func<dynamic, dynamic>`, so any function that meets this type signature can be used. Often we write little adapters to other types without thinking about it, e.g. `Get["/cakes"] = _ => GetCakes();` contains an function `_ => GetCakes()` which adapts this `Func<dynamic, dynamic>` to `Func<CakeResponse>`.

The input dynamic object is the "[dynamic dictionary](https://github.com/NancyFx/Nancy/wiki/Taking-a-look-at-the-DynamicDictionary)". We don't actually use it directly at all when we get the params via `module.BindAndValidate`.

I came up with a generic functional "[shim](http://www.merriam-webster.com/dictionary/shim)" or wrapper to use in all cases. 

```csharp
public static object RunHandler<TIn>(this NancyModule module, Func<TIn, object> handler)
{
	TIn model;
	try
	{
		model = module.BindAndValidate<TIn>();
		if (!module.ModelValidationResult.IsValid)
		{
			return module.Negotiate.RespondWithValidationFailure(module.ModelValidationResult);
		}
	}
	catch (ModelBindingException)
	{
		return module.Negotiate.RespondWithValidationFailure("Model binding failed");
	}

	return handler(model);
}
```

Use it like this:

```csharp
Get["/cake/details/{cakeId}"] = _ => this.RunHandler<CakeRequest>(GetCakeDetails);
```

And with another generic helper because attaching a handler for HTTP `Get` on a route is very common:

```csharp
public static void GetHandler<TIn>(this NancyModule module, string path, Func<TIn, object> handler)
{
  module.Get[path] = _ => RunHandler(module, handler);
}
```

Is becomes even simpler:

```csharp
this.GetHandler("/cake/details/{cakeId}", GetCakeDetails);
```

A different approach to the same code would be to have a method on a module base class descended from `NancyModule`. But I find this more versatile and less ceremony. You opt into these handlers on each endpoint that you want to work that way, regardless of base class, but they are one-liners so there's minimal extra code to do so.

I made variations on the same theme, for requests with and without a request DTO, and for synchronous and asynchronous handlers.


## Extending to async

I like async, not just because it's a cool new language feature, but because any operation that goes to a different machine, be it a database or a web service, is inherently asynchronous. You have to wait. It may be quick or slow, or it may time out and fail. The only guarantee is that it won't complete instantly, so synchronous calls must tie up a thread waiting.

Assuming that we have altered the cake module to support async code as follows:

```csharp
public async Task<CakesReponse> AllCakesAsync()
{
	return new CakesResponse
	{
		Cakes = await _cakeRepository.GetCakesAsync()
	};
}

public async Task<object> GetCakeAsync(CakeRequest request)
{
	var cakeById = await _cakeRepository.GetAsync(request.Id);
	if (cakeById == null)
	{
		return HttpStatusCode.NotFound;
	}
	return cakeById;
}
```

We can do an async version using [Nancy's way of defining async handlers](https://github.com/NancyFx/Nancy/wiki/Async):
`Get["/cakes", true] = async (_, ct) => await this.RunHandlerAsync(AllCakesAsync);`

You can also wrap up the handler creation with another extension method, e.g:

```csharp
public static void GetHandlerAsync<TIn>(this NancyModule module, string path, Func<TIn, Task<object>> handler)
{
	module.Get[path, true] = async (x, ctx) => await RunHandlerAsync(module, handler);
}
```

Which is actually pretty simple. The hardest part was getting the type signature right! So using that, instead of this code:

```csharp
Get["/cakes", true] = async (_, ct) => await this.RunHandlerAsync(AllCakesAsync);
Get["/cake/{id}", true] = async (_, ct) => await this.RunHandlerAsync<CakeRequest>(GetCakeAsync);
```

We have:

```csharp
this.GetHandlerAsync("/cakes", AllCakesAsync);
this.GetHandlerAsync<CakeRequest>("/cake/{id}", GetCakeAsync);
```

The code is far less noisy to read.

A full copy of the code for the `GetHandler` and `RunHandler` extensions is at the bottom of this page. 


### Part Three: Nancy handler functions revisited


I have an update on our experience with using Nancy for web APIs. Since I last wrote in part two we have made a couple of changes which make the code simpler.

Firstly we removed the `HttpException` class. It turned out to be an antipattern.

Recall that in [Part One](./NancyWebServicePatterns) I mentioned the [Single Responsibily Principle](https://en.wikipedia.org/wiki/Single_responsibility_principle) dictates that the Nancy module handles the conversion from HTTP to c# code and back, and only that responsibility. The corollary is that _nothing else does this_. Using the `HttpException` class is against this rule - it allows you to push knowledge of HTTP status codes down into layers that are better off knowing nothing about them, or which may even be engineered for a more general usage, not just on a website. It's a form of the [Flow control via exception anti-pattern](http://softwareengineering.stackexchange.com/questions/189222/are-exceptions-as-control-flow-considered-a-serious-antipattern-if-so-why). It was mostly used in Nancy modules anyway, where there are simpler ways to return http statuses. So we removed the class. 

The consequences give rise to a simplification - the Nancy module's handler method must now be able to return different things - the "happy path" result DTO, an error object or a HTTP response code. Therefore these methods now always return `object`. This removed the generic type `TOut` leaving just `TIn`. Nancy casts the response as dynamic anyway so there's no additional overhead to this pattern.

Instead of throwing http exceptions, domain services can return null when the object is not found (or you could design a suitable result to return that indicated "not found"). It is up to the handler method in the Nancy module to translate this into HTTP 's language, e.g. convert a null to a `404` response code. Also, an expected failure, for instance when the item id supplied in the request doesn't refer to an existent record, are best handled by conventional flow control not exceptions.

There are no changes to what I said earlier about model binding. We still think that "[Async](https://github.com/NancyFx/Nancy/wiki/Async) all the network operations" is the way forward. We [arrange the code into feature folders](http://www.slideshare.net/Anthony_Steele_/feature-folders).

We use base classes for Nancy modules as well, for checking user permissions on API endpoints. Having handler functions as extension methods not base class makes them a simple opt-in, avoids having multiple base classes, and sidesteps the decisions of how to arrange the type hierarchy of unrelated concerns.

A sample module, with async and async methods, with and without input request objects, looks like this:

```csharp
public class CakeModule : NancyModule
{
	public CakeModule()
	{
		this.GetHandler("/cakes", AllCakes);
		this.GetHandlerAsync<CakeRequest>("/cake/{id}", GetCakeAsync);
	}

	public CakesReponse AllCakes()
	{
		return new CakesResponse
		{
			Cakes = _cakeRepository.GetCakes()
		};
	}

	public async Task<object> GetCakeAsync(CakeRequest request)
	{
		var cakeById = await _cakeRepository.GetAsync(request.Id);
		if (cakeById == null)
		{
			return HttpStatusCode.NotFound;
		}
		return cakeById;
	}
}
```

the final code for the extensions looks like this:

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

using Nancy;
using Nancy.ModelBinding;
using Nancy.Responses.Negotiation;
using Nancy.Validation;


public static class ModuleExtensions
{
	public static void GetHandler(this NancyModule module, string path, Func<object> handler)
	{
		module.Get[path] = _ => RunHandler(module, handler);
	}

	public static void GetHandler<TIn>(this NancyModule module, string path, Func<TIn, object> handler)
	{
		module.Get[path] = _ => RunHandler(module, handler);
	}

	public static void GetHandlerAsync<TIn>(this NancyModule module, string path, Func<TIn, Task<object>> handler)
	{
		module.Get[path, true] = async (x, ctx) => await RunHandlerAsync(module, handler);
	}

	public static void GetHandlerAsync(this NancyModule module, string path, Func<Task<object>> handler)
	{
		module.Get[path, true] = async (x, ctx) => await RunHandlerAsync(module, handler);
	}


	public static object RunHandler(this NancyModule module, Func<object> handler)
	{
		return handler();
	}

	public static async Task<object> RunHandlerAsync(this NancyModule module, Func<Task<object>> handler)
	{
		return await handler();
	}

	public static object RunHandler<TIn>(this NancyModule module, Func<TIn, object> handler)
	{
		TIn model;
		try
		{
			model = module.BindAndValidate<TIn>();
			if (!module.ModelValidationResult.IsValid)
			{
				return module.Negotiate.RespondWithValidationFailure(module.ModelValidationResult);
			}
		}
		catch (ModelBindingException)
		{
			return module.Negotiate.RespondWithValidationFailure("Model binding failed");
		}

		return handler(model);
	}

	public static async Task<object> RunHandlerAsync<TIn>(this NancyModule module, Func<TIn, Task<object>> handler)
	{
		TIn model;
		try
		{
			model = module.BindAndValidate<TIn>();
			if (!module.ModelValidationResult.IsValid)
			{
				return module.Negotiate.RespondWithValidationFailure(module.ModelValidationResult);
			}
		}
		catch (ModelBindingException)
		{
			return module.Negotiate.RespondWithValidationFailure("Model binding failed");
		}

		return await handler(model);
	}
	
	public static Negotiator RespondWithValidationFailure(this Negotiator negotiate, ModelValidationResult validationResult)
	{
		var model = new ValidationFailedResponse(validationResult);

		return negotiate
			.WithModel(model)
			.WithStatusCode(HttpStatusCode.BadRequest);
	}

	public static object RespondWithValidationFailure(this Negotiator negotiate, string message)
	{
		var model = new ValidationFailedResponse(message);

		return negotiate
			.WithModel(model)
			.WithStatusCode(HttpStatusCode.BadRequest);
	}
}
```

And the response DTO

```csharp
public class ValidationFailedResponse
{
	public List<string> Messages { get; set; }
		
	public ValidationFailedResponse()
	{}

	public ValidationFailedResponse(ModelValidationResult validationResult)
	{
		Messages = new List<string>();
		ErrorsToStrings(validationResult);
	}

	public ValidationFailedResponse(string message)
	{
		Messages = new List<string>
			{
				message
			};
	}

	private void ErrorsToStrings(ModelValidationResult validationResult)
	{
		foreach (var errorGroup in validationResult.Errors)
		{
			foreach (var error in errorGroup.Value)
			{
				Messages.Add(error.ErrorMessage);
			}
		}
	}
}
```