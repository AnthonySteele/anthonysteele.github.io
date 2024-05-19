# Testing

Most unit testing in CSharp, ASP and .NET code could be done better. I mean in ways that produce better outcomes, better code and make your life as a developer easier.

Below I present a better way that you might already be aware of but probably would benefit from using more.

 I was sold on the better way not from a good logical argument but by working with these approaches. This is all from experience, I have seen these code testing styles, for years on end.

## Preamble

[Interfaces are overused, mocks are overused](https://www.anthonysteele.co.uk/InterfacesAreOverused) but [the test pyramid is still relevant](https://www.anthonysteele.co.uk/TestPyramid).

Most tests that we see in practice are too closely coupled to the code under test.

There is [demo code for this blog post](https://github.com/AnthonySteele/CoupledTestDemo), so that the techniques can be worked through. But bear in mind that this code does nothing, it is merely a reworking of the sample ASP "weather forecast controller". In itself, it is far to simple to need the testing done here. But it must be small to be a readable stand-in for a much larger app, that would have many more classes with multiple methods.

There is a Controller, that calls a service that calls a repository that presumably does the data retrieval. So far so mundane.

The repository in a real app would be e.g. a concrete dependency on a database. We cannot unit test it, so we must give it an interface, and then swap in a different implementation for tests, and test the rest of the app without the real repository.

The same app is tested in multiple ways.

## Isolated style

Isolated, "Mock the interface" tests.

Sadly this is the default. I have seen a co-worker get very good at this style. While I was saying "100% coverage isn't necessarily, important or even always possible", they turned in 100% coverage, even exception handling. And yet the app was still a terrible codebase to work on. Tech debt was the main impediment to process, but refactorings just didn't happen.

## The Isolated tests

[In `WeatherForecastControllerIsolatedTests.cs`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsWithMocks/WeatherForecastControllerIsolatedTests.cs), Each class is tested in isolation with mocks. The test coverage is high. mocking code is verbose and repetitive.

A mock repository is injected into the service, and then separately, a mock service is injected into the controller. Both service and repository must have interfaces for this.

## Sociable style

This style is called ["sociable unit tests"](https://martinfowler.com/bliki/UnitTest.html) where an assembled subsystem is tested.

## The Sociable tests

[In `WeatherForecastControllerSociableTests.cs`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsWithMocks/WeatherForecastControllerSociableTests.cs) we test an assembled stack of real controller, real service and mock repository.

This is better in some ways. We focus a bit more on outcome not implementation.

The mocking code is extracted for re-use, which makes it less verbose.

If this approach is taken, then we can refactor the service at will. The service does not even need to have an interface at all, and this extra code should be deleted. You should be able to "extract class" as a refactoring on the code and still verify that nothing is broken.

## Decoupled style

For the last, [decoupled style](https://github.com/AnthonySteele/CoupledTestDemo/tree/main/WeatherServiceTestsHost), it took a while for me to get used to it when used as the main engine of unit testing.

I started by thinking "this is a lot of indirection to accomplish the same outcome". And then a few months later I noticed that I had got into the habit of starting a feature task by making a failing test. Without touching a single line app code I was able to add a test that "when _new thing_ happens, then _outcome_ should result. It felt like starting a building work by putting up a scaffold that would support the structure safely. The tricky part was done when I had a failing test.

This was a breakthrough. "Test first" is good, or so I had always heard. Yet it wasn't prevalent. Why? Was this purely because I lacked the self-discipline or the innate skill? But instead it seems that a major factor was that _the test style before now did not encourage test first coding_.

Consider this: if "refactoring" is changing code under test, then how well can you refactor if any change breaks tests?

With this approach, we have freedom to refactor the code fairly freely. Split 1 controller into 2? No problem, as long as it's under test. Do away with controllers entirely and use [minimal web API routes](https://learn.microsoft.com/en-us/aspnet/core/tutorials/min-web-api) instead? No problem, it's under test!

### But is it _unit_ tests?

My view is that these outside-in tests are unit tests. Simple as that, end of discussion. However, this is mostly a semantic distinction, you can get the same value from them if you think otherwise.

But, the idea that they must be "integration" because they test multiple classes is IMHO a complete misunderstanding of what is being "integrated".

What is a "unit" in "unit tests" anyway? Views on what's a "unit" that I have heard  that I have sympathy for include:

* It's called a neutral term, "unit" because it's your choice of what that unit is. If it was always "method" it would be called a "method test", and it's not called that, therefore it's not always that.
* It tests a unit of app functionality, [a single logical concept in the system](https://www.artofunittesting.com/definition-of-a-unit-test).

Ultimately the only view that's wrong and harmful is that "a unit test always tests a method on a class, in isolation".

Consider the definition of unit tests, where "A test is not a unit test if: It talks to the database; It communicates across the network; It touches the file system" (from ["A Set of Unit Testing Rules", Michael Feathers, 2005](https://www.artima.com/weblogs/viewpost.jsp?thread=126923) )   - these decoupled tests _meet all of those criteria_.

Consider the other definition of unit tests, where they are small, fast, cheap, numerous, can be run concurrently etc: these decoupled tests meet all of those criteria too.

By reasonable and pragmatic definitions, these are unit tests.

> "Unit tests are completely isolated from each other, creating their test fixtures from scratch each time." the word unit in unit testing refers to the test itself: unit tests are isolated from other tests. Beck argues that "tests should be coupled to the behaviour of code and decoupled from the structure of code."

https://www.infoq.com/articles/unit-testing-approach/

## the demo app decoupled

We use the [Test Host](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests). contrary to what that page says, this host is not _just_ for "integration tests" that "include the app's supporting infrastructure, such as the database, file system". It depends on how you set it up.

 The demo app has [a `TestApplicationFactory`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsHost/TestApplicationFactory.cs) that  replaces repositories with mocks - there is only one in this simple case, but it can be as many as needed.

There's a bit of overhead to get it all set up, but this is a once-off cost so it matters less on larger apps. With a few helper classes it's fairly transparent. That is demonstrated with the `TestContext` This can be shared for even better performance. I have never found this technique to be "too slow".

It's clear that it tests more of the application as well: if you mess up your http configuration so that the route is wrong, or forget to register a necessary service, you'll know it.

 We also demo [a `FakeWeatherForecastDataStore`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsHost/FakeWeatherForecastDataStore.cs) a fake implementation, instead of a mock using a mocking tool. It tracks calls with a `CallCount` property. This technique is also underused. In many cases, this is simpler, clearer code than the equivalent using mocking framework. What I typically find that if it's a bit more verbose - more lines, of code, but simpler lines of code - than the mocking framework equivalent. But then the mocking equivalent gets repeated multiple times in the codebase, adding up to far more lines of code then declaring a "Fake data store" or "In-memory repository" once.

## End note

I don't advocate for these "decoupled" tests to be the _only_ kind of test, just the _default_ kind. i.e. about 80% of the test coverage, depending on the specifics of the app. There will be business logic cases where you are better off dropping down to a class-level test and pumping many test cases into that subsystem. Even then, these might be "sociable tests" that cover multiple classes, as the current exact subdivision of the code into classes is _not the test's concern at all_.

I didn't come to this position out of theoretical reasoning: This worked for me, even better than I thought it would, and it could work for you too.

Use TestHost for more tests. If you find yourself unable to do a simple "extract class" refactoring because it would both break existing tests and the new class would require new tests, they something is wrong: The app code is too coupled to the test code.
