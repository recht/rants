# Testing for fun and profit

Many people regard testing as somewhat of a necessary evil. They see some value to doing it, but most often testing is an afterthought. In this piece, I'd like to explain why that shouldn't be the case.

Let me start by saying that I'm not really religious about when tests are written or which framework is used. I am pretty religious about the style of the tests and that they're supposed to do. Being religious about anything is usually a bad thing, and I'm sure what I describe here is not the end solution. It's the best I know of at the moment, though, and it's been refined over many years.

In this entire piece I'm assuming that you're in an environment where specifications are few don't exist and where developers are responsible for testing their own code.


## The purpose of testing

What is the purpose of a test? You'll probably say something along the lines of "to make sure stuff works". That's not wrong, but it's also not completely right.

To me, the real purpose of a test is quite simple: a test is a specification of a behavior of the system being implemented.

What does this mean? That's what I'll use the remainder of this article to explain, but it basically boils down to this:

- By reading the tests I can understand how the system works.
- By having the tests I'm sure I don't break intended behavior when I refactor or add new code.
- By writing the tests I can explain my intention to both other developers and to my future self.

That's on a fairly abstract level. On a code level, this means:

- Use proper test naming.
- Use plain and precise language when naming.
- Keep tests small to focus them on single behaviors.
- Keep tests maintainable just like you would with normal code.

Let's look at each of these below.


## Proper test naming

(I also call this BDD naming. When I review code and see this being off I also just write "naming".)

The main purpose of test naming is to convey the specification of a behavior. This can be done in a number of ways, but I've found that by far, the easiest and best way is to use the method name of the test (if your framework of choice doesn't use test methods then use whatever mechanism comes with the framework).

In the good old days when jUnit just came out, Eclipse added this wonderful feature where you could right-click a class and select "New jUnit test". That would create a new class with one method per public method in the original class. Each method name would be prefixed with "test".

Many people do the same today. However, in my opinion, it's one of the worst test practices out there. Why? Having a method called `get_state` and a test method called `test_get_state` doesn't convey any information except that there is indeed a method called `getState` and that a test presumably calls it. It doesn't in any way explain how the system works.

Let's say that the method implementation looks something like this (to keep this article from getting too long, the example is very simple - bear with me):

```
def get_state(self):
    if self.last_success is None:
        raise Exception('There is no state')
    if self.last_success:
        return 'OK'
    else:
        return 'ERROR'
```

The usual `test_get_state` would probably look something like this:

```
def test_get_state():
    obj = Obj()
    assert obj.last_success is None
    obj.last_success = True
    assert obj.get_state() == 'OK'
    obj.last_success = False
    assert obj.get_state() == 'ERROR'
    try:
        obj.last_success = None
        assert False
    except:
        pass
```

The test provides full coverage, so that's a good thing. However, it still doesn't really explain the behavior in readable language - indeed, it's hard for anybody to explain what the intended behavior is (as opposed to accidental behavior).

In a much better world, there would be several tests, each specifying a different behavior. For example:

```
def test__getting_state_before_anything_has_run_will_raise_an_exception():
    obj = Obj()
    try:
        obj.last_success = None
        assert False
    except:
        pass


def test__getting_state_after_successful_execution_returns_ok():
    obj = Obj()
    obj.last_success = True
    assert obj.get_state() == 'OK'


def test__getting_state_after_failed_execution_returns_error():
    obj = Obj()
    obj.last_success = False
    assert obj.get_state() == 'ERROR'
```

Now it's suddenly quite clear to everybody (including non-programmers) what the intended behavior is. Sure, there's also more code now, but that's only due to two reasons: the original test did nothing to explain what the intended behavior is, and there's some code duplication going on. Had this been a real test, I'd probably have created a small helper method to use, thus shorting the body of each test to just a single line:

```
def get_obj(last_success):
    obj = Obj()
    if last_success is not None:
        obj.last_success = last_success
    return obj
```

By the way, I usually begin test methods in Python with `test__` - Python test frameworks usually detect test methods by looking at the name, so they have to start with `test`. Using `test__` is just a way of not making `test` part of the spec. You could of course use names such as `test_that_when_...`.

