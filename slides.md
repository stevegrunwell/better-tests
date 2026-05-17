<!-- .slide: class="title-slide" data-hide-footer -->
# Getting the Most From Your Test Suite

Steve Grunwell <!-- .element: class="byline" -->
[@stevegrunwell@phpc.social](https://phpc.social/@stevegrunwell)
[stevegrunwell.com/slides/better-tests](https://stevegrunwell.com/slides/better-tests)

---

## Fundamentals of<br> automated testing

Note:

A quick disclaimer: this is not necessarily an "intro to testing talk", so we're not going to get too into the weeds setting up a test runner or anything. That being said, even if you're just familiar with the basic _concepts_ of testing, there should be something useful in this session.

----

### Why do we test?

* <!-- .element: class="fragment" --> Reduce toil of manual testing
    - Reproducible, trivial to run
* <!-- .element: class="fragment" --> Functions as advertised
* <!-- .element: class="fragment" --> Better understand the code we're writing, problems we're trying to solve
* <!-- .element: class="fragment" --> Simplifies refactoring

Note:

To kick things off, let's do a quick refresher on some of the big reasons _why_ we automate our testing:

* Manual testing is toilsome and rife with opportunities for human error
    - Automated testing makes it trivial to run, and ensure that it runs the same steps each time
    - Running tests as part of CI helps ensure that new code isn't breaking existing functionality
* We want to make sure things do what we expect them to, and continue to do so as things change
    - This includes things like regressions, AI slop, etc.
    - Well-written tests help to document intended behavior
* It can help us design our code better
    - If it's hard to test, that's a bad sign
    - The entire point of TDD!
* Having a strong test suite also makes it much easier to refactor code, as a failing test would indicate that something has significantly changed

----

### Test Automation Pyramid

![A hand-drawn representation of the Test Automation period, with three layers (from top to bottom): E2E, Integration, and Unit. The X-axis is labelled "# of Tests" while the Y-axis is labelled "Cost $".](resources/testing-pyramid.svg)<!-- .element: style="max-height: 45vh; margin: 0; padding: 0;" -->

Note:

If you've worked with automated testing before, you've probably seen this graphic: the "Test Automation Pyramid" (or simply "Test Pyramid").

The general concept is that we have a strong foundation of unit tests: these should be easy to write and quick to run. Generally we'll use PHPUnit for these.

The next level up is integration or feature tests: they might be more involved to write and take a bit longer to run, but they verify that the individual components integrate as we expect them to. This is also often in PHPUnit, but you might opt to use something like Behat instead.

The top layer is end-to-end or UI tests: these might involve scripting a browser with a tool like Playwright, and they will likely be the most time-consuming to write and execute.

It's important to remember that this is a guideline! Write the tests that are most meaningful for your codebase!

E.g. a library without a UI probably doesn't need any E2E tests, and even the integration tests might be sparse

Sometimes when adding tests to existing code, it can be helpful to start at the top

----

### Testing beyond the pyramid

* <!-- .element: class="fragment" --> Linting (e.g. PHP_CodeSniffer, ESLint)
* <!-- .element: class="fragment" --> Static code analysis (e.g. PHPStan, Psalm)
* <!-- .element: class="fragment" --> Security scanning, infrastructure testing, etc.

Note:

"Automated testing" covers so much more beyond that pyramid, though:

* Linting (checking coding standards)
* Static code analysis (find bugs, logical errors, etc. via heuristics run against code)
* Anything else you might throw into a CI pipeline

For this talk, we're going to focus on our unit and integration test suites. However, be aware that a well-rounded CI pipeline will likely include more than just unit/integration tests, and these tools all have their individual roles to play.

---

## Not all tests are made equal

Note:

The difference between a good test and a bad test is the difference between a relaxing weekend and being paged at 2am.

Just because tests _exist_, that doesn't mean that they're good or useful. Let's look at some common issues with tests:

----

### Tests that fail to capture the SUT

<u>S</u>ystem <u>U</u>nder <u>T</u>est

Note:

When we're writing tests, it's important to remember what piece of code we're intending to test: we call this the "System Under Test", or "SUT".

* This should be obvious for unit tests, generally a single method at a time
* Feature tests should only focus on the feature being tested
    - e.g. Authentication failures should not cause Checkout tests to fail
    - We should be using test doubles to mock unrelated parts of the application
* Isolating the SUT reduces complexity, improves maintainability and effectiveness of tests
    - Tests should test *one* thing!

----

### Tests that focus on implementation, not behavior

Note:

When people are first learning how to write tests, it's not uncommon to see tests that are effectively repeating what the code itself is doing: "we expect that calling this method will also call these three methods, each of which will return these values, etc."

This can result in brittle tests that break as soon as anything in the implementation changes, because they're being prescriptive.

Instead, think of tests as inputs and outputs: given this input, I expect to see this output. Anything else is incidental, unless you're explicitly setting out to test side-effects (e.g. ensure some method is called to test that a lifecycle hook is executed)

----

### Tests that fail for the wrong reasons

```php [|6]
public function testSaveThrowsOnInvalidEmail(): void
{
    $user = new User();
    $user->email = 'not an email address';

    $this->expectException(\Throwable::class);
    $user->save();
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

I've lost track of how many times I've come across tests like this: we want to verify that we can't persist a user with some invalid data, so we set some invalid data and tell PHPUnit that expect to see an instance of `Throwable` thrown.

However, we're looking for _any_ `Throwable` here. Maybe it's a `ValidationException` thrown because of the invalid email, but it could also be an error because some other required field isn't set. Or maybe a type error, or a missing argument somewhere. This test wouldn't catch those instances, because we're being too broad in our expectations.

----

### Non-deterministic tests

Running the same test multiple times should produce the same result!

Note:

This one seems obvious, but you might be surprised how often people mess this up.

* If tests rely on another test having been run before it in order to pass, that's an unwanted dependency.
* If tests start failing because the clock rolled over, that's an unwanted dependency.
    - Daylight Saving Time is great at catching these
* If tests break because some third-party service is down, that's an unwanted dependency.

----

### Tests that test nothing at all

```php
public function getID(): int
{
    return $this->id;
}
```

<pre class="code-wrapper fragment-replacement fragment hide-line-numbers" data-fragment-index="0"><code class="hljs php language-php fragment fade-out" data-fragment-index="1">public function testGetID(): void
{
    $this->user->setID(123);

    $this->assertIsInt($this->user->getID());
}
</code><span class="fragment fade-out" data-fragment-index="2"><code class="hljs php language-php fragment fade-in" data-fragment-index="1" data-line-numbers="5">public function testGetID(): void
{
    $this->user->setID(123);

    $this->assertIsInt($this->user->getID());
}
</code></span><code class="hljs php language-php fragment fade-in" data-fragment-index="2" data-line-numbers="5">public function testGetID(): void
{
    $this->user->setID(123);

    $this->assertSame(123, $this->user->getID());
}
</code></pre>

Note:

Imagine we have a simple `getID()` method on a model, which returns an integer.

You might find a corresponding test that looks something like this, but what's wrong with it?

This test doesn't tell us anything that the PHP type system doesn't already enforce. The return type-hint guarantees that `getID()` can *only* ever return an integer, so this test will always pass.

A better test for this method would be asserting that the value of `$this->user->getID()` is identical to a known value like `123`.

Speaking of assertions...

---

## Using the right assertion(s)

* <!-- .element: class="fragment" --> PHPUnit has dozens of assertions available
* <!-- .element: class="fragment" --> If you need something more-specific, write your own!
* <!-- .element: class="fragment" --> Everything boils down to true or false

Note:

Assertions are a crucial part of automated tests, but far too many people are content to just `assertEquals()` and call it a day.

* PHPUnit has dozens of assertions, and each assertion has both an affirmative and negative variant. Not everything needs to just be `assertEquals()`
* You can also write your own, custom assertions: this makes it easy to encapsulate business logic and reuse these assertions across your test suite
* Remember that at a fundamental level, every assertion boils down to true or false: does this string match what we're expecting? Do we see the array key we expect to see? Is this object of the right type?

Remember that using the right assertion can not only improve the quality of your tests, but also make debugging failures easier.

----

### Equal, but not the same!

```php [|3|4]
public function testEquality(): void
{
    $this->assertEquals(123, '123'); // 123 == '123'
    $this->assertSame(123, '123');   // 123 === '123'
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

One of the most frequent issues I see when reviewing tests is using `assertEquals()` instead of `assertSame()`:

* `assertEquals()` uses loose equality checks (e.g. "double equals")
* `assertSame()` uses strict equality ("triple equals")

Great way for subtle bugs to sneak through, since `0`, `false`, an empty string, null, and an empty array are all equivalent when compared with loose equality checks.

Rule of thumb: default to `assertSame()` anywhere you can't find a more-specific exception, then only resort to `assertEquals()` when absolutely necessary (e.g. two separate instances of an object that are otherwise equals)

----

### `assertCount()`

<pre class="code-wrapper fragment-replacement"><code class="hljs php language-php fragment fade-out" data-fragment-index="1">public function testCount(): void
    {
        $items = ['a', 'b', 'c'];

        $this->assertSame(2, count($items));
    }</code><code class="hljs php language-php fragment fade-in" data-fragment-index="1">public function testCount(): void
    {
        $items = ['a', 'b', 'c'];

        $this->assertCount(2, $items);
    }</code></pre>

<pre class="code-wrapper fragment-replacement fragment overflow-hidden" data-fragment-index="0"><code class="hljs language-text fragment fade-out" data-fragment-index="1">1) Tests\AssertionTest::testCount
Failed asserting that 3 is identical to 2.</code><code class="hljs language-text fragment fade-in" data-fragment-index="1">1) Tests\AssertionTest::testCount
Failed asserting that actual size 3 matches expected size 2.</code></pre>

Note:

A classic example of picking the right tool for the job: `assertCount()`.

When the assertion fails, `assertSame()` will tell us that we "failed asserting that 3 is identical to 2."

Meanwhile, `assertCount()` gives us more details: "Failed asserting that the actual size matches expected size 2."

It's a subtle difference, but makes it clear at a glance that we're comparing the size of two things.

----

### JSON assertions

```php[|5,9]
public function testJsonAssertions(): void
{
    $expected = json_encode([
        'key1' => 'val1',
        'key2' => 'val2',
    ]);
    $actual = json_encode([
        'key1' => 'val1',
        'key2' => 'val3',
    ]);

    // ...
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

In other situations, the failure output we get will be tailored to the assertion that failed.

Imagine we're comparing two JSON objects: we're expecting "key2" to have "val2" as its value, but our `$actual` has a value of "val3".

----

### JSON assertions

<pre class="code-wrapper fragment-replacement hide-line-numbers"><code class="hljs php language-php fragment fade-out" data-fragment-index="1">public function testJsonAssertions(): void
{
    // ...
    $this->assertSame($expected, $actual);
}</code><code class="hljs php language-php fragment fade-in" data-fragment-index="1">public function testJsonAssertions(): void
{
    // ...
    $this->assertJsonStringEqualsJsonString($expected, $actual);
}</code></pre>

<pre class="fragment fragment-replacement" data-fragment-index="0"><code class="hljs language-diff fragment fade-out" data-fragment-index="1">1) Tests\AssertionTest::testJsonAssertions
Failed asserting that two strings are identical.
--- Expected
+++ Actual
@@ @@
-'{"key1":"val1","key2":"val2"}'
+'{"key2":"val3","key1":"val1"}'</code><code class="hljs language-diff fragment fade-in overflow-hidden" data-fragment-index="1">1) Tests\AssertionTest::testJsonAssertions
Failed asserting that '{"key2":"val3","key1":"val1"}' matches (...)
--- Expected
+++ Actual
@@ @@
 {
     "key1": "val1",
-    "key2": "val2"
+    "key2": "val3"
 }</code></pre>

Note:

If we just compared the two as strings, PHPUnit would tell us that the strings don't match, but seeing *how* they differ is left up to us.

Meanwhile, the `assertJsonStringEqualsJsonString()` assertion will print a line-by-line diff *and* disregard the ordering of the properties.

----

### Assertion messages

<pre class="code-wrapper hide-line-numbers overflow-hidden fragment-replacement"><code class="hljs php language-php fragment fade-out" data-fragment-index="0">public function testGetOrderProcessDate(): void
{
    $order = new Order(/* ... */);

    $this->assertTrue(
        $order->hasBeenProcessed(),
        'Test is predicated on $order having been processed!'
    );

    // Make more assertions
}</code><code class="hljs php language-php fragment fade-in has-highlights" data-line-numbers="5-8" data-fragment-index="0">public function testGetOrderProcessDate(): void
{
    $order = new Order(/* ... */);

    $this->assertTrue(
        $order->hasBeenProcessed(),
        'Test is predicated on $order having been processed!'
    );

    // Make more assertions
}</code></pre>

```text
1) Tests\Unit\Models\OrderTest::testGetOrderProcessDate
Test is predicated on $order having been processed!
Failed asserting that false is true.
```
<!-- .element: class="fragment" data-fragment-index="1" -->

Note:

PHPUnit assertions generally accept an optional `$message` string as their last arguments, which lets us provide additional context when an assertion fails.

This not only adds details to failures, but can help document oddities in the codebase. In this case, we're making a quick assertion in our test that we've actually set up our `Order` object properly; it's not unreasonable for a `getOrderProcessDate()` method to behave differently if an order has not yet been processed!

----

#### Making assertion messages useful

```diff [1-5|7-11]
  $this->assertFileExists(
      $cacheFile,
-     'Assertion failed.',
+     'The cache file should have been created.'
  );

  $this->assertEmpty(
      $user->posts,
-     'Expected it to be empty.'
+     'Expected zero posts for the newly-created user.'
  );
