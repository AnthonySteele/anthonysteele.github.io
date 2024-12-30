# Decoupled Unit Tests Style

Decoupled unit tests are a test type that I consider to be good "unit tests".

But you can still understand structure and benefits  of what I am describing even if you don't call them "unit tests". Naming is a semantic argument, but under any name I have found them to provide great value.

They are very [sociable unit tests](https://martinfowler.com/bliki/UnitTest.html). We find that this decoupled, outside-in style leads to better outcomes. It also supports both Test-Driven Development and Refactoring so much better than the default, mocks and isolation style of unit tests.

## Rules and definitions

The rules and names used in this context are:

### Decoupled

[A test should be sensitive to the behaviour of the code under test, but not to its structure.](https://www.youtube.com/watch?v=C5IH0ABmyc0&t=2108s) A test should in general fail when application behaviour changes, but not when the structure changes, e.g. a method is renamed. This means testing mostly "from the outside in".

### Unit as method

We reject entirely the idea that a unit test always tests a single method or a class. This is not our definition of a unit test. Although these class-method tests are unit tests, they are not the only kind of unit tests. They are not even the primary, most common kind of unit tests. While these tests are useful in some cases, they should not be our first choice, and should form only a small portion of the tests, deployed in cases where they clearly make the most sense. You might see these low-level tests form maybe 10% of the test cases.

### The meaning of "Unit"

The word "Unit" is the source of much confusion, as there are two distinct ideas here that get conflated: _what_ is under test, and _how_ it is tested.

**How** is the primary meaning: [The word "unit" refers to the isolated way how things are tested, more than it does what is tested](https://www.infoq.com/articles/unit-testing-approach/). It tests in a unitary way, rather than testing "units".

A unit test can be run on its own, or run in parallel with any or all other unit tests. They should  not affect each other, as they are units independent of each other. It is not using any external service that might fail, it is an unit independent of that.

A unit test is "I/O free", it follows ["A set of Unit testing rules" (Michael Feathers, 2005)](https://www.artima.com/weblogs/viewpost.jsp?thread=126923). It can be made fast and deterministic, with no external dependencies or special rules that prevent it being run quickly, reliability and on a number of machines as an independent unit.

**What**:  The scope of what a unit test tests is intentionally loosely defined, and can be used flexibly.

Individual tests test pieces of application functionality: [we test behaviours not implementation details](https://www.youtube.com/watch?v=EZ05e7EMOLM&t=1428s). While often the behaviour under test actually is located in one method or one class, that fact about the structure of the code is not the concern of the test. It can change without affecting the test.

### Integration tests

We specifically reject the use of "integration test" to describe tests that test multiple application classes at once. That is an irrelevant, actively unhelpful definition here. It doesn't lead to good tests. This definition makes sense only in as much as it follows from a false premise: if only single-class tests count as "unit tests", then a name is needed for tests that cover two or more classes.

And so long as the tests are I/0-free, and can be made small, fast and deterministic, these are not "integration" or any other different kind of test. They are just unit tests to us. This follows from tests not being coupled to application code structure.

We use the word "integration" to refer to external dependencies, only. Historically, this is accurate: Mr Feathers lists things that unit tests do not use: external databases, external HTTP services, etc, and uses the word "integration" once only, with regard to them. He refers to "the **integration** of your code **with that other software**" as a thing that unit tests do not do. As a corollary, if only I/0-free tests count as "unit tests", then tests that have I/O are something else. They test "the integration with other software", i.e. are "integration tests".

### Mocks and other test doubles

We use test doubles (mocks, fakes, etc) sparingly. Typically they are used only when needed, to stub out I/O, to avoid using these external service integrations. This is when mocking is _necessary_ in order to have unit tests.  We avoid using mocks when it is not necessary.

We use mocking frameworks even more sparingly. It is often better to simply equip the class that has a concrete dependency with an interface, and in tests [use an in-memory fake implementation of the interface](https://dunnhq.com/posts/2024/prefer-test-doubles-over-mocking/), that can also be coded to respond and record as needed. This is often simpler and easier to create this class  once, than to scatter loads of mock setup and verification code throughout each test. Using a mocking library, then adding complex mocking code everywhere is a false economy.

But behind the scenes, "interception" frameworks such as [WireMock library](https://github.com/WireMock-Net/WireMock.Net) may be useful, as they maximise the code that is under test, before the test implementation has to take over.

We don't pay a lot of attention to the categories of test doubles. The fine grained-distinction between a "spy" and a "mock" is not generally relevant, and the same test double can be used for both purposes, interchangeably.

### Interfaces

Code style in C# is often to add interfaces to all kinds of classes as some kind of rote action. This is often pointless.

 we add interfaces to classes when it is needed. Often it is not, and so we do not do it. The Dependency Injection container does not need all classes to have interfaces. classes with no interface can easily be registered for DI. When an interface that adds no value is encountered, we delete it, like any other kind of dead code.

## Test Setup

We express tests in the language of the business domain, not of the class structure. We can even stand up most of the applications for unit tests. This is how we cover e.g. application startup, Dependency registration, controller entry points etc. with the same tests as the rest of the application. We mostly test from the outside-in. This is how we decouple.

There may be more test setup than before. Especially if you use e.g. Messages queues. However this setup is not verbosely repeated in each test.

In the ASP .NET world we use the [test server](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests) to allow this test style. We customise the startup where needed by adding using fakes for external dependencies, but otherwise leave it as is. Thus tests also test a lot of the application's startup code, and will tell us if e.g. a new service is missing from the Dependency Injection Container.

## Bad tests

Tests can be bad in a number of ways:

Bad tests can be hard to read. Bad tests can fail to explain the application functionality.

Bad tests can be flaky, and fail sometimes for mysterious reasons such as timing. Bad tests can always pass, even when important parts of the code are removed. Bad tests can fail to cover all the important functionalities of the application.

The under-recognised way in which tests are bad, is having bad tests that that break hard when the code under test is changed slightly. They are tightly coupled to implementation details.

## Other kinds of tests

Other kinds of tests are also OK. The general concept of a "testing pyramid" has merit. We recognise that there is no one single test style that will do everything. I have mentioned lower-level tests and integration tests. Tests on an application that has been deployed to a development environment are also useful. As is production monitoring. However, these decoupled unit tests can form the backbone, the single largest and most thorough part of the testing part of the process.

## Refactoring

We have found this test style to be better for refactoring.  Refactoring is "changing code, while tests still pass". But how can refactoring happen at all, if your tests are so coupled that changing anything breaks tests - and not due to changes of functionality, but merely due to e.g. extracting a class, renaming a method, adding or removing a method parameter? If _refactoring_ is changing code without breaking tests, then how can you even refactor when any change to a public method or constructor causes tests to no longer even compile?

It is better to be free to refactor, even extract classes, or merge classes, and immediately afterwards get green tests, rather than tests that don't even compile as they are so tightly bound to public methods on classes. The test should fail when the application is doing something different, not when the code is moved around.

Coupled tests are fragile. Coupled tests are verbose and hard to read. Coupled tests impede refactoring but don't give good assurances that the app as a whole works well. They promote a "test after implementation" workflow, which cannot be TDD. The one thing that they excel at is test coverage. And past a certain point, this is not an important metric.

We have also found that decoupled tests much better support Test-Driven Development. Working from the outside in, it is much easier to code up e.g. a new field on a request and the expected outcome, before even touching the code under test.

## The original meaning

These are my definitions for "decoupled unit tests".

If your definitions are different, then ok, you're doing something else. We can agree that our practices and vocabularies don't match, rather than one being "wrong". Both are consistent in their respective spheres.

But, this decoupled kind of test is a useful and productive practice, and it can be usefully done even if you don't use these names for it.

I said that if you don't want to call these tests "unit tests", that's would be fine. That I would not dispute it. Anyway, I didn't quite tell the truth.

These tests are closer than most to what is described in the literature as "unit tests". And the stated benefits of "unit testing" might be found here more than with other practices that are coupled and full of mocks. This style is what was meant by unit tests.

And these tests, in my experience, better deliver the stated benefits of unit tests: both TDD and fearless refactoring. And are what was originally meant.

## Related

* [Decouple your Unit Tests](https://www.anthonysteele.co.uk/CoupledTesting)
* [Interfaces are Overused](https://www.anthonysteele.co.uk/InterfacesAreOverused)
* [The Test Pyramid](https://www.anthonysteele.co.uk/TestPyramid)
