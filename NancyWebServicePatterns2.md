# More patterns for web services in Nancy

I wrote this in March 2014. It is hosted here again for reference.

## Original text

To carry on from Part one about patterns of web services in Nancy, we have a functional approach, but I'll start with an easy refactoring about modules:

## Refactor Number Three: Modules

You can have as many Nancy modules as you like in the project. If the module defines two or three endpoints that's fine by me, but if you have six or seven in the same module then they had better be very closely related. When to split the module a judgement call based on if they have a natural division and how much code they have in common, but a large Nancy module with lots of endpoints is a "code smell". The same applies to a ServiceStack Service or a ASP MVC Controller.

The modules don't need to all live in the same folder. Take advantage of this to make "Feature folders" containing the Nancy module, DTOs, validators and other files that make up a feature.

## The Boilerplate Problem

In the previous blog post I showed the code used to validate input and return a `400 Bad Request` if it fails. In the cakeshop example, this works if I request the url `/cake/0` which fails validation in the fluent validator due to the `RuleFor(cr => cr.Id).GreaterThan(0);` rule. But if I request `/cake/chocolate` then I sadly get a ModelBindingException in a `500 Internal Server Error` since "chocolate" can't be turned into an int at all.

So the boilerplate becomes:

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

I also may want to do things like have an exception class so somewhere deeper in the code I can do throw `HttpException.BadRequest();` and have this result in that response code. Nancy's Pipeline handlers are supposed to be the right way to deal with exceptions and other cross-curring concerns, but I didn't find a way to do `module.Negotiate.With...` in a Pipeline handler, or any simple equivalent. And I _always_ want to do content negotiation.

With each additional thing, the code in the handler becomes even more complex. At this point I felt that the boilerplate that I was copying around was getting out of hand and I wanted to extract the pattern. And it's a well-defined pattern - the only things that differ are the type of the input model ( call it `TIn`), the response type (`TOut`) and the response generation function such as `GetCake` which can be typed as `Func<TIn, TOut>`. Then there's a variation when there are no request parameters and thus no request model, and the response generation is just `Func<TOut>`.

Refactor Number Four: Shimming Nancy with functions.
So I found another way, with a little generic functional code. It's a function that we pass the response generation function into, and which runs it with the boilerplate around it. A Nancy handler is actually a `Func<dynamic, dynamic>`, so any function that meets this type signature can be used. Often we write little adapters to other types without thinking about it, e.g. `Get["/cakes"] = _ => GetCakes();` contains an function `_ => GetCakes()` which adapts this `Func<dynamic, dynamic>` to `Func<CakeResponse>`.

The input dynamic object is the "dynamic dictionary". We don't actually use it directly at all when we get the params via `module.BindAndValidate`.

I came up with a generic functional "shim" to use in all cases. 

```csharp
public static object RunHandler<TIn, TOut>(this NancyModule module, Func<TIn, TOut> handler)
{
	try
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
	catch (HttpException hEx)
	{
		return module.Negotiate.WithStatusCode(hEx.StatusCode).WithModel(hEx.Content);
	}
}
```

Use it like this:

```csharp
Get["/cake/details/{cakeId}"] = _ => this.RunHandler<CakeRequest, CakeDetailsResponse>(GetCakeDetails);
```

I made four variations on the same theme, for requests with and without a request DTO, and for synchronous and asynchronous handlers.

A different approach to the same code would be to have a method on a module base class descended from `NancyModule`. But I find this more versatile and less ceremony. You opt into these handlers on each endpoint that you want to work that way, regardless of base class, but they are one-liners so there's minimal extra code to do so.

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

We can do an async version using Nancy's way of defining async handlers:
`Get["/cakes", true] = async (_, ct) => await this.RunHandlerAsync(AllCakesAsync);`

You can also wrap up the handler creation with another extension method, e.g:

```csharp
public static void GetHandlerAsync<TIn, TOut>(this NancyModule module, string path, Func<TIn, Task<TOut>> handler)
{
	module.Get[path, true] = async (x, ctx) => await RunHandlerAsync(module, handler);
}
```

Which is actually pretty simple. The hardest part was getting the type signature right! So using that, instead of this code:

```csharp
Get["/cakes", true] = async (_, ct) => await this.RunHandlerAsync<CakesResponse>(AllCakesAsync);
Get["/cake/{id}", true] = async (_, ct) => await this.RunHandlerAsync<CakeRequest, object>(GetCakeAsync);
```

We have:

```csharp
this.GetHandlerAsync<CakesResponse>("/cakes", AllCakesAsync);
this.GetHandlerAsync<CakeRequest, object>("/cake/{id}", GetCakeAsync);
```

The code is about as long, but to my eye far less noisy to read.

A full copy of the code for the "GetHandler" and "RunHandler" extensions is here. You are welcome to use it "as is" or to cut, paste and alter it; or just take the idea and implement something similar to meet your own requirements.

I have updated this approach in part three: Nancy handler functions revisited.