```
<!-- .element: class="hide-line-numbers" -->

Note:

When writing assertion messages, make sure you're actually adding useful information.

For example, the failure message for `assertFileExists()` already tells us that the given file does not exist, so a message like "assertion failed" doesn't tell us anything. Instead, we might say "The cache file should have been created" so it's obvious why this assertion

For example, the output when `assertFileExists()` fails already says "Failed asserting that file (file) exists", so adding "assertion failed" doesn't tell us anything. Instead, provide context: we failed to verify that the appropriate cache file was created.

Similarly, `assertEmpty()` against an array will fail with "Failed asserting that an array is empty." Instead of just repeating that, we can explain that we expect the array of posts associated with a newly-created user should be empty because they haven't yet posted anything.

Now, if either of these assertions fail we can understand *why* the assertion was being made, which can help point us to where the error may be occurring.

----

### Sometimes, you assert nothing

<pre class="fragment-replacement hide-line-numbers"><span class="fragment fade-out" data-fragment-index="1"><code class="hljs php language-php fragment fade-out" data-fragment-index="0">public function testItDoesNotThrowAnException(): void
{
    $this->instance->doSomething();

    $this->assertTrue(true);
}</code><code class="hljs php language-php fragment fade-in" data-fragment-index="0" data-line-numbers="5">public function testItDoesNotThrowAnException(): void
{
    $this->instance->doSomething();

    $this->assertTrue(true); // No, bad. No cookie for you!
}</code></span><code class="hljs php language-php fragment fade-in" data-fragment-index="1">use PHPUnit\Framework\Attributes\DoesNotPerformAssertions;

#[DoesNotPerformAssertions]
public function testItDoesNotThrowAnException(): void
{
    $this->instance->doSomething();
}</code></pre>

Note:

If you're trying to write a test to basically verify that nothing blows up, be sure you're annotating it correctly.

I've fixed _so many_ tests in my career that use this pattern of `$this->assertTrue(true)`, which is just sloppy. However, PHPUnit will mark tests as risky if they don't make any assertions, so what are you supposed to do?

The correct approach is the `#[DoesNotPerformAssertions]` attribute, which tells PHPUnit "hey, we're testing something but won't be performing any assertions."

