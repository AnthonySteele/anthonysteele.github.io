# The Test Pyramid

_An intermediate-level refresher of my take on the whats and whys of the "Test Pyramid" concept._

I still find a lot of value in [the "Test Pyramid" concept](https://martinfowler.com/bliki/TestPyramid.html).
 Martin Fowler's post is good and short read, and most of what I am saying here is either stated or implied there.

The lowest layer of tests is usually called "unit tests" but past that, the naming is not standard. They get called integration tests, acceptance tests, end-to-end-tests etc. The different layers can have wildly varying names, or different definitions of the same terms, and you can spend a lot of time trying to define terms. And even [how to do unit tests is debatable, not necessarily just class tests](https://www.youtube.com/watch?v=EZ05e7EMOLM).

Your team and organisation will need to have a common vocabulary of naming test layers, but bear in mind that these are not universal. I think that it's more important to have a feel about where tests are placed, based on the metrics below; and about how these layers relate.

## Pyramid as architecture

There are basic characteristics that can help you place tests on the pyramid.

Tests at the bottom are "Low-Level": **small, decoupled, numerous, fast, frequent, cheap**.

As you move up, this changes.

Test at the top are "High-level": **large, coupled, few, slow, infrequent, expensive**.

Unit tests are:

* **Small**: They test a part of the system.
* **Decoupled**: They have few, if any, external dependencies such as http services, database, file systems, etc. There may be use of mocks.
* **Numerous**: There are a lot of them.
* **Fast**: Test execution time is low. You can still run the whole test suite quickly, despite the large numbers of tests.
* **Cheap**: It is not hard to create, add to or delete tests.
* **Frequent**: As a consequence, they are run often, for fast feedback in a development cycle.

Tests at the top are:

* **Large**: They test the whole system.
* **Coupled**: Test multiple parts from UI through to services and data stores. Often a "user journey".
* **Slow**: Take a while to run, are run less often.
* **Expensive**: May require complex frameworks or tools, configuration and setup, and need upkeep.
* **Few**: As a consequence, there are not many of them, Or at least there _should not be_ many of them if you want the testing to work well.

Tests in the middle of the pyramid are intermediate on these metrics.

## Consequences of the pyramid

### Layers as support

The basic architectural principle of ancient stone pyramids is that each layer of stonework supports the layer above it. Tests are not all that different. Each layer depends on the layer below: that is, it is run afterwards, and assumes that the previous layer passes and provides guarantees about the software. There is no need for e.g. integration tests to verify details that unit tests have already proved, only to prove that these correct parts can be combined.

For an example, an integration test that feeds in multiple incorrect values into a user interface and clicks "submit" in order to test each validation case is misunderstanding this point. The many cases of validation pass and fail can be exhaustively tested in unit tests, and then the integration test just needs one case to confirm that this working validation component is in fact wired up to the UI.

Sometimes you find a bug, and characterise it with a failing end-to-end test. But then it's best to "Push down" the details into smaller and faster tests. This keeps the bulk of tests at the bottom and avoids drifting into the "[Ice-cream cone of shame](https://medium.com/@fistsOfReason/testing-is-good-pyramids-are-bad-ice-cream-cones-are-the-worst-ad94b9b2f05f)"

## How many layers

YMMV, but approximately, more than 2, but less than 8. Play it by ear.

New techniques do come along allowing new testing seams. e.g. in ASP.NET, you can now launch the web application in a light-weight in-memory host, and optionally supply mock data stores to it, send http requests to it, all of this in-process. This is above a conventional unit test in scope, but below a test that deploys the web application to a test server and then makes queries to it across a network.

Would these replace unit tests and post-deploy integration tests? Only partly. I would advocate for a mix of all of these in the test structure.

## Test-like techniques

There are also things that we don't strictly consider to be tests, but play similar roles to tests. You can consider them "test-adjacent" techniques:

At the bottom, fastest and most frequent is in the development environment:
If you have a language with [static type checking](https://en.wikipedia.org/wiki/Type_system#Static_type_checking), the compiler rejects some things as invalid upfront. With types, you don't need a test to tell you that you passed a customer to an order method - the compiler will block that. When the language is not compiled, a linter such as [ESLint for JavaScript](https://eslint.org/) plays a similar role. And in compiled languages, linters are also useful for producing high-quality code. e.g. [Roslyn analysers](https://docs.microsoft.com/en-us/visualstudio/extensibility/getting-started-with-roslyn-analyzers) and [SonarQube](https://www.sonarqube.org/).

At the other end are post-deploy things sometime called "[testing in production](https://medium.com/@copyconstruct/testing-in-production-the-safe-way-18ca102d0ef1)".

logging and metrics on error rates and execution times will warn you of issues soon after they happen. These techniques are very valuable, as an addition to tests. And we can amplify this with canary release or mirroring live traffic onto a test environment.

You may be working with a well-tested system, and thinking that "we test so that we never deploy bad code anywhere" or "we have a full-featured staging environment so that we catch all problem before they get to production".
These are worthy goals, but perfection is unattainable and each technique reaches the point of diminishing returns.
If making your staging environment ever more production-like is becoming more and more expensive and delivering less benefit, take a step back and consider if spending the effort on e.g. enabling canary releases to production instead would be more effective.

On the other hand, you might be working with an untested system, in which case try merely having a run of unit tests on CI server before merging changes, and after merging checking that the system can be deployed to a test server and will respond `200 OK` to simple get request. This will be immensely valuable - you now have a simple 2-level test pyramid that can be expanded upon.

## Pitfalls

## Mocks

Over-reliance on mocks in unit tests is a smell, it leads to tests that are not readable or maintainable.
Mr Cooper's solution: test a subsystem not a class, refactor the code to be easier to work with [by pushing Input and Output to the edges](https://www.goparamore.io/ports-adapters); consider eliminating interfaces that are there purely for DI. If you can mock the data stores and other external dependencies, you might have all the interfaces that you need.

If you have to touch the tests a lot when doing refactoring, then you probably aren't doing it well.

## A Unit test tests a class

This is not always true. A unit test tests one "thing", be it a class, a function or a subsystem composed of one or more closely related classes.  or [whatever code meets the requirement](https://www.youtube.com/watch?v=EZ05e7EMOLM&t=1490s) Much like the "Responsibility" in SRP, there is no precise definition of the thing: it's an design guideline not a scientific measure. It is useful, even though we can't always remove ambiguity over where the boundary should be drawn.

Consider this: If I want to refactor to extract a class, [should I hesitate because of the overhead of changing tests that will ensue](https://www.youtube.com/watch?v=EZ05e7EMOLM&t=600s)? Shouldn't my tests still be useful and relevant without modification, after the class is extracted in the code under test? If they're too closely coupled, I cannot do that many more.

## Abstraction

Use of high-concept testing libraries is a thing that fails time and again.
These either obscure the above problems with mocks behind a façade, or it purports to "read like English, so the business people will get involved" - it doesn't and they won't. And the library inevitably comes with dependencies, learning curves, Impedance mismatch and other frictions.

## Don't Repeat Yourself

DRY should of course be used in tests, but not overused. "[Beware the share](https://github.com/97-things/97-things-every-programmer-should-know/blob/4e03ea6022379dc32f4b0cce82b64c323d7f23c1/en/thing_07/README.md)" of coupling unrelated and distant things in order to "reduce duplication" - it is correctly observed that coupling damages more software than repetition does. Being self-contained counts for a lot in unit tests. Look for dependencies on shared internal NuGet packages, or tests that can't be run at the same time as other test, especially near the bottom of the pyramid.

There are test-specific patterns to reduce duplication in tests. e.g. if the data setup is verbose and repetitive, look at a [Test Data Builder](https://wiki.c2.com/?TestDataBuilder) class.
If a group of asserts is repeated or verbose, extract a method named e.g. `AssertValidOrderWasDelivered`. If this is used in multiple source files, extract a class for it.
Avoid making a "base test for tests". It becomes unwieldy as unrelated concerns pile up there, so it is more flexible to extract classes, i.e.: [favour composition over inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance).

## Testing configuration

Should you test config files?

Yes. A rule of thumb: If it's in a file, and if it being incorrect can cause your system to not work properly, the try to cover it with a test. It doesn't matter if the file types are `.cs`, `.js` for code, or `.json` and `.yml` for config.

And "infrastructure as configuration and code" is good, so lots of things will rightly be in files. Tests are part of the chain of automation over these files.