Making it explicit what the intended behavior is also makes code much easier to review and reason about. It also makes it more natural for a reviewer to spot behaviors which are not yet tested, typically negative tests which are often forgotten. For example, if in the above example there was only `test__getting_state_after_successful_execution_returns_ok` then as a reviewer, I would immediately think "so what about failed execution?" and ask the author to add more tests.

A final point about naming is that it can help new developers gain confidence in refactoring. If they refactor and get precise and meaningful feedback on failures then it's much easier to change code. Having good naming also makes it possible to fairly quickly assess the behavioral test coverage of a system, again so that refactoring becomes safer.


### Common complaints

- Why not just comments?

  In theory you can. However, in reality, comments tend to get out of date very fast. More important, if you make a change and a test fails then it's easy for you to spot what's wrong by looking at the test output if the test is named correctly.

  A second reason is that tooling works much better. For example, `git diff` will show which method a change is in, so if you're modifying an existing test method then it's very easy for reviewers to see which behavior you're changing. Another example is that you can get an editor outline of all test methods, copy the list to for example your product designer or other team members and ask "if my system does all this, is the system then complete?".

- Test method names are becoming too long which violates our standards!

  Two things you can do about that: live with it (I've found that it's possible to live with) or come up with shorter, yet meaningful, names. Of course, if you have a test named something like `test__calling_method_x_will_save_an_entity_to_the_database_and_put_it_on_cloud_storage_as_well_as_on_disk_and_then_convert_it_to_pdf_and_return_it_as_xml` then you might be in trouble (in more than one way). However, this is actually a symptom of not keeping to a single behavior per test. I'll get back to that later on.

- I understand my tests just fine without this, thank you very much!

  I would hope so. However, how easy is it for you to understand a test you wrote a year ago? How easy is it to understand for a newly hired developer? How easy is it to review?

  In other words, why not just make it easier for everybody to read your code?


## Clear and precise naming

There are two points to this: use English language that's semantically correct and precise, and use snake_casing for the method names.

Snake casing basically uses lower-case characters and `_` between words - this is the default in Python and many other languages, and should be used in test naming to make the names easier to read. When using camelCasing, for example, `getting_state_after_failed_execution_returns_error` becomes `gettingStateAfterFailedExecutionReturnsError`, which is just harder on the eyes. It goes against all naming conventions in for example Java, but it's worth it (should you be using Groovy or another language which allows strings as method names then just use spaces).

Using correct and precise language means that a test name should be a regular sentence, just like one of those things you were taught in school. Some examples of invalid test names:

- `test__state_after_successful_execution_ok`
- `test__get_state_ok`
- `test__getting_state_fail`

Secondly, the names should be precise. Remember that a test method defines behavior, so you can't assume that the reader will know the intended behavior by any other means. This most often boils down to avoiding words like "correct", "properly", "works" in method names. For example, these names don't really make sense:

- `test__get_state_works`
- `test__getting_state_returns_correct_state`

When I see tests like this, I can only ask "how should it work?" and "what is the correct state?"

### Common complaints

- It takes up too much space to write that much!

  Test method names can become rather long, yes. I don't see that as an issue if it creates clarity.

- Comments are much better suited for stuff like this.

  See the section on proper test naming.

- Why explain what the correct behavior is when it's obvious from reading the code?

  First of all, what's obvious to you might not be obvious to someone else. Secondly, it can be hard to know what is intended behavior and what's accidental behavior. If you've explicitly declared that a certain behavior must hold then it's much easier to reason about failures later on.


## Temporal test naming

Remember to explain in the test names when a certain behavior is expected (and who it's expected by, if it's not given by the context of the test file/class/module).

For example, `test__getting_state_returns_ok` is valid English, but it doesn't really explain when that's the case. That's why the example above has it as `test__getting_state_after_valid_execution_returns_ok`.

### Common complaints

None so far.


## Keep tests small and focused

A single test should only test a single behavior of the system (system can either be the system as a whole, a class or module, or a method). One easy way of verifying this is to check if the word "and" is in the test name. If it is then it's quite likely that the test is testing multiple behaviors (it's not always the case, but often it is).

