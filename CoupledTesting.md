# Testing  and Coupling

Most unit testing in C#, .NET and ASP code could be done better. I mean in ways that produce better outcomes, better code, and make your life as a developer easier.

Below I present a better way that you might already be aware of but probably would benefit from using more.

 I was sold on the better way not from a good logical argument but by working with these approaches. This is all from experience, I have seen these code testing styles, for years on end.

## Preamble

[Interfaces are overused, mocks are overused](https://www.anthonysteele.co.uk/InterfacesAreOverused) but [the test pyramid is still relevant](https://www.anthonysteele.co.uk/TestPyramid).

Most tests that we see in practice are too closely coupled to the code under test.

There is [demo code for this blog post](https://github.com/AnthonySteele/CoupledTestDemo), so that the techniques can be worked through. But bear in mind that this code does nothing, it is merely a reworking of the sample ASP "weather forecast controller". In itself, it is far to simple to need the testing done here. But it must be small to be a readable demo stand-in for a much larger app, that would have many more classes with multiple methods arranged in more layers.

The demo code has a Controller, that calls a service that calls a repository that presumably does the data retrieval. So far so familiar.

In a real app there would be multiple repositories that would be e.g. a concrete dependency on a database, along with other ways to get and send data to http services etc. We cannot _unit_ test this repository, so we must give it an interface, and then swap in a different implementation for tests, and so test the rest of the app without the real repository.

The same app is tested in multiple ways.

## Isolated style

Isolated, "Mock the interface" tests.

Sadly this is the default, by far the most common style. I have seen a co-worker get very good at this style. While I was saying "100% coverage isn't necessarily, important or even always possible", they turned in 100% coverage, even exception handling. And yet the app was still a terrible codebase to work on. Tech debt was the main impediment to process, but refactorings just didn't happen. A large body of tests in this style can be as much a tedious liability as it is a useful safety net.

## The Isolated tests

[In `WeatherForecastControllerIsolatedTests.cs`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsWithMocks/WeatherForecastControllerIsolatedTests.cs), Each class is tested in isolation with mocks. The test coverage is high. mocking code is verbose and repetitive.

A mock repository is injected into the service, and then separately, a mock service is injected into the controller. Both service and repository must have interfaces for this. We verify the the correct method was called.

The code smells that we typically see include tests with dozens of mocks, verbose and repetitive mock setup to test very little logic, and even for extra insanity, [key business logic in AutoMapper mappers](https://www.anthonysteele.co.uk/AgainstAutoMapper), which are called in the middle of the code, but are not part of the test, but mocked for test.

## Sociable style

This style is called ["sociable unit tests"](https://martinfowler.com/bliki/UnitTest.html) where an assembled subsystem is tested.

## The Sociable tests

[In `WeatherForecastControllerSociableTests.cs`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsWithMocks/WeatherForecastControllerSociableTests.cs) we test an assembled stack of real controller, real service and mock repository.

The mocking code is extracted for re-use, which makes it less verbose.

This is better in some ways. We focus a bit more on testing outcome not implementation. It's more about "do we return the expected value?" and less "did we call the expected method?".

If this approach is taken, then we can refactor the service at will. The service does not even need to have an interface at all, and this extra declaration should be deleted. We are at the point that the foolish dogma of "always put an interface on every class" is actually useless.

You should also be able to "extract class" as a refactoring in some cases without changing tests, and still verify that nothing is broken.

## Decoupled style

For the last, [decoupled style](https://github.com/AnthonySteele/CoupledTestDemo/tree/main/WeatherServiceTestsHost), it took a while for me to get used to it when used as the main engine of unit testing.

I started by thinking "this is a lot of indirection to accomplish the same outcome". And then a few months later I noticed that I had got into the habit of starting a feature task by first making a failing test. Without touching a single line app code I was able to add a test that "when _new thing_ happens, then _outcome_ should result. It felt like starting a building work by putting up a scaffold that would support the structure safely. The tricky part was done already when I had a failing test.

This was a breakthrough. "Test first" is good, or so [I had always heard](https://www.youtube.com/watch?v=fPlBLlE8vOI). Yet it wasn't prevalent in most places where I have worked. I didn't do it much. Why? Was this purely because I lacked the self-discipline or the innate skill? But instead it seems that a major factor was that _the test style before now did not encourage test-first coding_.

Consider this: if "refactoring" is changing code under test, then how well can you refactor if any change breaks tests?

With this approach, we have freedom to refactor the code under test fairly freely. Split 1 controller into 2? No problem! Do away with controllers entirely and use [minimal web API routes](https://learn.microsoft.com/en-us/aspnet/core/tutorials/min-web-api) instead? No problem, it's under test!

### But is it _unit_ tests?

My view is that these outside-in tests are unit tests. Simple as that. However, this is mostly a semantic distinction, you can get the same value from them if you think otherwise.

But, the idea that they must be "integration" because they test multiple classes is IMHO a complete misunderstanding of what is being "integrated".

What is a "unit" in "unit tests" anyway? Views on what's a "unit" that I have heard  that I have sympathy for are:

* It's called "unit", a neutral term, because it's your choice of what this unit is. If it was always "method" it would be called a "method test". It's not called that, therefore it's not always that.
* It tests a unit of app functionality, [a single logical concept in the system](https://www.artofunittesting.com/definition-of-a-unit-test).
* [Test behaviors, not implementation details](https://www.youtube.com/watch?v=EZ05e7EMOLM&t=1428s) - this blog post would be _incomplete_ without referencing "TDD, Where Did It All Go Wrong" by Ian Cooper.

Ultimately the only view that's wrong and harmful is that "a unit test always tests a method on a class, in isolation". Also false is that "integration test integrate different classes in the app".

Consider the definition of unit tests, where "A test is not a unit test if: It talks to the database; It communicates across the network; It touches the file system" (from ["A Set of Unit Testing Rules", Michael Feathers, 2005](https://www.artima.com/weblogs/viewpost.jsp?thread=126923) )   - these decoupled tests _meet all of those criteria_.

Consider the other definition of unit tests, where they are small, fast, cheap, numerous, reliable, can be run frequently, can be run concurrently: these decoupled tests meet all of those criteria too.

By reasonable and pragmatic definitions, these are unit tests.

Kent Beck:

> "Unit tests are completely isolated from each other, creating their test fixtures from scratch each time." the word unit in unit testing refers to the test itself: unit tests are isolated from other tests. Beck argues that "tests should be coupled to the behaviour of code and decoupled from the structure of code."

https://www.infoq.com/articles/unit-testing-approach/

## the demo app Decoupled

We use the [Test Host](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests). Contrary to what that page says, this host is not _just_ for "integration tests" that "include the app's supporting infrastructure, such as the database, file system". It depends on how you set it up.

 The demo app has [a `TestApplicationFactory`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsHost/TestApplicationFactory.cs) that  replaces repositories with mocks - there is only one that needs to be replaced in this simple app, but it can be as many as needed.

 The _only_ classes that _need_ interfaces in this design are the ones that need to be mocked for unit testing, because they have dependencies such as databases that can't be unit tested. Other interfaces can simply be deleted.

There's a bit of overhead to get it all set up, but this is a once-off cost so it matters less on larger apps. With a few helper classes it's fairly transparent. That is demonstrated with the `TestContext` This can be shared for even better performance in cases where this doesn't affect the test. I have never found this technique to be "too slow".

It's clear that it tests more of the application as well: if you mess up your http configuration so that the route is wrong, or forget to register a necessary service, you'll know it, whereas the other tests don't get to that before actual "deploy and run tests on the deployed app" integration tests.

We also demo [a `FakeWeatherForecastDataStore`](https://github.com/AnthonySteele/CoupledTestDemo/blob/main/WeatherServiceTestsHost/FakeWeatherForecastDataStore.cs) [a fake implementation](http://xunitpatterns.com/Fake%20Object.html), instead of a mock using a mocking tool. It tracks calls with a `CallCount` property. This technique is also underused. In many cases, this is simpler, clearer code than the equivalent using mocking framework.

What I typically find that this kind of Fake is simpler, but a bit more verbose - more lines of code, but simpler lines of code - than the mocking framework equivalent. But then the mocking equivalent gets repeated multiple times in the codebase, adding up to far more lines of code than declaring a "Fake data store" or "In-memory repository" once.

## End note

Use TestHost for more tests. If you find yourself unable to do a simple "extract class" refactoring because it would both break existing tests and the new class would require new tests, they something is wrong: The app code is too coupled to the test code.

Favour State-Based Testing over Interaction  testing: favour test that test _outcomes_ such as returning a result, enqueueing a message, or modifying a state in a data store correctly, over tests that test that the code calls the expected _method_.

> State-based testing is always preferred. The primary reason is that it is less coupled to your code. So you can more easily change your code without changing the test.

[Interaction vs State-Based Testing](https://thinkster.io/tutorials/blogs/interaction-vs-state-based-testing)

I don't advocate for these "decoupled" tests to be the _only_ kind of test, just the _default_ kind that you [should use](https://datatracker.ietf.org/doc/html/rfc2119). 

About 80% of the test coverage can be decoupled, depending on the specifics of the app. There will be business logic cases where you are better off dropping down to a class-level test and pumping many test cases into that subsystem. Even then, these might be "sociable tests" that cover multiple classes, as the current exact subdivision of the code into classes is _not the test's concern at all_ since you're testing the _behaviour_ of code and not coupled to the _structure_ of code.

I didn't come to this position out of theoretical reasoning: This was given to me by a team already using it. And it worked for me, even better than I thought it would. And it could work for you too.
