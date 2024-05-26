# Decouple your unit tests

Most unit testing in C#, .NET and ASP code could be done better. Less coupled. And that would produce better outcomes, better code, and make your life as a developer easier.

Below I present a better way that you might already be aware of but probably would benefit from using more.

 I was sold on the better, less coupled way not from a good logical argument but by working with these approaches. This is all from experience, I have seen these code testing styles, for years on end.

## Preamble

[Interfaces are overused, mocks are overused](https://www.anthonysteele.co.uk/InterfacesAreOverused) but [the test pyramid is still relevant](https://www.anthonysteele.co.uk/TestPyramid).

Most tests that we see in practice are too closely coupled to the code under test.

[There is demo code in a GitHub repository for this blog post](https://github.com/AnthonySteele/CoupledTestDemo), so that the techniques can be worked through. But bear in mind that this code does nothing, it is merely a reworking of the sample ASP "weather forecast controller". In itself, it is far to simple to need the testing done here. But it must be small to be a readable demo stand-in for a much larger application, that would have many more classes with multiple methods arranged in more layers.

The demo code is a .NET 8 [ASP.NET Core](https://dotnet.microsoft.com/en-us/apps/aspnet) Web Api. It has [a controller](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherService/Controllers/WeatherForecastController.cs), that calls [a service](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherService/Controllers/WeatherForecastService.cs) that calls a repository that presumably does the data retrieval. So far, so familiar.

In a real application there would be sufficient complexity to make multiple layers a good iea, and  multiple repositories that would have e.g. a concrete dependency on a database, along with other ways to get and send data to http services, message queues etc.

The one repository stands in for all of those. We cannot _unit test_ this repository, so we must give it an interface, and then swap in a different implementation for tests. Then we can test the rest of the application without the real repository.

The same app is tested in multiple ways.

## Isolated style

The first example has isolated, "Mock the interface" tests. Sadly this is the default, by far the most common style.

I have seen a co-worker get very good at this style. While I was saying "100% coverage isn't necessarily, important or even always possible", they turned in 100% coverage, even exception handling. And yet the app was still a terrible codebase to work on. Tech debt was the main impediment to process, but refactorings just didn't happen.

A large body of tests in this style can be as much a tedious liability as it is a useful safety net.

### The Isolated tests

```csharp
    [Fact]
    public void Controller_Should_CallService()
    {
        var service = Substitute.For<IWeatherForecastService>();
        service.Get().Returns(new List<WeatherForecast>
        {
            new WeatherForecast
            {
                Date = DateOnly.FromDateTime(DateTime.Today),
                Summary = "Testy",
                TemperatureC = 20
            }
        });

        var controller = new WeatherForecastController(
            new NullLogger<WeatherForecastController>(), 
            service);

        controller.Get();

        service.Received().Get();
    }

    // etc
```

[In `WeatherForecastControllerIsolatedTests.cs`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsWithMocks/WeatherForecastControllerIsolatedTests.cs), Each class is tested in isolation with mocks. The test coverage is high. Mocking code is everywhere, it is verbose and repetitive.

A mock repository is injected into the service, and then separately, a mock service is injected into the controller. Both service and repository must have interfaces for this. We verify the the correct method was called.

The code smells that we typically see include tests with dozens of mocks, verbose and repetitive mock setup to test very little logic, and even for extra insanity, [key business logic in AutoMapper mappers](https://www.anthonysteele.co.uk/AgainstAutoMapper), which are called in the middle of the code, but are not part of the test, but mocked for test.

## Sociable style

```csharp
    [Fact]
    public void Controller_Should_ReturnExpectedData()
    {
        var forecastDataStore = CreateMockWeatherForecastDataStore();
        var controller = new WeatherForecastController(
            new NullLogger<WeatherForecastController>(), 
            new WeatherForecastService(forecastDataStore));

        var response = controller.Get();

        Assert.NotEmpty(response);
        Assert.Equal("Testy", response.First().Summary);
    }

    /// etc
```

This style is called ["sociable unit tests"](https://martinfowler.com/bliki/UnitTest.html) where an assembled subsystem is tested.

### The Sociable tests

[In `WeatherForecastControllerSociableTests.cs`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsWithMocks/WeatherForecastControllerSociableTests.cs) we test an assembled stack of real controller, real service and mock repository.

The mocking code is extracted for re-use, which makes it less verbose.

This is better in some ways. We focus a bit more on testing outcome not implementation. It's more about "do we return the expected value?" and less "did we call the expected method?".

If this approach is taken, then we can refactor the service at will. The service does not even need to have an interface at all, and this extra declaration should be deleted. We are at the point that the foolish dogma of "always put an interface on every class" is actually useless.

You should also be able to "extract class" as a refactoring in some cases without changing tests, and still verify that nothing is broken.

## Decoupled style

For the last, [decoupled style](https://github.com/AnthonySteele/CoupledTestDemo/tree/main/WeatherServiceTestsHost), this was the last style that I saw. It took a while for me to get used to it when used as the main engine of unit testing.

I started by thinking "this is a lot of indirection to accomplish the same outcome". And then a few months later I noticed that I had got into the habit of starting a feature task by first making a failing test. Without touching a single line app code I was able to add a test that "when _new thing_ happens, then _outcome_ should result." It felt like starting a building work by putting up a scaffold that would support the construction, safely. The tricky part was done already when I had a failing test.

This was a breakthrough. "Test first" is good, or so [I had always heard](https://www.youtube.com/watch?v=fPlBLlE8vOI). Yet it wasn't prevalent in most places where I have worked. They didn't do it much. And I didn't do it much. Why? Was this purely because I lacked the self-discipline or the innate skill? But instead it seems that a major factor was that _the test style before now did not support test-first coding_.

Consider this: if "refactoring" is changing code under test, then how well can you refactor if any change breaks tests?

With this approach, we have freedom to refactor the code under test liberally. The service layer is not doing anything except forwarding calls and could be deleted? Go ahead, tests should still compile and pass. Split one large controller into two? No problem! Do away with controllers entirely and use [minimal web API routes](https://learn.microsoft.com/en-us/aspnet/core/tutorials/min-web-api) instead? Fine, it's under test!

### But is it _Unit_ tests?

My view is yes. These outside-in tests are unit tests. Simple as that. However, this is mostly a semantic distinction, you can get the same value from them if you think otherwise. But you should understand what they do and don't have.

But, the idea that they must be "integration" because they test multiple classes is IMHO a complete misunderstanding of what is being "integrated". It's referring to external dependencies in this case.

What is a "unit" in "unit tests" anyway? Views on what's a "unit" that I have heard  that I have sympathy for are:

* It's called "unit", a neutral term, because it's your choice of what this unit is. If it was always "method" it would be called a "method test". It's not called that, therefore it's not always that.
* It tests a unit of app functionality, [a single logical concept in the system](https://www.artofunittesting.com/definition-of-a-unit-test).
* A unit test [Tests behaviors, not implementation details](https://www.youtube.com/watch?v=EZ05e7EMOLM&t=1428s) - this blog post would be _incomplete_ without referencing "TDD, Where Did It All Go Wrong" by Ian Cooper. Who in turn quotes [Kent Beck](https://www.goodreads.com/book/show/387190.Test_Driven_Development).

Consider the definition of unit tests, where "A test is not a unit test if: It talks to the database; It communicates across the network; It touches the file system" (from ["A Set of Unit Testing Rules", Michael Feathers, 2005](https://www.artima.com/weblogs/viewpost.jsp?thread=126923) )   - these decoupled tests _meet all of those criteria_ by [configuring the test host to replace such dependencies with fakes](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsHost/TestApplicationFactory.cs#L15).

Consider the other definition of unit tests, where they are small, fast, cheap, numerous, reliable, can be run frequently, can be run concurrently: these decoupled tests meet all of those criteria too.

By reasonable and pragmatic definitions, these are unit tests.

Kent Beck:

> "Unit tests are completely isolated from each other, creating their test fixtures from scratch each time." the word unit in unit testing refers to the test itself: unit tests are isolated from other tests. Beck argues that "tests should be coupled to the behaviour of code and decoupled from the structure of code."  
  [The Unit in Unit Testing](https://www.infoq.com/articles/unit-testing-approach/)

Ultimately the only views are wrong and harmful are that "a unit test always tests a method on a class, in isolation" and that "every class has a matching test class, no more and no less is needed".

## the demo app Decoupled

```csharp
    [Fact]
    public async Task GetForecast_Should_ReturnData()
    {
        var response = await _testContext.GetForecastTyped();

        Assert.NotNull(response);
        Assert.NotEmpty(response);
        Assert.Equal("Testy", response.First().Summary);
    }

    // etc
```

We use the [Test Host](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests). Contrary to what that page says, this host is not _just_ for "integration tests" that "include the app's supporting infrastructure, such as the database, file system". They can be mocked here.

```csharp
public class TestApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // this is where services that have concrete dependencies
            // are replaced by fakes/mocks

            RemoveService<IWeatherForecastDataStore>(services);
            services.AddSingleton<IWeatherForecastDataStore>(new FakeWeatherForecastDataStore());
        });
    }
```

 The demo app has [a `TestApplicationFactory`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsHost/TestApplicationFactory.cs) that  replaces repositories with mocks - there is only one that needs to be replaced in this simple app, but it can be as many as needed.

 I have never found this technique to be "too slow".

We can [create the `TestApplicationFactory` new for each test like this](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsHost/WeatherForecastHostTests.cs) or [abstract it into an injectable `TestContext`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsHost/WeatherForecastHostTestsWithSharedContext.cs).
In this case it is shared for even better performance in cases where this doesn't affect the test.

 There's a bit of overhead to get it all set up, but this is a once-off cost so it matters less on larger apps. With a few helper classes it's fairly transparent.
 Bear in mind that as your test suite grows, this layer becomes more worthwhile. So that the test is concise and more readable, and does not specify everything at the http request level.

You can do it all in the test method, including the `WebApplicationFactory` code, but this technique that doesn't scale at all to many tests.

A technique that scales a bit better is doing it all in a test base class; but soon that class becomes far too large and lacks coherence. i.e. "favour composition over inheritance".

 You would be better off with separate helper classes for common code for wrappers and abstractions that you might find in a larger app's test suite.

 The _only_ classes in the app that _need_ interfaces in this design are the ones that need to be mocked for unit testing, because they have dependencies such as databases that can't be unit tested. Other interfaces are noise, and can simply be deleted.

It's clear that it tests more of the application as well: if you mess up your http configuration so that the route is wrong, or forget to register a necessary service, you'll know it, whereas the other tests don't get to that before actual "deploy and run tests on the deployed app" integration tests.

We also demo [a `FakeWeatherForecastDataStore`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsHost/FakeWeatherForecastDataStore.cs). This is a [fake implementation](http://xunitpatterns.com/Fake%20Object.html), instead of a mock using a mocking tool. It tracks calls with a `CallCount` property. This technique is also underused. In many cases, this is simpler, clearer code than the equivalent using mocking framework.

What I typically find that this kind of Fake is simpler, but a bit more verbose - more lines of code, but simpler lines of code - than the mocking framework equivalent. But then the mocking equivalent gets repeated multiple times in the codebase, adding up to far more lines of code than declaring a "Fake data store" or "In-memory repository" once.

## End note

Decouple your unit tests. If you find yourself unable to do a simple "extract class" refactoring because it would both break existing tests and the new class would require new tests, they something is wrong: The app code is too coupled to the test code.

Use [the Test Host](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.testhost) for more of your tests. Push mocks to the edges off the app. Use them for the _sinks_ where state leaves or enters the app.

Favour State-Based Testing over Interaction  testing: favour test that test _outcomes_ such as returning a result, enqueueing a message, or modifying a state in a data store correctly, over tests that test that the code calls the expected _method_.

> State-based testing is always preferred. The primary reason is that it is less coupled to your code. So you can more easily change your code without changing the test.

[Interaction vs State-Based Testing](https://thinkster.io/tutorials/blogs/interaction-vs-state-based-testing)

I don't advocate for these "decoupled" tests to be the _only_ kind of test, just the _default_ kind. About 80% of the test coverage can be decoupled, depending on the specifics of the app. [They are a "Should" recommendation](https://datatracker.ietf.org/doc/html/rfc2119).

 There will be business logic cases where you are better off dropping down to a class-level test and pumping many test cases into that subsystem, instead. Even then, these might be "sociable tests" that cover multiple classes, as the current exact subdivision of the code into classes is _not the test's concern at all_ since you're testing the _behaviour_ of code and not coupled to the _structure_ of code.

I didn't come to this position out of theoretical reasoning: This was given to me by a team already using it. And it worked for me, even better than I thought it would. And it could work for you too.

The second-order conclusions are that like with Continuous Delivery, it's the downstream effects that deliver the big benefits over time.

And that semantic drift happens, such that over time the practice ends up substantially different, easier, and often worse, not giving much of those benefits.

Also that you can work for many years in "good practice" employers, with ever seeing first-hand what good really looks like, or even knowing that better exists. After all "we do testing" ticks the box.
