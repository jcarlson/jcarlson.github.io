---
title: Thoughts on Test-Driven Development
---

Having recently spent time interviewing with numerous development shops, it's clear that Test-Driven Development (TDD) is quite the rage right now. I've found this especially true of the larger, more enterprisey organizations that I've interviewed with.

For the purposes of this article, I'll assume you (the reader) have at least some idea what TDD is about. If you don't, allow me to briefly summarize: write a failing unit test (red), implement the code that resolves the failing test (green), refactor the code until satisfied (refactor) and repeat. There are plenty of great resources just a quick Google search away; I'm partial to definitions offered by [Martin Fowler](http://martinfowler.com/bliki/TestDrivenDevelopment.html).

In theory, TDD makes your code more robust and less bug-ridden. But there are plenty of criticisms of TDD, and even the aforementioned [Martin Fowler has them](http://martinfowler.com/articles/is-tdd-dead/).

I'll admit, I don't always write my tests first, and I'm not the only one. Neither does [David Heinemeier-Hansson](http://david.heinemeierhansson.com/2014/tdd-is-dead-long-live-testing.html).

It's not that I think writing tests first is a _bad_ practice. But I do think that writing software is a bit of an art form, and in the early phases of a new piece of functionality, a lot changes and quickly.

When I build software, I spend a lot of time in that _refactor_ phase, and I iterate on designs to improve clarity. But this can only happen because I first hammered out the rough shape of my ideas in the form of [prototype] code.

I often find I don't really know what I want to write until I start writing it, and only then do real, substantive patterns and designs begin to emerge. Writing detailed unit-level tests of the code at this early stage would be detrimental to the refactor cycle that will likely replace or grossly modify the first draft of code.

The only sorts of tests that might reasonably be ready to write first, at the outset, are the sort of high-level business facing tests. Something like a Cucumber test, such as:

```
Given I am not logged in
When I visit click on the 'profile' link
Then I am redirected to the login page
When I login as 'user' with password 'secret'
Then I am redirected to the profile page
```

In the example above, we're clearly embarking on a new feature set to implement some kind of authentication mechanism for a new application. But authentication is a complex and time-consuming feature to implement (at least, it can be). And this test offers little detail about how the underlying system actually works.

True, most tests should avoid knowledge of implementation details, but as we flesh out the requirements and functionality (for example, adding "Login with Facebook"), this test becomes increasingly generic and loses value quickly. It will have to continuously be refined alongside our application code. Worse, until authentication is completely implemented, this test will continue to fail, making the red-green-refactor feedback loop either very long or very useless.

Rather than relying on strict adherence to the TDD principles, I prefer to focus on honing my skills in another area: identifying the necessary tests that _should_ exist, given a block of code or feature. Let's look at an example:

```coffeescript
update = -> (event)
  event.preventDefault()
  @item.fromForm(event.target)
  if @item.save()
    @navigate $(event.target).attr('action')
```

This snippet comes from an old SpineJS controller I wrote years ago, and reacts to the user clicking the 'Save' button on an "update profile" sort of form. Lets go line by line.

1. `event.preventDefault()` - the function should stop the default browser behavior
2. `this.item.fromForm(event.target)` - the function should assign attributes to the model from the form object
3. `if (this.item.save())` - we have a conditional branch with two outcomes
  * `save()` is `true` - the function should navigate the user to another screen
  * `save()` is `false` - the function should not navigate the user

We've pretty much laid out all the test cases that need to exist, based on what this function is designed to do, and we can convert that right into a set of Jasmine tests, complete with nested contexts.

```coffeescript
describe 'submit form', ->
  beforeEach ->
    # some bits omitted for clarity
    @client.save.andReturn(true)

  it 'prevents default action', ->
    spyOnEvent('form', 'submit')
    @$('form').submit()
    expect('submit').toHaveBeenPreventedOn('form')

  it 'applies form data to client record', ->
    @$('form').submit()
    expect(@client.fromForm).toHaveBeenCalled()

  it 'saves the client record', ->
    @$('form').submit()
    expect(@client.save).toHaveBeenCalled()

  describe 'when client record is valid', ->
    it 'navigates to client detail view', ->
      @$('form').submit()
      expect(Spine.Route.navigate).toHaveBeenCalledWith('/clients')

  describe 'when client record is invalid', ->
    beforeEach ->
      @client.save.andReturn(false)

    it 'does not navigates to client detail view', ->
      @$('form').submit()
      expect( Spine.Route.navigate ).not.toHaveBeenCalled()
```

There's a pretty big takeaway that I should point out explicitly here: anytime you have conditionals in your code, you are almost guaranteed to need nested contexts in your tests. Forcing this mantra upon my testing practice very quickly got me in the habit of avoiding complex, nested conditionals. They're ugly to read and hard to test, given the quickly multiplying permutations of decision trees that can arise from nesting if/then/else clauses.

The lesson here is that I don't think it's critical that you write your tests first, but I do think it's critical that you have a complete test suite. It's also important that your test suite is as unassuming as possible about the implementation of your code. Keep your code's surface area small, and your tests focused on just that surface area, and you'll be fine.
