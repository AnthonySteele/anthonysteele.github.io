# The Test pyramid

I still find a lot of value in [the "test pyramid" concept](https://martinfowler.com/bliki/TestPyramid.html).
 Martin Fowler's post is good and short read, and most of what I am saying here is either stated or implied there.

The lowest layer of tests is usually called "unit tests" but past that, the naming is not standard. They get called integration tests, acceptance tests, end-to-end-tests etc. The different layers can have wildly varying names, or different definitions of the same terms, and you can spend a lot of time trying to define terms. And even [how to do unit tests is debatable, not necessarily just class tests](https://www.youtube.com/watch?v=EZ05e7EMOLM).

## Pyramid as architecture

But there are basic characteristics that can help you place tests on the pyramid.

Tests at the bottom are "Low-Level": **small, isolated, numerous, fast, frequent, cheap**.

As you move up, this changes.

Test at the top are "High-level": **large, coupled, few, slow, infrequent, expensive**.

Unit tests are:

* Small: test a part of the system.
* Isolated: have few, if any, external dependencies such as http services, database, file systems, etc. There may be use of mocks.
* Numerous: there are a lot of them.
* Fast: Test execution time is low. You can still run the whole test suite quickly.
* Frequent: They are run a lot, for fast feedback in a development cycle.
* Cheap: they're not hard to create, add to or delete.

Tests at the top are:

* Large: They test the whole system.
* Coupled: Test multiple parts from UI through to services and data stores. often a "user journey".
* Few: not many of them.
* Slow: Take a while to run, are run less often.
* Expensive: May require complex frameworks or tools, configuration and setup, and need upkeep.

Tests in the middle of the pyramid are intermediate on these metrics.

## Consequences of the pyramid

### Layers as support

The basic architectural principle of ancient stone pyramids is that each layer of stonework supports the layer above it. Tests are not all that different. Each layer depends on the layer below: that is, it is run afterwards, and assumes that the previous layer passes and provides guarantees about the software. There is no need for e.g. integration tests to verify details that unit tests have already proved, only to prove that these correct parts can be combined.

For an example, an integration test that feeds in multiple incorrect values into a user interface and clicks "submit" in order to test each validation case is misunderstanding this point. The many cases of validation pass and fail can be exhaustively tested in unit tests, and then the integration test just needs one case to confirm that this working validation component is in fact wired up to the UI.

Sometimes you find a bug, and characterise it with a failing end-to-end test. But then it's best to "Push down" the details into smaller and faster tests. This keeps the bulk of tests at the bottom and avoids drifting into the "[Ice-cream cone of shame](https://medium.com/@fistsOfReason/testing-is-good-pyramids-are-bad-ice-cream-cones-are-the-worst-ad94b9b2f05f)"

## How many layers

YMMV, but approximately, more than 2, but less than 8. Play it by ear.

New techniques do come along allowing new testing seams. e.g. in ASP.NET, you can now launch the web app in a light-weight in-memory host, and optionally supply mock data stores to it, send http requests to it, all of this in-process. This is above a conventional unit test in scope, but below a test that deploys the web application to a test server and then makes queries to it across a network.

Would these replace unit tests and post-deploy integration tests? Only partly. I would advocate for a mix of all of these in the test structure.

## Test-like techniques

There are also things that we don't consider to be tests, but play similar roles to tests. You can consider them "test-adjacent" techniques:

At the bottom, fastest and most frequent is in the development environment:
If you have a language with strong typing, the compiler rejects some things as invalid upfront. With types, you don't need a test to tell you that you passed a customer to an order method - the compiler will block that. A linter plays a similar role in e.g. JavaScript.

At the other end are post-deploy things sometime called "[testing in production](https://medium.com/@copyconstruct/testing-in-production-the-safe-way-18ca102d0ef1)".

logging and metrics on error rates and execution times will warn you of issues soon after they happen. These techniques are very valuable, as an addition to tests. And we can amplify this with canary release or mirroring live traffic onto a test environment.

You may be working with a well-tested system, and thinking that "we test so that we never deploy bad code anywhere" or "we have a full-featured staging environment so that we catch all problem before they get to production". These are worthy goals, but consider that each technique reaches the point of diminishing returns: if making your staging environment ever more production-like is becoming more and more expensive and delivering less benefit, consider if canary releases to production would be more effective for you.

On the other hand, you might be working with an untested system, in which case merely having a unit test run on CI server before merging, and after merging checking that the system can be deployed to a test server and will respond `200 OK` to simple get request will be immensely valuable - you quickly have a simple 2-level test pyramid.

## Pitfalls

## Mocks

Over-reliance on mocks in unit tests is a smell, it leads to tests that are not readable or maintainable.
Mr Cooper's solution: test a subsystem not a class, refactor the code to be easier to work with, consider fewer interfaces that are just there for DI. If you can mock the data stores and other external dependencies, you might have all that you need.

If you have to touch the tests a lot when doing refactoring, then you probably aren't doing it well.

## Abstraction

Use of high-concept testing libs is a thing that fails time and again.
These either obscure the above problems with mocks behind a façade, or it purports to "read like English, so the business people will get involved" - it doesn't and they wont. Or both. And the library inevitably comes with dependencies, learning curves, Impedance  mismatch and other frictions.

## DRY

DRY should of course be used in tests, but not overused. "Beware the share" of coupling unrelated and distant things in order to "reduce duplication" - it is correctly observed that coupling damages more software than repetition does. Being self-contained counts for a lot in unit tests. Look for dependencies on shared internal NuGet packages, or tests that can't be run at the same time as other test, especially near the bottom of the pyramid.

## Testing configuration

Should you test config files?

Rule of thumb: If it's in a file, and if it being incorrect can cause your app to not work properly, the try to cover it with a test. it doesn't matter if the file type is `.cs`, `.js` or `.json` and `.yml`.

And "infrastructure as configuration and code" is good, so lots of things will rightly be in files. Tests are part of the chain of automation.

