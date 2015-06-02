---
layout: page
title: 2. App
category: tdt-core
permalink: /tdt-core/app/
---

OK, we are now finally ready for the next level of complexity in a Shoes app!

```ruby
Shoes.app do
  puts 'Hello World'
end
```

The app method gets defined for the `Shoes` class [right off the bat](https://github.com/shoes/shoes4/tree/master/shoes-core/lib/shoes/app.rb#L18-L20).
It accepts options and a block to (`initialize`) and is the first `subject` we assign for testing.
Defining a subject like this is nice for when we'd like to test the same object a few different ways or
test several attributes on the same object. In this case we passed to the `subject` method an optional parameter `:app`
so that we would have a name to call our subject by in addition to the default `subject`.

Before moving on to testing this subject, we politely ask `RSpec` to unregister all apps after each test.
This is important because during the `initialize`, the app gets registered with a master
list that knows about all the Shoes apps that are running via the [registration module](https://github.com/shoes/shoes4/tree/master/shoes-core/lib/shoes/common/registration.rb) (more on this later)
which gets pulled in by `DSL`.

And it is this `DSL` module that we spec out in great detail [here](https://github.com/shoes/shoes4/tree/master/shoes-core/spec/shoes/app_spec.rb#L12) which pulls in all of the [dsl tests](https://github.com/shoes/shoes4/tree/master/shoes-core/spec/shoes/shared_examples). We'll save those for a different tutorial.

These DSL tests and the next few lines:

```ruby
it { is_expected.to respond_to :clipboard }
it { is_expected.to respond_to :clipboard= }
it { is_expected.to respond_to :owner }
```


use the `it` syntax where the previously assigned `subject` is assumed to be the object that is expected to respond to "`#a_method`".
This syntax is made available by the `rspec/its` require we did back in `spec_helper`.

Before moving on to spec'ing out individual methods (like `initialize`), we check that `it` has the `app` method which Shoes 3 provided as an alias for `self` like so

```ruby
it "exposes self as #app" do
  expect(app.app).to eq(app)
end
```

This test demonstrates a core idea in RSpec:

* describe an expected behavior in words
* write a test using the `expect` syntax
* keep it simple

Ideally in test-driven development, we would first write a simple test like this down. Then we'd go back to the codebase and implement the expected behavior.

Looking back to the [`App` class](https://github.com/shoes/shoes4/tree/master/shoes-core/lib/shoes/app.rb),
we can see that one of the first things the author did in `Shoes::App` was define this alias.

## Initialize

With some basic app tests walked through, we're now ready to dig into individual methods that a `Shoes::App` should have.
To do this here, we decided to use a new `describe` block. In RSpec, we use `describe` and `context` to organize the code.

For example, an `App` may be initialized with no options or with a bunch of options passed in to the `opts` hash. So we want to `describe`
how the `App`'s initialize should behave in both `contexts`.

In the defaults context, we first test to see if `Shoes::App` is getting its properties from the internal app. So there are two questions
I want to answer about this: why do we do this, and how do we do it?

### Why

We separate App from internal app because `App` is the "user-facing app". As we'll see later, a part of the magic of Shoes is that
whatever you put into the `do` `end` block gets evaluated in the context of `Shoes::App`. So let's say for example that instead of using an internal app like we've done here, we put it directly on the app:

```ruby
class Shoes
  class App
    attr_accessor :width
  end
end
```

Then what would happen if a user used Shoes like this:

```ruby
Shoes.app do
  width = 50
  rect 0, 0, width
end
```

Instead of declaring a variable called width, that user would have just set the width of the app itself to 50!

We would much prefer if the context in this block was such that the user could write any kind of variable they want for their own program and not worry about what we did behind the scenes.

### How

We accomplish this little trick firstly by passing the block and opts onto a new class called `InternalApp` during the initialize. It's over there that we'll see how the block gets called, and the attributes get set up from the opts.

The next thing we do is use `def_delegators` (from the [`Forwardable`](http://ruby-doc.org/stdlib-2.0/libdoc/forwardable/rdoc/Forwardable.html) module) to give `App` a list of methods to pass on to `@__app__` instead of responding to them directly. To get an idea of what this does, check out the following app

```ruby
Shoes.app do
  width = 50
  rect 0, 0, width
  alert width
  alert self.width
end
```

Since the method `width=` isn't delegated, a local variable called width gets set and used in the first two lines. But the app still holds onto its own width and that can be accessed via `self.width`. That's why specs looking at width, height, etc are checked against a constant that `InternalApp` holds onto instead of `App`.


### Inspect

The last thing we look at in this context is what inspect returns. One of the purposes of Shoes is to be newcomer friendly and one
way to do that is to get nice feedback from `to_s`, and `inspect`. Therefore we include the `Common::Inspect` module and test its output with the known default title.

For [this module](https://github.com/shoes/shoes4/blob/master/shoes-core/lib/shoes/common/inspect.rb) we've included some empty methods to be overwritten like `inspect_details` and `to_s_details`. They are implemented in `Shoes::App` right [here](https://github.com/shoes/shoes4/tree/master/shoes-core/lib/shoes/app.rb#L102-L108). And you can see that for `to_s` we expect to get back the title of the app and for `inspect` we want the title *and* an object-id (note that the method `shoes_object_id_pattern` is provided by the `InspectHelpers` module).

### From Opts

This section does what's expected, it checks out that the passed in opts are actually being returned from `InternalApp`, but then it looks at making sure things get set up correctly. Namely, it verifies that InternalApp receives an App and opts when getting initialized with a call to `new`. A similar verification checks to see that the main `Flow` of the app gets set up correctly.

This kind of expectation used in these two tests is different than the ones we've seen so far because the code that's being checked actually happens on the following line with the call to `subject`. When you ask RSPect to `expect` an object to `receive` a method call, you are stubbing out that method so that it doesn't actually get called, plugging `and_call_original` on the end makes sure that the actual method does get called. So the idea is that if you want to watch an object (not an instance of an object) for certain calls, you should *mock* it by stubbing out the method you want to call, and then pass the work back on to the actual object so that we're testing the actual codebase as much as possible. [Here](https://github.com/rspec/rspec-mocks#delegating-to-the-original-implementation)'s a little more info on the topic.

### When Registering

The last thing to check during the `initialize` is registration. As discussed a bit ago, Shoes contains a list of all apps that are running so during initialization of the app this app needs to be added to the list. Now, if you scroll to the end of the specs, you'll see that there are more registration specs there. The purpose for splitting registration up like this, is that here we are still focused on initialization. That means we want to focus on the work that gets done by initialization here and nothing more; however, since registration is just complex enough to deserve its own module, we also want to spec it out separately. Hence, we give it its own section at the bottom of the specs.

### Styles

The `Style` module will need its own tutorial, so I won't go deeply into what is being tested here, and these styles actually belong to the `InternalApp` so you won't see much in `app.rb` about this. But it is nevertheless important to make sure these things are all available from the `Shoes.app` block. So I will just make a few points for these tests

*  These tests are testing the `style` method which is avaialble in the block thanks to [DSL](https://github.com/shoes/shoes4/tree/master/shoes-core/lib/shoes/dsl.rb#L167-L174)
* We ensure that any styles set for one app are not also set in other apps
* We check that styles can be set and passed on to objects inside an app.

So please take a look at these and hold off detailed questions for when we look at the `style` method of DSL and the [`Style`](https://github.com/shoes/shoes4/blob/master/shoes-core/lib/shoes/common/style.rb) module.

### GUI

Next we check that `@__app__` (the `InternalApp`) can talk to the `gui` (the backend) and perform some basic functions.

In the first test, we want to make sure that when the `app` is asked to do something with `clipboard` that call gets passed back to the gui.
Here's how that works: `clipboard` is a method provided by the [DSL](https://github.com/shoes/shoes4/tree/master/shoes-core/lib/shoes/dsl.rb#L609-L611), which directs a call to `@__app__`. A quick look inside `InternalApp` [shows]() that `clipboard` is forwarded on to gui as expected.

The pattern of putting work that the gui *should* do (clipboard, quitting, fullscreen etc) on the backend is a common pattern in Shoes design. And it's an important distinction to pay attention to throughout these lessons because its not always so clearly cut! Even here we're essentially testing gui methods from the frontend. Getting these two things clearly separated is a primary goal of excellent shoes development.

<hr>

## Moving Forward

At this point, I'll stop explaining each test. Hopefully some of the guidance and explanation above makes it easier to understand how a lot of RSpec tests work, where to look for methods not found directly in `app.rb`, and how to navigate around. From here out, I'd like to focus on tests that look at code found *directly* in the `app.rb` file and only stray to explain new types of tests or large groups of related tests.

### Additional Context

The next major section of the `App` and for its tests is the [handling of contexts](https://github.com/shoes/shoes4/blob/master/shoes-core/lib/shoes/app.rb#L85-L98). Let's start again in the tests to see what we hope to get out of this "context" stuff.

The tests are telling us that we expect `App` to throw and error when given an unknown method call unless something else (an `additional_context`) is provided that might understand the call. To dig into why a tool like this was created, I searched on github for the method name (`eval_with_additional_context`) and found [this](https://github.com/shoes/shoes4/pull/861) pull request on the matter. So this method is provided in app so that Shape can use it [here](https://github.com/shoes/shoes4/blob/master/shoes-core/lib/shoes/shape.rb#L28);

Now you can reread that code and [the tests](https://github.com/shoes/shoes4/blob/master/shoes-core/spec/shoes/app_spec.rb#L312-L347) with the idea in mind that Shape is evaluating `self` as the context so that methods which Shape understands (like `arc_to`) are available inside of the `Shape`  do block.

It's also useful to point out the use of `double` here.

We don't want to actually create an execution context with an `asdf` method, so instead, we create a test double called `context`
and tell RSpec that we want it to respond to the `asdf` method by returning `nil`

### Subscribing to DSL Methods

The last thing we need to cover in this tour through `Shoes::App` is [DSL method subscription](https://github.com/shoes/shoes4/blob/master/shoes-core/lib/shoes/app.rb#L110-L127). As before, we want to read the specs and look at how it's used before looking at how a solution was implemented.

For this test, we want to make sure objects that "subscribe" to the DSL have certain behaviors. So instad of testing those objects (URL, Widget) we create a test class. The idea is that we want to separate the *idea* of subscription from the implementation. Test classes like these are great for that!

The tests tell us that an object which has subscribed to DSL methods should have access to a specific sub-list of these methods and pass them on to the app for handling. It also provides a way that a new method can be added to the dsl (i.e. through a widget) and the subscription will still work (so one widget is available from another).

The problem is solved by creating a class method on `Shoes::App` which first extends `Forwardable` to the class and then uses our old friend `def_delegators` to forward a list of methods to the app. In addition, a list of classes that want a DSL subscription is kept so that any methods added in the future can get the same Forwardable treatment. Nice!

### Wrap-up

So we took a quick tour through app. We saw that the heavy lifting is spread between App and InternalApp, along with  DSL and some `Common` modules. We also learned a bit about how testing with RSpec works and some good examples of well-written tests. I think the next place to go, while App is fresh in mind would be `InternalApp` and `DSL`. These two classes will continue the theme of gently maneuvering through some of the tricky spots that make Shoes tough to build, but easy to use.