Let's say you have a method which takes an object and saves it to both a local cache and a persistence layer. The wrong test would be `test__saving_object_adds_it_to_local_cache_and_saves_it_in_mysql`. Better tests would be `test__saving_object_adds_it_to_local_cache` and `test__saving_object_saves_it_in_mysql`. Incidentally, this is also how you avoid the very long test names.

This also means that the test should only be asserting things which are relevant to the behavior being tested. Assume that all other behavior is being tested elsewhere.

Why is it better? First of all, it keeps the test methods short. Like normal business methods, methods really shouldn't span many lines.

Secondly, it separates concerns. If there only is the single test then if you break whatever assertion came first then you don't know if you also broke the remaining behavior. By splitting the test into separate concerns it becomes much easier to pinpoint where the issues are when you refactor code.

### Common complaints

- But what if I need to test a long chain of calls?

  Consider if your architecture is good. If you really have to do a lot of work to set up your test then perhaps you're holding it wrong. Consider maybe stubbing or mocking some of the dependencies.

  If you really do need a long setup sequence then just do that - but put the setup in a fixture method and reuse that across tests. Often, when you have a complicated setup then you also have many behaviors you want to test at the end of the setup.

- It's crazy! I now need to run the setup code for every small test, and the setup is so expensive!

  Again, consider if you're doing it right. Even if you are then it should be possible to do some caching of the setup. For example, Python's `py.test` has the concept of fixtures which can be scoped to a module.


## Keep tests maintainable

To me, the most important part of a system is actually the tests. Without them I basically don't know anything about the system - I certainly don't know if it actually works. And if it does seem to work then I don't know what the intended behavior was.

Given that, it seems reasonable to ask that test code be treated just like normal production code. This means proper naming of all method, not just test methods, refactoring tests and test fixtures when needed, avoiding code duplication, adding abstractions where necessary, etc.

I've even gone so far as to review tests first and then review the rest once I've read the tests and gotten an understanding on how the system is intended to work. If the tests are not structured nicely then I'll just stop there and ask for changes. OK, sometimes I'm not that nazi, but having nicely structured tests makes it much easier to review code.

### Common complaints

None yet.


## Technicalities

You'll notice that I haven't really talked much about test frameworks, how to run tests, if tests should be unit tests, integration tests, or something in between (or around). These are all interesting topics, but I'm happy if people first of all just write tests. The level of testing and which framework is being used comes after that.

I will say, though, that in Python I prefer [py.test](http://pytest.org/) with just plain functions (no wrapper classes) and `pytest.fixture` for fixtures. As for the level of testing, I tend to do more system testing than unit testing, but that's mostly because the systems I work on are normally DB heavy and most of the business logic is just pushing bits around. Doing pure unit testing of that usually requires a fair amount of mocking, which gets a little tedious.


## Conclusion

I hope that by now, you've at least understood the importance and value of tests. I know that some people won't agree with all my points (or even any of them). In that case, I hope that they at least have a viable alternative which isn't just "I can understand my tests just fine" or "I have never had any issues with my tests". It's not all just about yourself, it's also about the greater good of everybody involved in development.

So, to sum up, here are the main points:

Tests exist because:

- They serve as documentation of the system.
- They help guard behavior against regressions.

When writing tests do this:

- Use proper test naming: explain the intended behavior in the test name.
- Use clear and precise naming: use English language and avoid terms like "correct", "works", "properly".
- Use snake_casing to increase readability.
- Keep tests to a single behavior. Only do assertions for things which are related to the behavior being tested.
- Behaviors should be granular. Avoid the word "and" in test names, prefer splitting into multiple separate tests instead.
- Write test code like you write production code. Use all the usual practices around good code, such as reuse, abstractions, low coupling, etc.
- Use fixture methods to move setup code out of the actual test methods.

And finally, if you want to be an efficient reviewer and appear smart (perhaps smarter than you actually are):

1. Start by reading the test names. If they don't make sense, make a comment.
1. Consider if some behavior is missing. If so, make a comment.
1. Then read the test bodies. If they're testing all over the place, make a comment.
1. If they're not testing the right thing, make a comment.
1. Proceed to review the code under test. This is now pretty easy if you assume that the tests actually run. At this point you can mainly focus on style, efficiency, and things like that.