---

## Test doubles

Note:

As we've already established, we want our tests to be deterministic (same result every time we run it) and to focus on the system under test. In order to do so, we often rely on test doubles, which are stand-ins for other units of code.

We can use test doubles to ensure that we're not accidentally writing tests for code that we don't own, but we also use it to isolate the SUT.

----

### Five types of test doubles

**Dummy**<br>
Simply satisfies required arguments
<!-- .element: class="fragment" data-fragment-index="0" -->

```php
User::create('test@example.com', 'password123!');
```
<!-- .element: class="fragment" data-fragment-index="0" -->

**Fake**<br>
Test-only implementation of interface
<!-- .element: class="fragment" data-fragment-index="1" -->

```php
class TestLogger implements Psr\Log\LoggerInterface {}
```
<!-- .element: class="fragment" data-fragment-index="1" -->

Note:

The first two types of test doubles are pretty straightforward:

* A dummy simply exists to satisfy a required argument. Maybe you're passing a string like "first_name" or an empty array. These arguments have no bearing on behavior, we just need to satisfy requirements.
* A fake is an implementation of an interface that's used instead of the real class. Generally the fake implementations would not be production-ready.
    - A great example here is Monolog's `TestLogger`: instead of writing to a log file somewhere, it just logs everything to an array.
    - You can get the entries back out to ensure the right messages were logged.

----

### Five types of test doubles

**Stub**<br>
_When I call `X()`, return `Y`_
<!-- .element: class="fragment" -->

**Mock**<br>
_Expect `X(A, B)`, return `Y`_
<!-- .element: class="fragment" -->

**Spy**<br>
_Did `X()` get called with `A`, `B`?_
<!-- .element: class="fragment" -->

Note:

The other three types are all closely related:

* A stub allows us to dictate how something should behave; when I call this function/method, respond with this.
* A mock builds on this idea, letting us make assertions around interactions with the test double
    - Expect that this method will be called exactly two times with these specific arguments; when it is, return this value, execute this callback, throw this exception, etc.
* A spy is like a mock, but after the fact: after we've executed the system under test, verify that some method was called exactly once with these these arguments.
    - A great use-case would be ensuring that appropriate errors are emitted: when we forced this failure, did it trigger a call to our alerting service?

----

##### Mocking in practice

