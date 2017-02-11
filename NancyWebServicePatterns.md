# Patterns for web services in Nancy

I wrote this in February 2014. It is hosted here again for reference.

## Original text
 
I wanted to write a bit about C# and HTTP and some patterns of code in [Nancy](http://nancyfx.org/) to tie them together. This is informed by some of the things that I've been doing at work at [7digital](https://www.7digital.com/). The lessons that I have learned are due to 7digital, but the faults and opinions in this post are mine.

The goal in a lot of our coding is to integrate small services over HTTP [into an API](http://developer.7digital.com/resources/api-docs/introduction). We use a few different web frameworks, including [Nancy](http://nancyfx.org/). The main things that we want from Nancy and other web frameworks are:

## Endpoints

* They must give us HTTP endpoints. There will be at least one endpoint in the project and usually more, but more than 10 in the same service is rare. This is [microservices as a SOA strategy](https://en.wikipedia.org/wiki/Service-oriented_architecture).
* We use different HTTP verbs as appropriate, but most of the endpoints are for the verb `GET`. Most of the rest respond to `POST`.
* Endpoints return data. We are not doing html views in the services. There is a C# [Data Transfer Object (DTO)](http://en.wikipedia.org/wiki/Data_transfer_object) to represent that data.
* We always want to do [standard content negotiation](http://en.wikipedia.org/wiki/Content_negotiation) so that the client gets the DTO formatted as XML or Json.
* There is always a DTO to send. Even a complete failure is turned into an error DTO (with response code `500 Internal Server Error`).
* We occasionally want to set the response code to something other than `200 OK`.


Nancy is a good fit to this scenario, but I want to talk about some strategies for scaling it beyond simple cases.

## Digression: Modern websites

Nancy is a good fit for the Api scenario, but if I was starting a website right now and targeting only modern browsers, I would use [AngularJS](https://angularjs.org/) for the front end, maybe with [bootstrap](http://getbootstrap.com/) (I'm no designer!) and ng-boilerplate for the jumpstart and build automation tools. Once you have that, you find that there is little or no server-side html generation. Just data endpoints that serve Json. So the scenario is again similar and Nancy is a good fit.

## The simplest case

Here is the simplest case in Nancy

```csharp
public class HelloModule : NancyModule
{
	public HelloModule()
	{
		Get["/"] = _ => "Hello World!";
	}
}
```

This one-line "hello world" is great advertisement for Nancy, so of course we use it as a template for a service that’s more complex than this. If you just extend the pattern without refactoring it and add code inside the handler inside the constructor, you run into issues. I'm going to use an extended example of a web site for a cake shop for a slightly more complex site:

```csharp
public class CakeModule : NancyModule
{
	public CakeModule()
	{
		Get["/cake/"] = _ =>
		{
			var cakes = new List<Cake>();

			// code to get cakes data

			return cakes;
		};

		Get["/cake/{id}"] = _ =>
		{
			int id = _.id;

			// code to get data for a cake
			var response = FindCake(id);

			return response;
		};

		// another handler, with more code...
		
	} // Constructor of Doom ends here!
}
```

You can end up with a 100 or 200 line Constructor of Doom containing everything nested inside it. It happens. Not many people like code like that, but it's not Nancy's fault - it's easily fixable by changing the pattern.

## Refactor Number One: SRP

The first tool to reach for is the [standard Single Responsibility Principle](https://en.wikipedia.org/wiki/Single_responsibility_principle) and [Dependency Injection](http://en.wikipedia.org/wiki/Dependency_injection), likely using [an IoC Container](http://www.hanselman.com/blog/ListOfNETDependencyInjectionContainersIOC.aspx). In the cake shop web site, we might have a `CakeRepository`. The interface to it might look something like this:

```csharp
public interface ICakeRepository
{
	<List>Cake> All();
	Cake Get(int id);
}
```

Now the data store access can be integration tested separately, and the Nancy module can be unit tested with a mock repository.

```csharp
public class CakeModule : NancyModule
{
	public CakeModule(ICakeRepository cakeRepository)
	{
		Get["/cake/"] = _ => cakeRepository.All();
		Get["/cake/{id}"] = _ => cakeRepository.Get(_.id);
	}
}
```

## Digression: The Single Responsibility of a Nancy module

The Single Responsibility Principle urges us to find the single thing that a class does, to make sure that it does it well and move out code that does anything else. The _NancyModule_ or equivalently the _ServiceStack Service_ or the _ASP MVC Controller_ has a responsibility given to it by the framework: It mediates from HTTP into code and back again. It puts a HTTP request into your code, and allows you to return a HTTP response. It is where the rubber meets the road.

So we want to make the module contain code that is about turning HTTP into c# and back again, and leave all the other processing to other classes behind it.

And yet there are always complications to inspecting the input and returning the output: we may want exceptions to cause `50x` status codes, or we may want to control the status code from a particular exception. We want validation failures to cause `400 Bad Request` responses. Sometimes we may want to set the response code manually. The rest of the time it should default to `200 OK`.

## Refactor number Two: Public methods and objects

Somewhere along the line it becomes useful to have a request object that holds the fields extracted from the HTTP request, and a response object for the data returned. These objects may be used on just one endpoint or shared across a few. The request data is simple in this case:

```csharp
public class CakeRequest
{
	public int Id { get; set; } 
}
```

The request has all the advantages of a strongly typed language; avoiding [injection attacks](https://www.owasp.org/index.php/Injection_Flaws) where the value contains malicious text and [mass assignment attacks](http://odetocode.com/Blogs/scott/archive/2012/03/11/complete-guide-to-mass-assignment-in-asp-net-mvc.aspx) where unwanted fields are set, and allows the use of [fluent validation](http://fluentvalidation.codeplex.com/), a great library to validate request values. The fluent validator for this request looks like this:

```csharp
public class CakeRequestValidator : AbstractValidator<CakeRequest>
{
	public CakeRequestValidator()
	{
		RuleFor(cr => cr.Id).GreaterThan(0);
	}
}
```

The response is also simple in this case:

```csharp
public class CakesResponse
{
	public List<Cake> Cakes { get; set; }
}
```

I know that all of this looks like overkill for this simple example, but the trick as the code grows is to introduce this infrastructure just before you need it. Too early and it's a burden; too late and lack of it is a burden. Fluent Validation is really worthwhile when you have multiple endpoints with multiple parameters, some of which are numeric and others have different constraints. The `CakeResponse` is also a simple object, but would be necessary as soon as you wanted to return things such as paging data with the results list.

With this in place, we can use Nancy's `BindAndValidate` helper. In addition, I find it useful to separate the request parsing into the request DTO from the response generation. The response generation can then be made a public method:

```csharp
public class CakeModule : NancyModule
{
	private readonly ICakeRepository _cakeRepository;

	public CakeModule(ICakeRepository cakeRepository)
	{
		_cakeRepository = cakeRepository;

		Get["/cake/"] = _ => AllCakes();

		Get["/cake/{id}"] = _ =>
			{
				var request = this.BindAndValidate<CakeRequest>();
				return GetCake(request);
			};
	}

	public CakesResponse AllCakes()
	{
		return new CakesResponse
		{
			Cakes = _cakeRepository.All()
		};
	}

	public Cake GetCake(CakeRequest request)
	{
		return _cakeRepository.Get(request.Id);
	}
}
```

There is complexity that you may need to add here: for instance, if you can't find a cake with the given Id, you want to return a HTTP `404 Not Found`. You can do this as follows after relaxing the `GetCake` method to return an object:

```csharp
public object GetCake(CakeRequest request)
{
	var cakeById = _cakeRepository.Get(request.Id);
	if (cakeById == null)
	{
		return HttpStatusCode.NotFound;
	}
	return cakeById;
}
```

This logic belongs in the Nancy module as it's about mediating c# code into HTTP. but why make it a public method? Aside from breaking up a long method at a logical point, the advantage of making it a public method is that it's easy to test without any HTTP mocking, just a simple mock repository (using [NUnit](http://nunit.org/) and [NSubstitute](https://nsubstitute.github.io/) in this example).

```csharp
[Test]
public void Should_return_404_when_cake_is_not_found()
{
	var repo = Substitute.For<ICakeRepository>();
	repo.Get(Arg.Any<int>()).Returns((Cake)null);
	var module = new CakeModule(repo);

	var response = module.GetCake(new CakeRequest());

	Assert.That(response, Is.EqualTo(HttpStatusCode.NotFound));
}
```

## Refactor Number Three: Modules

You can have as many [Nancy modules](https://github.com/NancyFx/Nancy/wiki/Exploring-the-nancy-module) as you like in the project. If the module defines two or three endpoints that's fine by me, but if you have six or seven in the same module then they had better be very closely related. When to split the module a judgement call based on if they have a natural division and how much code they have in common, but a large Nancy module with lots of endpoints is a "[code smell](https://blog.codinghorror.com/code-smells/)". The same applies to a [ServiceStack](https://servicestack.net/) Service or a [ASP MVC](https://www.asp.net/mvc) Controller.

The modules don't need to all live in the same folder. Take advantage of this to make "[Feature folders](http://www.slideshare.net/Anthony_Steele_/feature-folders)" containing the Nancy module, DTOs, validators and other files that make up a feature.

## Next time

And that's it. I have a least one more pattern in mind, but that will have to wait for another blog post. For now, it's worth noting that instead of `BindAndValidate`, you will want to do this:

```csharp
var request = this.BindAndValidate<CakeRequest>();
if (! ModelValidationResult.IsValid)
{
	return HttpStatusCode.BadRequest;
}
return GetCake(request);
```

But this becomes verbose boilerplate if you do it on every request path that has any parameters. And it hasn't covered that we want to return content-negotiated DTO populated with data from `ModelValidationResult.Errors` along with the status code in all cases. [This is continued in part two](.\NancyWebServicePatterns2).
