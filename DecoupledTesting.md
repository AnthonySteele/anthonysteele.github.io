# Decoupled Unit Tests Manifesto

Decoupled unit tests are a test type that I consider to be "unit tests". But if you don't, then you can still understand structure and benefits  of what I am describing. Naming is a semantic argument, but under any name I have found them to provide great value.

We find that this style leads to better outcomes. It also supports both Test-Driven Development and Refactoring so much better than the default, mocks and isolation style of unit tests.

## Rules and definitions

The rules and names used in this context are:

**Decoupled**: tests should couple to application behaviour, not to class methods. The test should be sensitive to the behaviour of the code under test, but ignore its structure. A test should in general fail when application behaviour changes, but not when the structure changes, e.g. a method is renamed. This means testing mostly "from the outside in".

**Unit**: We reject entirely the idea that the "unit" in "unit test" is a method or a class. This is not our definition of a unit test. Although these class-method tests are unit tests, they are not the only kind of unit tests. They are not even the primary, most common kind of unit tests. While these tests are useful in some cases, they should not be our first choice, and should form only a small portion of the tests, deployed in cases where they clearly make the most sense. You might see these low-level tests form maybe 10% of the test cases.

Each test is instead about an element of app behaviour. Individual tests test pieces of application functionality. The scope of a "unit" is intentionally loosely defined, and can be used flexibly. It should be decoupled from the structure of the code. While often the behaviour under test actually is located in one method of one class, that fact about the structure of the code is not the concern of the test.

Instead, a test is a unit test if it is "I/O free": if it follows "A set of Unit testing rules" (Michael Feathers, 2005) and can be made fast and deterministic with no external dependencies or special rules that prevent it being run quickly, reliability and on a number of machines.

**Integration**: We specifically reject the use of "integration test" to describe tests that test multiple application classes at once. That is an irrelevant, actively unhelpful definition here. It doesn't lead to good tests. This definition makes sense only in as much as it follows from a false premise: if only single-class tests count as "unit tests", then a name is needed for tests that cover 2 or more classes.

And so long as the tests are I/0-free, and can be made small, fast and deterministic, these are just unit tests to us, they are not "integration" or any other different kind of test. This follows from tests not being coupled to application code structure.

We express  tests in the language of the business domain, not of the class structure. We can even stand up most of the applications for unit tests. This is how we decouple.

We use the word "integration" to refer to external dependencies, only. Historically, this is accurate: Mr Feathers lists things that unit tests do not use: external databases, external http services, etc, and uses the word "integration" once, with regard to "the integration of your code with that other software" as a thing that unit tests do not do.As a corollary, there are also integration tests do use these integrations.

**Mocks**: We use test doubles (mocks, fakes, etc) sparingly. Typically they are used only when needed, to stub out I/O, to avoid using these external service integrations. Not actually calling that database or HTTP web service is when mocking is necessary in order to have unit tests.  We avoid mocks when it is not necessary.

We use mocking frameworks even more sparingly. It is often better to simply equip the class that has a concrete dependency with an interface, and in test supply an in-memory fake implementation of the interface, that can also be coded to respond and record as needed. This is often simpler and easier to do this once, than to scatter loads of mock setup and verification code throughout each test. Using a mocking library, then adding complex mocking code everywhere is a false economy.

But behind the scenes, "interception" frameworks such as Wiremock may be useful, as they maximise the code that is under test, before the test implementation has to take over.

We don't pay a lot of attention to the categories of test doubles. The fine grained-distinction between a spy and a stub is not generally relevant, and the same test double can be used for both purposes, interchangeably.

**Setup**: There may be more test setup than before. Especially if you use e.g. Messages queues. But other parts of the application, e.g. application startup is also largely tested.

**Interfaces**: we add interfaces to classes when it is needed. Often it is not, and we do not add it. The DI container does not need all classes to have interfaces, classes with no interface can easily be registered. When an interface that adds no value is encountered, we delete it, like any other kind of dead code.

## Other kinds of tests

Other kinds of tests are also OK. The general concept of a "testing pyramid" has merit. We recognise that there is no one single test style that will do everything. I have mentioned lower-level tests and integration tests. Tests on an application that has been deployed to a development environment are also useful. As is production monitoring. However, these decoupled unit tests can form the backbone, the single largest and most thorough part of the testing part of the process.

In the ASP .NET world we use the test host extensible to allow this test style. Thus tests also test a lot of the application's startup code, and will tell us if e.g. a new service is missing from the Dependency Injection Container.

Refactoring: We have found this test style to be better for refactoring.  Refactoring is "changing code, while tests still pass". It is better to be free to refactor, even extract classes, or merge classes, and immediately afterwards get green tests, rather than tests that don't even compile as they are so tightly bound to public methods on classes. The test should fail when the application is doing something different, not when the code is moved around.

Coupled tests are fragile. Coupled tests are verbose and hard to read. Coupled tests impede refactoring but don't give good assurances that the app as a whole works well. They promote a "test after implementation" workflow, which cannot be TDD. The one thing that they excel at is test coverage.

We have also found that decoupled tests much better support Test-Driven Development. Working from the outside in, it is much easier to code up e.g. a new field on a request and the expected outcome, before even touching the code under test.

## The twist

These are my definitions for "decoupled unit tests".

If your definitions are different, then ok, you're doing something else. We can agree that our practices and vocabularies don't match, rather than one being "wrong". Both are consistent in their respective spheres.

But, this decoupled kind of test is a useful and productive practice, and it can be usefully done even if you don't use these names for it.

I said that if you don't want to call these tests "unit tests", that's fine. And I won't dispute it. I didn't quite tell the truth.

These tests might be closer than most to what is described in the literature as "unit tests", and that the stated benefits of "unit testing" might be found here more than with other practices that are coupled and full of mocks. This is what was meant by unit tests.

Mainly, these tests are in my experience better suited to enabling both TDD and fearless refactoring.