```php [|3-8|10|12]
public function testGetProducts(): void
{
    $products = [/* ... */];
    $repository = $this->mock(ProductRepository::class);
    $repository->expects($this->once())
        ->method('fetch')
        ->with('store_123', 20)
        ->willReturn($products);

    $store = new Store('store_123', $repository);

    $this->assertSame($products, $store->getProducts(20));
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

In this example, we're testing the `getProducts()` method on a store, which accepts a limit; in this case, 20 products.

We create a mock of the ProductRepository class, then tell it "when someone calls my `fetch()` method with arguments 'store_123' and 20, then return this array of product records."

We inject our mock via a constructor argument on the `Store` class, along with the store ID ("store_123").

Finally, we assert that we got back the 20 products we were expecting.

Note that we're not actually reading anything from the database or making any HTTP calls: we're telling `ProductRepository` how to behave, and PHPUnit will fail the test if we don't call `fetch()` exactly one time with those arguments. This lets us verify that we're interacting with `ProductRepository` correctly (e.g. the store ID is being passed along with the limit), but we're not concerned with how the internals of the repository are working.

----

#### Beware over-mocking!

```php[|4-6]
public function testValidateUsername(): void
{
    $validator = $this->createStub(Validator::class);
    $validator->method('validateUsername')->willReturn(true);

    $this->assertTrue($validator->validateUsername('sam'));
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

When working with test doubles, be careful that you're not overly-reliant on test doubles.

I've actually caught tests like this before: people were trying to test a method and ended up writing a test that stubbed the SUT.

This test does *nothing* except prove that PHPUnit stubs work.

---

## Test organization

> You miss 100% of the tests you can't find <cite>Me, just now</cite>

Note:

If you and your team can't find your tests, you're far less likely to maintain your tests (or write new tests). Take the time to make your tests discoverable and it will pay dividends.

----

### The hierarchy of PHPUnit tests

* <!-- .element: class="fragment" --> Test suite(s) > classes > methods > assertions
* <!-- .element: class="fragment" --> Unit tests: follow directory structure
* <!-- .element: class="fragment" --> Feature/integration tests: It Depends&trade;

Note:

A well-organized test suite is easier to maintain and for others to contribute to, so it's worth talking about how we organize tests.

* Test suites are comprised of one or more test classes, which contain test methods
    * Can have multiple test suites defined (e.g. unit, feature)
    * Test classes extend `PHPUnit\Framework\TestCase` and logically group test methods
    * Test methods are what we think of as "tests", which make then assertions. Think of these as test *scenarios*
* For unit tests, strongly recommend mirroring the app directory structure (e.g. unit test for `app/models/User.php` lives in `tests/app/models/UserTest.php')
* Integration/functional tests: More dependent upon organization, easier if you're practicing DDD
    * Remember: it's not illegal to put documentation in your test directories explaining the organization logic!

----

### Benefits of multiple test suites

* <!-- .element: class="fragment" --> Clear separation of intent
    * Unit, Feature, API, E2E, etc.
* <!-- .element: class="fragment" --> Only run relevant test suites
* <!-- .element: class="fragment" --> Can calculate code coverage separately

Note:

Remember that you can define multiple test suites, which can be a good way to split things up.

By doing so, we can clearly delineate between unit vs feature tests, API tests vs E2E, etc. For example, maybe there are different base test cases to enable things like database interactions for feature (but not unit) tests.

This is also the first of several ways to filter tests: maybe we want to run just unit tests, but leave out integration tests. Or maybe we only want to run E2E tests before a deployment, not for every PR. Splitting up our test suites gives us this flexibility.

----

### Organizing a test class

1. <!-- .element: class="fragment" --> Fixtures
2. <!-- .element: class="fragment" --> Test methods (follow order in SUT)
    1. <!-- .element: class="fragment" --> Generic, happy path
    2. <!-- .element: class="fragment" --> Other happy paths
    3. <!-- .element: class="fragment" --> Argument validation
    4. <!-- .element: class="fragment" --> Failure paths
3. <!-- .element: class="fragment" --> Helper methods, custom assertions, etc.

Note:

To keep your test classes organized, I generally recommend ordering your methods like this:

1. Fixtures (`setUpBeforeClass()`, `setUp()`, `tearDown()`, `tearDownAfterClass()`)
2. Test methods, which I'll generally try to order in a way that matches the class we're testing for easier discovery
    1. Generally I'll reserve `testMethodName()` for the most generic happy path
    2. If there are other happy paths (e.g. return from cache if it exists, behavior dependent upon passed arguments, etc.), these come next.
    3. Tests related to argument validation (empty strings/arrays, invalid objects, etc.)
    4. Places where we may trigger exceptions or errors
3. Helper methods, custom assertions, etc. (more on this in a minute)

Remember that the whole point is making sure that people can find the tests relevant to the code they're working on!

----

#### Organizing a test class (example)

```php[|1|3-11|13-16]
protected function setUp(): void {}

public function testCreate(): void {}

public function testCreateForExistingAccount(): void {}

public function testCreateThrowsOnEmptyUsername(): void {}

public function testCreateThrowsOnExistingUsername(): void {}

public function testCreateThrowsIfRecordCreationFails(): void {}

private function assertUserExists(
    string $username,
    string $message = ''
): void {}
```
<!-- .element: class="growth-spurt hide-line-numbers" -->

Note:

If we were testing a `create()` method, our test class might look something like this:

* Fixture(s) up top (`setUp()`)
* Happy paths, argument validation errors, other errors and exceptions
* Helpers 'n such (in this case, a custom assertion)

If the SUT also had a `delete()` method defined after `create()`, I'd put the tests for `delete()` after `testCreateThrowsIfRecordCreationFails()` and before `assertUserExists()`.

----

### Naming: to "test" or to `#[Test]`?

```php [3-6|1,8-12]
use PHPUnit\Framework\Attributes\Test;

public function testNewUserIsRegistered(): void
{
    // ...
}

#[Test]
public function itShouldRegisterANewUser(): void
{
    // ...
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

By convention, PHPUnit will assume that any public method on a test class that begins with "test" (e.g. `testTheThing()`) is a test method.

You can also use the `#[Test]` attribute to label test methods
    * Can be helpful when describing things in a more BDD way (e.g. `itShouldRegisterANewUser()`)

Ultimately, use whatever reads best to you and be consistent!

----

#### More on naming

* <!-- .element: class="fragment" --> Include the method name in unit tests
    ```diff
    - testNewUser()
    + testCreate()
    ```
* <!-- .element: class="fragment" --> Describe the scenario being tested
    ```diff
    - testBadUsername()
    + testCreateThrowsIfUsernameIsInvalid()
    ```

Note:

* When writing unit tests, include the function/method being tested in the test method name (e.g. `testCreate()` when testing the `create()` method)
* When naming tests with specialized scenarios (such as a test that verifies that invalid usernames trigger an exception), be descriptive!

----

### Test groups

Run related tests across classes & suites

* <!-- .element: class="fragment" --> Feature (e.g. "Login", "Checkout")
* <!-- .element: class="fragment" --> Domain (e.g. "Security", "Database")
* <!-- .element: class="fragment" --> Type of class (e.g. "Models", "Controllers")

Note:

PHPUnit includes the ability to tag test methods and classes into groups, which are free-form and can be applied as you see fit.

This is another way to filter which tests we're running.

----

#### Using test groups

```php[|1,4,7]
use PHPUnit\Framework\Attributes\Group;
use PHPUnit\Framework\TestCase;

#[Group('Models')]
final class UserTest extends TestCase
{
    #[Group('Login')]
    public function testCanLogin(): void
    {
        // ...
    }
}
```
<!-- .element: class="hide-line-numbers" -->

```sh
phpunit --group Models --exclude-group=Login
```
<!-- .element: class="fragment" -->

Note:

* In recent versions of PHPUnit, we use the `PHPUnit\Framework\Attributes\Group` attribute to apply groups
    - These stack, so `testCanLogin()` is part of both the "Models" and "Login" groups
* Groups can be assigned at the test class or test method level
* Running PHPUnit with the `--group` option lets us filter which tests should be run; we can also `--exclude-group`
    - This would run tests in `UserTest` _except_ `testCanLogin()` (or anything else in the "Login" group)

----

#### Group by test size

```php[|2,4-8|1,10-14]
use PHPUnit\Framework\Attributes\Large;
use PHPUnit\Framework\Attributes\Small;

#[Small]
public function testSomethingQuickly(): void
{
    // ...
}

#[Large]
public function testSomethingMoreComplicated(): void
{
    // ...
}
```
<!-- .element: class="hide-line-numbers" -->

```sh
phpunit --enforce-time-limit
```
<!-- .element: class="fragment" -->

Note:

PHPUnit also has three built-in groups (with their own attributes) to indicate relative test size: Small, Medium, and Large.

These groups can be used to control test execution limits. Example: automatically fail this small test if it takes more than 1 second to execute, but maybe give the large tests upwards of 15 seconds.

By default, these are 1, 10, and 60 seconds, respectively (but can be configured).

----

#### Group by ticket

```php [|1-3]
use PHPUnit\Framework\Attributes\Ticket;

#[Ticket('https://bugs.example.com/issues/123')]
public function testUsernamesCannotBeEmpty(): void
{
    // ...
}
```
<!-- .element: class="hide-line-numbers" -->

```sh
phpunit --group https://bugs.example.com/issues/123
```
<!-- .element: class="fragment" -->

Note:

Another useful way to apply groups is through the `#[Ticket]` attribute, which implicitly creates a new group around particular issue tracker URLs.

This can be really helpful with bugfixes to indicate "this test was written to help fix this bug" and give people a place to gain more context about the test/fix.

----

### A different brand of <abbr title="Don't Repeat Yourself">DRY</abbr>

Readability begets maintainability

Note:

Tests will inherently have some duplication, but that's okay!

A more readable test class will be a more maintainable test class, so don't go overboard trying to keep things DRY.

----

#### Break out common patterns

<pre class="fragment-replacement code-wrapper growth-spurt"><code class="hljs php language-php fragment fade-out" data-fragment-index="0">use GuzzleHttp\Client;
use GuzzleHttp\Handler\MockHandler;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Psr7\Response;

$client = new Client([
    'handler' => HandlerStack::create(new MockHandler([
        new Response(200, [], 'First response'),
        new Response(404)
    ]),
]);</code><span class="fragment fade-out" data-fragment-index="1"><code class="hljs php language-php fragment fade-in" data-fragment-index="0">use GuzzleHttp\Client;
use GuzzleHttp\Handler\MockHandler;
use GuzzleHttp\HandlerStack;
use Psr\Http\Message\ResponseInterface;

private function mockGuzzleClient(
    ResponseInterface ...$responses
): Client {
    return new Client([
        'handler' => HandlerStack::create(
            new MockHandler($responses)
        ),
    ]);
}</code></span><code class="hljs php language-php fragment fade-in" data-fragment-index="1">use GuzzleHttp\Psr7\Response;

public function testHandlingOf404Errors(): void
{
    $client = $this->mockGuzzleClient(new Response(404));
    $service = new MyService($client);

    // ...
}</code></pre>

Note:

If you've ever used Guzzle, you've probably had to use this pattern to stub out HTTP responses. However, this can be tedious to write over and over again.

We can simplify the process by creating a `mockGuzzleClient()` method, which accepts any number of `ResponseInterface` instances.

Now, to construct a client, we can just call `$this->mockGuzzleClient()` with our expected responses, then inject the client into whatever we're testing.

We can even take this a step further and break it out into a trait like `MocksGuzzle`, which could be applied to any test class that needs it.

----

#### Testing traits

* <!-- .element: class="fragment" --> Setting up user/application state
* <!-- .element: class="fragment" --> Custom assertions
* <!-- .element: class="fragment" --> Injecting mocks into global state
* <!-- .element: class="fragment" --> Common setUp/tearDown fixtures

Note:

Traits can be helpful for encapsulating common patterns across your test suite.

* This could be things like mocking Guzzle or setting up users, seeding activity, etc.
* Maybe you have some custom assertions that you want to make available for a subset of tests
* If your app relies on global state, you might want a single, canonical way of setting that up for tests
* Perhaps you have specific setup/teardown fixtures that you need to run

----

#### Testing traits vs base TestCases

```php[|1,5|2-3,7-8]
use PHPUnit\Framework\TestCase;
use Test\Support\AuthAssertions;
use Test\Support\MocksGuzzle;

final class MyClassTest extends TestCase
{
    use AuthAssertions;
    use MocksGuzzle;

    // ...
}
```
<!-- .element: class="hide-line-numbers" -->

**Pro-tip:** <u>one</u> base class per test suite
<!-- .element: class="fragment" -->

Note:

PHPUnit test classes extend `PHPUnit\Framework\TestCase`, but you are able to create custom sub-classes.

Example: setting up basic application state, database connectivity for integration tests, expose custom assertions, etc.

However, PHP is single-inheritance (classes can only extend a single class), so this can lead to base test cases getting bloated.

A better approach would be to use composition rather than inheritance: break things up into traits, which you can then import only where you need them.

General rule of thumb: if you find yourself with multiple base test cases in a single test suite (e.g. ApiTestCase, ModelTestCase, etc.), that could be a sign that things are over-engineered. Consider converting to traits and/or splitting into separate test suites.

---

## Writing maintainable tests

Note:

Now that our test classes are tidy, let's talk about how to improve the test methods themselves.

Please note that while the examples here are centered around PHPUnit, the principles are fairly universal across languages and test runners.

----

### Test Structure

Arrange, Act, Assert

(a.k.a. "Given, When, Then")<!-- .element: class="fragment" -->

Note:

When writing tests, we generally want to follow a pattern of "Arrange, Act, Assert":

1. Arrange: set up objects, prerequisites
2. Act: actually call/invoke the system under test
3. Assert: Make assertions about what happened, what we received, etc.

If you're more used to a BDD-style testing approach, you may also know this as "Given, When, Then"

----

#### Test orchestration

```php [|3-4|6-7|9-10]
public function testEmailValidationWithInvalidEmail(): void
{
    // Arrange/Given
    $email = 'this is not an email address';

    // Act/When
    $result = Validator::validateEmail($email);

    // Assert/Then
    $this->assertFalse($result);
}
```
<!-- .element: class="hide-line-numbers overflow-hidden" -->

Note:

This test could easily be reduced to a single line, but breaking it out to show the structure.

1. First, we set up what we need: this might involve constructing objects, setting up test doubles, etc.
2. Next, we invoke the SUT (in this case, call the `validateEmail()` method on our `Validator` class)
    - This should be expected, given the name of the test method
3. Finally, we make our assertion(s); in this case, we want to assert that `validateEmail()` returned false when given an email address of "this is not an email address"

----

### Fixtures

Methods that automatically run at certain points in the test lifecycle

```php
// Run before/after each test method.
protected function setUp(): void;
protected function tearDown(): void;
```
<!-- .element: class="fragment" -->

```php
// Run once at start/end of test class.
public static function setUpBeforeClass(): void;
public static function tearDownAfterClass(): void;
```
<!-- .element: class="fragment" -->

Note:

Fixtures are methods that you can define in a test class to handle common setup/tear-down operations.

The `setUp()` and `tearDown()` methods are run before/after each test method (respectively), and are great for setting up test doubles, overriding configuration, etc. Instead of having to explicitly do this at the start of each test method, we can define fixtures to ensure that each test is starting from the same place.

There are also static variants, `setUpBeforeClass()` and `tearDownAfterClass()`, which run at the beginning and end of the test class. These are useful for especially-expensive operations and/or ensuring one test class doesn't leak into another.

Whenever possible, you should focus on the `setUp()` side of things

----

#### Fixtures via attributes

```php [|1,5-9]
use PHPUnit\Framework\Attributes\Before;

trait FixturesWithAttributes
{
    #[Before]
    protected function doSomethingBeforeEachTest(): void
    {
        // ...
    }
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

If you're breaking things into traits and need to use fixtures, it's recommended that you use PHP attributes. This prevents any sort of conflicts arising from multiple traits defining things like `setUp()`: just mark the fixture methods with the appropriate PHP attributes: Before, After, BeforeClass, AfterClass.

----

#### Fixtures: in practice

```php [|1,7,11|6,12]
use PHPUnit\Framework\MockObject\Stub;
use PHPUnit\Framework\TestCase;

final class NewsFeedTest extends TestCase
{
    private NewsFeed $instance;
    private RssFeed&Stub $rssFeed;

    protected function setUp(): void
    {
        $this->rssFeed = $this->createStub(RssFeed::class);
        $this->instance = new NewsFeed($this->rssFeed);
    }
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

To see how fixtures might work in practice, let's imagine we have a `NewsFeed` class that accepts an instance of our `RssFeed` service class as a constructor argument.

We might create a stub within our `setUp()` fixture, so that we know we have a fresh instance at the start of each test.

Then we might create an instance of `NewsFeed`, injecting our stubbed RssFeed instance.

Note that if the constructor args will change between tests, we probably wouldn't want to do this in a fixture: generally speaking, we only want to perform steps in a fixture that we intend to run for the majority—if not all—of our test methods.

----

#### Using properties set in fixtures

```php [|7-10|12-13]
final class NewsFeedTest extends TestCase
{
    // ...

    public function testGetPosts(): void
    {
        $this->rssFeed->method('getFeedItems')->willReturn([
            new FeedItem(/* ... */),
            // ...
        ]);

        $posts = $this->instance->getPosts();
        // Now, make some assertions!
    }
}
```
<!-- .element: class="growth-spurt hide-line-numbers" -->

Note:

Now, we'll start writing our test methods. Since we've already stubbed out the `RssFeed` and `NewsFeed` instances in our fixture, we can simply reference them as properties within our test method.

First, we'll tell PHPUnit how we want our stubbed `RssFeed` instance to behave when we call `getFeedItems()`; since this is the happy path, let's say we want to return an array of `FeedItem`s.

Then we call `$this->instance->getPosts()`, which presumably calls `getFeedItems()` on our `RssFeed` instance, returning those `FeedItem`s we set up. Then we can make assertions: maybe the count, contents, etc.

----

#### Pro-tip: use fixtures for clean-up

```php [|8|10]
public function testUserIdDoesNotRandomlyChange(): void
{
    $auth = $this->stub(AuthService::class);
    $auth->method('isAuthenticated')->willReturn(true);

    AuthMiddleware::setService($auth);

    $this->assertTrue(AuthMiddleware::isLoggedIn());

    AuthMiddleware::reset();
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

If I had a nickel for every time I've seen this in a test, I'd be rich enough to never have to hear the phrase "static property" again.

If your tests have to manipulate static properties and/or global state, be sure to reset things in a `tearDown()` fixture, **not** in the test itself.

In this case, if our assertion fails, PHPUnit will bail out immediately, meaning the `AuthMiddleware::reset()` call never gets made. This can then cause our stubbed `AuthService` to leak into subsequent tests, which can be a nightmare for debugging.

----

### Data providers

Provide multiple scenarios for the same test

```php[|3]
use PHPUnit\Framework\Attributes\DataProvider;

#[DataProvider('provideSomeMethodScenarios')]
public function testSomeMethod(string $input, string $expected): void
{
    // Some complicated setup, perhaps?

    $this->assertSame($expected, $sut->someMethod($input));
}
```
<!-- .element: class="hide-line-numbers overflow-hidden"-->

----

#### Data provider method

<pre class="fragment-replacement"><code class="hljs php language-php fragment fade-out" data-fragment-index="0">/**
 * @return array&lt;string, array{string, string}&gt;
 */
public static function provideSomeMethodScenarios(): array
{
    return [
        'first scenario' => ['input', 'expected_output'],
        // ...
    ];
}</code><code class="hljs php language-php fragment fade-in" data-fragment-index="0">/**
 * @return iterable&lt;string, array{string, string}&gt;
 */
public static function provideSomeMethodScenarios(): iterable
{
    yield 'first scenario' => ['input', 'expected_output'];
    // ...
}</code></pre>

Note:

The data provider itself is responsible for providing scenarios for the test method: in this case, we're returning an array consisting of two strings for each scenario.

These will get passed to our `testSomeMethod()` test method's `$input` and `$expected` arguments.

If you'd like fewer arrays in your life, it's also worth noting that we can return a generator by using `yield` statements; this can make the list much cleaner if you have a lot of scenarios.

In either case, PHPUnit will use the key to create a name for the scenario (e.g. "testSomeMethod @ first scenario"), making it easier to troubleshoot failures and explain what's special about each scenario.

----

#### Avoid conditionals in tests

<pre class="fragment-replacement growth-spurt"><code class="hljs php language-php fragment fade-out" data-fragment-index="0">public static function provideBadTestScenarios(): iterable
{
    yield 'Happy path' => [
        [
            'input' => 'some input',
            'expected_exception' => null,
        ],
    ];
    yield 'Error state' => [
        [
            'input' => 'bad input',
            'expected_exception' => \RuntimeException::class,
        ],
    ];
}</code><code class="hljs php language-php fragment fade-in" data-fragment-index="0">#[DataProvider('provideBadTestScenarios')]
public static function testYourPatience(array $args): iterable
{
    // ...

    if (!empty($args['expected_exception'])) {
        $this->expectException($args['expected_exception']);
        $sut->someMethod($args['input']);
    } else {
        $this->assertTrue($sut->someMethod($args['input']));
    }
}</code></pre>

Note:

When you first learn about data providers, it can be really tempting to put every possible scenario in there and have one test method per method being tested. I **strongly** advise against doing this, as it will quickly make your test suite unmaintainable.

If someone can't look at your test method and immediately discern what's being tested, it's unlikely that anyone will ever want to touch that test again. The affirmative, happy path test methods should be separate from the negative/failure cases, which should also be separate from any exception/error expectations.

Instead, consider having a couple data providers: one might provide a couple scenarios for the happy path, another might have scenarios that trigger exceptions. Don't make your tests harder to follow by prematurely lumping them into data providers to save a few keystrokes.

---

## Testing beyond the happy path

* <!-- .element: class="fragment" --> What are the normal ("happy") routes?
* <!-- .element: class="fragment" --> What are the error states?
* <!-- .element: class="fragment" --> How do we handle invalid input?

Note:

A common gap in test suites is that people are much more likely to test the so-called "happy" paths—when everything goes right—than they are to test what happens when things go wrong.

When writing tests, it's important to write tests that verify error behaviors. Are we throwing the appropriate exception? Are we leaving things in a half-modified state?

Furthermore, what happens if we're given invalid data? We're expecting an email address as a string, but we're given an empty string: do we keep processing? Should we expect a validation error? What about a negative query limit?

Some of the most valuable tests are those that prove how things work under non-ideal situations.

----

### How many paths can you find?

```php [|3-11|3,9-11|4-8|5]
public function getClient(): Client
{
    if (!isset($this->client)) {
        try {
            $this->client = new Client($this->authToken);
        } catch (ClientException $e) {
            throw new InvalidClientException(/* ... */);
        }
    }

    return $this->client;
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

Given this `getClient()` method, let's look at all of the possible ways we can work through it:

1. If we don't have a client yet, we need to construct one. Do we get an instance of Client?
2. If we already have a client, are we getting back the same, cached instance with each call?
3. If constructing a new Client causes a `ClientException` to be thrown, are we then re-throwing as an `InvalidClientException`?
4. Are there other exceptions that might occur that we might intentionally not be catching?
5. What happens if `$this->authToken` is empty or of an invalid type?

---

## Writing tests for existing code

Note:

I've thrown a lot at you so far, so let's start applying some of these practices with actual code:

----

### Simple unit tests

```php
namespace App;

class Str
{
    public static function snakeCase(string $str): string
    {
        $str = preg_replace('/([A-Z]+)/', '_$1', $str);
        $str = preg_replace('/[^A-Z0-9_]/i', '_', $str);
        $str = preg_replace('/_{2,}/', '_', $str);

        return trim(strtolower($str), '_');
    }
}
```

Note:

Let's imagine we have a method like this in our codebase, which takes a string and produces the snake_case equivalent. Unfortunately, whoever wrote this didn't write any tests!

Fortunately, this is a very simple method: we give it a string and get a string in return. No other dependencies, nothing but pure PHP. This makes it a prime candidate for a unit test.

----

#### Testing snakeCase()

```php [|1|3-4,7|5,8]
namespace Tests\Unit\App;

use App\Str;
use PHPUnit\Framework\Attributes\CoversClass;
use PHPUnit\Framework\TestCase;

#[CoversClass(Str::class)]
final class StrTest extends TestCase
{
    // ...
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

If we don't yet have a unit test class for this `Str` class, let's create one.

1. We're putting this in the `Tests\Unit\App` namespace: this is in our unit test suite (`Tests\Unit`), then mirroring the app structure (SUT is `App\Str`)
2. We're using the `#[CoversClass]` attribute to make it clear that this test class is designed to test methods within the `App\Str` class
3. We're extending the base `TestCase` class from PHPUnit, as there's nothing special going on here

----

##### Scenarios for snakeCase()

Three approaches:

1. <!-- .element: class="fragment" -->One method, many assertions
2. <!-- .element: class="fragment" -->Many methods, one assertion each
3. <!-- .element: class="fragment" -->One method, data provider

Note:

When handling unit tests for simple I/O methods like this, there are generally three approaches

1. One test method, with a bunch of assertions (one for each scenario)
    - Important to explain *why* each assertion fails
    - If one assertion fails, all subsequent assertions are skipped
2. Many test methods, with one assertion each
    - A lot of repeated scaffolding, possibly overkill
3. One test method, with different input and output combinations fed into it from a data provider
    - Scenarios are run separately with minimal setup
    - Adding new scenarios is as simple as updating the data provider

----

##### Data Provider for snakeCase()

```php [|4-9|1-3,9-14]
use PHPUnit\Framework\Attributes\DataProvider;

#[DataProvider('provideSnakeCaseScenarios')]
public function testSnakeCase(
    string $str,
    string $expected
): void {
    $this->assertSame($expected, Str::snakeCase($str));
}

public static function provideSnakeCaseScenarios(): iterable
{
    // ...
}
```
<!-- .element: class="hide-line-numbers overflow-hidden" -->

Note:

To set this up, we'll define a `testSnakeCase()` test method, which accepts two strings: the string to be tested and the expected result.

The assertion itself is simple: when we call `snakeCase()` on the given string, we expect it to be identical to `$expected`

To provide these scenarios, we'll create a static method, `provideSnakeCaseScenarios()` and tell PHPUnit to use it as a data provider via the `#[DataProvider]` attribute.

----

##### Try to Break snakeCase()

* <!-- .element: class="fragment" --> Single vs multiple words?
* <!-- .element: class="fragment" --> Punctuation, whitespace?
* <!-- .element: class="fragment" --> Text already in snake_case?
* <!-- .element: class="fragment" --> Common forms (e.g. camelCase, PascalCase)?

Note:

Here's the fun part: now we can think through how the method _should_ work, then write scenarios to verify that behavior.

In this case, we define how strings should be converted to snake_case:

* If it's a single word, we don't expect to see underscores; multiple words should be separated by underscores
* Everything besides alphanumeric characters should be stripped: punctuation, whitespace, etc.
* What happens if we're given something that's already in snake_case?
* How should we handle capital letters within words, as we might find with camel and/or PascalCase?

Don't be surprised if you come up with scenarios that are handled incorrectly in the existing code: if it was complete, it would already have comprehensive tests.

----

###### provideSnakeCaseScenarios()

```php
public static function provideSnakeCaseScenarios(): iterable
{
    yield 'Single word' => ['Hello', 'hello'];
    yield 'Multiple words' => ['Hello there', 'hello_there'];
    yield 'Alphanumeric strings' => ['Check check 123', 'check_check_123'];
    yield 'Multiple words with punctuation' => ['Hello, there!', 'hello_there'];
    yield 'Multiple whitespace characters' => ['Hello ,  there', 'hello_there'];
    yield 'Already in snake case' => ['hello_there', 'hello_there'];
    yield 'camelCase' => ['helloThere', 'hello_there'];
    yield 'PascalCase' => ['HelloThere', 'hello_there'];
}
```
<!-- .element: style="font-size: .45em;" -->

Note:

(May be hard to read)

For each of this scenarios, we add an entry to our data provider. This can use generators as I've done here or use an associative array, but PHPUnit will pick up the keys we use when describing the test (e.g. `testSnakeCase()` with scenario "Multiple words"). This also provides better context as to *what* is being tested, which makes the tests more maintainable.

Each line here results in a new test being run, which can pass or fail independently of the other scenarios; if you come up with another scenario (like leading/trailing underscores), it's simply a matter of adding it to your data provider!

----

#### Tests vs type-hints

What about tests for bad arguments?

```php
Str::snakeCase(['array', 'of', 'strings']);
```

```php
public static function snakeCase(string $str): string;
```
<!-- .element: class="fragment" -->

```text
Fatal error: Uncaught TypeError: Str::snakeCase():
Argument #1 ($str) must be of type string, array given
```
<!-- .element: class="fragment" -->

Note:

"What about protecting against non-strings being passed to our method?", you might ask.

If you recall, we added both argument and return type-hints, which means that PHP is going to reject any non-strings automatically. We've pushed this off onto PHP's type engine, which isn't code that we own, so we don't need to concern ourselves writing tests for it!

Adding the type-hints here will also make it super-easy for IDEs and static code analysis tools like PHPStan to say "hey, you're trying to call snakeCase() with an array, but it only accepts a string!"

Generally speaking, the stricter your types, the less you need to explicitly write tests for. Once again, embracing strict types makes your life easier!

----

### Testing More-Complex Logic

```php[|5-7|9-11|]
public function makeThingsSoComplicated(
    string $path,
    array $args = []
): ?string {
    if (empty($path)) {
        throw new \InvalidArgumentException('$path cannot be empty!');
    }

    if ($args['skipComplications'] ?? false) {
        return null;
    }

    return str_rot13(md5($path . json_encode($args)));
}
```
<!-- .element: class="hide-line-numbers growth-spurt" -->

Note:

This is a silly example, but demonstrates my point well: how many paths do we have through this method?

1. The happy path, which results in a rot13'd, md5 checksum of the path and a JSON representation of the $args array
2. If we pass an empty $path, we should get an exception
3. If we set a truthy "skipComplications" key in $args, we should get null
4. If we pass a false-y skipComplications, we should still get that md5 checksum

I find that it can be really helpful to start from the top and read through, noting any places where the behavior might change. Each of these should be its own test scenario.

----

#### Testing makeThingsSoComplicated()

```php[|1-9|11-15]
public function testMakeThingsSoComplicated(): void
{
    $this->assertSame(
        'o0478736258o8233272rr9727q21pq60',
        $this->instance->makeThingsSoComplicated('Oh,', [
            'hello' => 'there',
        ]),
    );
}

public function testMakeThingsSoComplicatedThrowsOnEmptyPath(): void
{
    $this->expectException(\InvalidArgumentException::class);
    $this->instance->makeThingsSoComplicated('', ['abc' => '123']);
}
```
<!-- .element: class="hide-line-numbers growth-spurt" -->

Note:

Let's work down that list of scenarios:

1. First we have the happy path: if we call it with $path "Oh," and $args ['hello' => 'there'], we expect to see this rot13'd checksum
    - In this case, I precalculated it. You might also choose to generate that checksum within the test to make it more obvious where it's coming from
2. Next, let's verify that we throw an `InvalidArgumentException` when $path is empty

Notice that these tests are very simple: they're testing a single scenario and, since this class has very little setup work, we could do that in a fixture and just call the method on `$this->instance`. However, if suddenly `testMakeThingsSoComplicatedThrowsOnEmptyPath` starts failing, we know exactly where to look.


----

#### Testing makeThingsSoComplicated()

```php[|1-8|10-18]
public function testMakeThingsSoComplicatedSkipComplications(): void
{
    $this->assertNull(
        $this->instance->makeThingsSoComplicated('Hello!', [
            'skipComplications' => true,
        ]),
    );
}

public function testMakeThingsSoComplicatedWithFalseySkipComplications(): void
{
    $this->assertSame(
        '930268r85896s9976594043n5s9q9s31',
        $this->instance->makeThingsSoComplicated('Hello!', [
            'skipComplications' => false,
        ]),
    );
}
```
<!-- .element: class="hide-line-numbers growth-spurt" style="font-size:.45em;" -->

Note:

To round out the other two scenarios, we do the same thing:

1. If we set the "skipComplications" argument to true, we expect to get a null return value
2. If "skipComplications" is there but not truthy, we expect another checksum (just like in the first test)

With these four tests, we've covered every logical branch through this method. If someone removes the empty() check, or skipComplications changes, or someone changes how the checksums are calculated...we'll know.

----

### Should I have AI write my tests?

![Homer Simpson looking cheeky, asking "Can't AI do it?"](resources/cant-ai-do-it.gif)

Note:

A lot of people are talking about using AI these days to write tests, and there are a few schools of thought:

1. No.
2. Hell no.
3. Fine, but only to help scaffold your test class.
4. What could go wrong?

Tests are meant to be the objective arbiters of truth: if your tests are wrong, everything they're meant to protect is at risk.

If you _must_ use AI, be especially careful when reviewing its output; incorrect tests can be worse than having no tests at all.

If you're writing new code, you might even consider _only_ writing your tests, then tell AI that it can pound sand unless it can write code that causes all of those tests to pass ("Test-Driven Vibe Coding")

---

## Writing tests for existing ~code~ bugs

Note:

Now that we've backfilled our test suite, let's conclude by talking about regression tests.

----

### Regression testing in a nutshell

1. <!-- .element: class="fragment" --> Write test(s) to reproduce bug
2. <!-- .element: class="fragment" --> Fix bug (verified by tests passing)
3. <!-- .element: class="fragment" --> Never fix this bug again!

Note:

Writing regression tests can be one of the most effective way to fix bugs and ensure they _stay_ fixed. In practice, it looks a lot like regular ol' TDD:

1. When we find a bug, we should start by writing one or more tests to reproduce the behavior. This helps us determine how/why it's happening.
    - Tests should be failing, because we haven't fixed the bug yet!
2. Now we fix the bug; we know it's fixed because our tests should now be passing.
    - If the bug is "fixed" but the tests still fail, either the fix is incomplete or tests are bad!
3. Commit the new tests to your test suite, ideally using the `#[Ticket]` attribute to point back to the original bug ticket
    - If your fix ever gets removed/squashed/whatever, the tests will catch it and say "hey, we already fixed this!"

----

#### Regression testing example

```php [|6-7|1,3]
use PHPUnit\Framework\Attributes\Ticket;

#[Ticket('https://bugs.example.com/issues/123')]
public function testSnugglePetWithFish(): void
{
    $this->expectException(DoNotSnuggleException::class);
    $this->petOwner->snugglePet(new Fish('Nemo'));
}
```
<!-- .element: class="hide-line-numbers" -->

Note:

For a silly example, imagine we have a PetOwner class that includes a method named `snugglePet()`. However, we've received a bug report that people are able to snuggle their fish, which is generally not a great idea.

We _should_ be throwing a `DoNotSnuggleException` (which we originally implemented for reptiles and birds), but apparently we overlooked marine animals.

We'll write a quick test that verifies that we get a `DoNotSnuggleException` when we attempt to snuggle a fish; before we fix the bug, this test will fail. However, once we fix the bug, the test should pass.

Note that I included the `#[Ticket]` annotation, which creates a new group for this issue URL. It also makes it very clear that this test was a bugfix, and the URL where someone could get more context if they needed it.

---

## Test coverage

----

### What is test coverage?

The percentage of the total codebase that was executed by the test suite(s)

Note:

We'll spend a few minutes on this topic, but it boils down to this: test/code coverage is simply what percentage of the codebase was executed when we ran the test suite.

----

### Types of test coverage

<dl>
    <dt class="fragment" data-fragment-index="0">Line coverage</dt>
    <dd class="fragment" data-fragment-index="0">Which lines were executed?</dd>
    <dt class="fragment" data-fragment-index="1">Branch coverage</dt>
    <dd class="fragment" data-fragment-index="1">Which logical branches were executed?</dd>
    <dt class="fragment" data-fragment-index="2">Function coverage</dt>
    <dd class="fragment" data-fragment-index="2">Which functions/methods were called?</dd>
</dl>

Note:

Often when we talk about "code coverage", we're referring to line coverage: which lines were—or were not—executed as we run the test suite?

However, this doesn't give us the full picture, so we can also calculate branch coverage (are all permutations through a portion of code covered by tests?) and function coverage (which functions/methods were actually executed?).

There are also other forms—namely statement and condition coverage—but these three (especially line) are what most people think about. However, as you'll see, code coverage might not be all it's cracked up to be:

----

### Illustrating test coverage: Boolean

```php
class Boolean
{
    public static function true(): bool
    {
        return true;
    }

    public static function false(): bool
    {
        return false;
    }
}
```

Note:

To demonstrate test coverage, imagine we have a Boolean class with two methods, `true()` and `false()`, which simply return their respective boolean value.

----

#### Testing `Boolean`

```php[|8]
use PHPUnit\Framework\Attributes\CoversClass;

#[CoversClass(Boolean::class)]
final class BooleanTest extends TestCase
{
    public function testTrue(): void
    {
        $this->assertTrue(Boolean::true());
    }
}
```
<!-- .element: class="hide-line-numbers" -->

**Test Coverage:** 50%
<!-- .element: class="fragment" -->

Note:

Now we'll write a test, and assert that the return value of the `true()` method is indeed true.

Note that we're also using the `#[CoversClass]` attribute, which lets us specify "this test class is only meant to cover the Boolean class, so don't count this towards any other class' test coverage". This is a good habit that can be enforced by PHPUnit.

We haven't tested the `false()` method, so we've tested 50% of the `Boolean` class.

----

#### Testing `Boolean` (poorly)

```php [8]
use PHPUnit\Framework\Attributes\CoversClass;

#[CoversClass(Boolean::class)]
final class BooleanTest extends TestCase
{
    public function testTrue(): void
    {
        $this->assertIsBool(Boolean::true());
    }
}
```
<!-- .element: class="hide-line-numbers" -->

**Test Coverage:** (still) 50%
<!-- .element: class="fragment" -->

Note:

Now, let's change the test a bit: instead of asserting that the `true()` method returns true, we'll just verify that it returns a boolean value.

What does this do to our test coverage? Not a damn thing. We could rewrite the `true()` method to return `false` and our tests would still pass and we'd still have 50% coverage.

What does this tell us?

----

### Test coverage is <u>not</u> (necessarily):

* <!-- .element: class="fragment" --> An indication of code quality
* <!-- .element: class="fragment" --> Proof that code is bug-free

Note:

It's important to recognize that test coverage does not measure the quality of our tests, nor does it guarantee that our code is bug free.

High test coverage does not equate to *good* test coverage; all it does is tell us which portions of our code are executed when we run our tests.

----

### Is test coverage a useful metric?

![Scott Lang (Paul Rudd) from Avengers: Endgame (2019) asking "So Test Coverage is a bunch of bullshit?"](resources/scott-lang-test-coverage.gif)

Note:

So what good is code coverage, then?

* Code coverage is only one factor to consider with regards to code quality
    - Identify the areas of our application, logical branches, etc. that don't have any tests
    - Combine with type coverage (via static code analysis), cyclomatic complexity, etc.
* Don't focus too much on the coverage percentage: it's an attractive metric for dashboards, but doesn't really tell us anything useful by itself.
    - 50% coverage with good tests beats 100% coverage with bad tests

---

## Thank You!

Steve Grunwell<br>
<span style="font-size: .75em;">Staff Software Engineer, Mailchimp</span>

[stevegrunwell.com/slides/better-tests](https://stevegrunwell.com/slides/better-tests)<br>
[joind.in/talk/cb4cd](https://joind.in/talk/cb4cd)
<!-- .element: class="slides-link" -->

Note:

REMEMBER TO REPEAT THE QUESTION!!
