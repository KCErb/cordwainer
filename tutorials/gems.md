---
layout: page
title: 2. Gem Install
category: executables
permalink: /gems/
---

In [the previous tutorial](http://kcerb.github.io/cordwainer/start/) we learned everything there was to know about the
simplest Shoes program I could think of:

```ruby
puts "Hello World"
```

which we run as

```
$ bin/shoes testing/hello.rb
```

That walked us quite a bit through the *development* part of getting Shoes fired up but in the process of doing so, we kept bumping into comments [like these](https://github.com/shoes/shoes4/blob/master/bin/shoes#L3-L4):

```ruby
# This is NOT the primary shoes that's installed--just a helper for local
# development purposes
```

So I say, while the issue is fresh in mind, and before we really dive into Shoes itself, let's find out what those comments are really about and find out what happens when we run `hello.rb` with the installed executable like so:

```
$ shoes testing/hello.rb
```

This exploration will take us through the multi-gem structure of Shoes. When I wrote this I'd never cut a gem before or even read a tutorial on it. Since then I've done both of those and found there's a lot of overlap with what I have here and some great tutorials out there on gems. So I encourage you to google around for advice on writing Ruby gems to see where some of this comes from (hint `Bundler` does *a lot* of stuff automatically).

## shoes --pre3

So first of all the `shoes` command-line app runner is only available if we

```
$ gem install shoes --pre
```

So let's start finding out what happens there.

## The shoes Gem

Although Shoes contains three gems right now (`core`, `swt`, and `package`) it is also a gem itself (albeit just a tiny little one). Let's take a look at the [`gemspec`]() for the `shoes` gem:

```ruby
# -*- encoding: utf-8 -*-
version = File.read(File.expand_path('../VERSION', __FILE__)).strip

Gem::Specification.new do |s|
  s.name        = "shoes"
  s.version     = version
  s.platform    = Gem::Platform::RUBY
  s.authors     = ["Team Shoes"]
  s.email       = ["shoes@librelist.com"]
  s.homepage    = "https://github.com/shoes/shoes4"
  s.summary     = 'Shoes is the best little GUI toolkit for Ruby. Shoes runs on JRuby only for now.'
  s.description = 'Shoes is the best little GUI toolkit for Ruby. Shoes makes building for Mac, Windows, and Linux super simple. Shoes runs on JRuby only for now.'
  s.license     = 'MIT'

  s.files         = ["LICENSE", "README.md"]

  s.add_dependency "shoes-core", version
  s.add_dependency "shoes-swt",  version
  s.add_dependency "shoes-manual", "~> 4.0.0", ">= 4.0.0"

  # shoes executables are actually installed from shoes-core
end
```

This bit of code starts off by pulling a version string in from the top-level `VERSION` file. Next it does typical `gemspec` things such as defining the name of the gem, the authors, homepage etc. and then we get to `s.files`. This is the part of the spec that tells us which files should be included in the gem. Since the `shoes` gem is really just a home for all of its parts, it only includes the `README` and `LICENSE` and leaves the rest of the installation to the other gems which are referenced with the `add_dependency` method. Let's start with `shoes-core`.

## The shoes-core Gem

If you take a look at [`shoes-core.gemspec`](https://github.com/shoes/shoes4/blob/master/shoes-core/shoes-core.gemspec) you'll see a lot in common with `shoes.gemspec` when it comes to authors, version, etc. But there's some new stuff here too

```ruby
Gem::Specification.new do |s|
  # ...
  s.files         = `git ls-files`.split($INPUT_RECORD_SEPARATOR)
  s.test_files    = s.files.grep(%r{^(test|spec|features)/})
  s.require_paths = ["lib"]

  # Curious why we don't install shoes? See ext/Rakefile for the nitty-gritty.
  s.executables   = ['shoes-picker', 'shoes-stub']
  s.extensions    = ['ext/install/Rakefile']
end
```

First of all, we point the `files` method at a list of ALL THE FILES by passing to `git ls-files` a reference to `/` or `\` depending on your OS. Then we specify which ones are `test_files` (which seems to be a feature on its way out; see [this](http://stackoverflow.com/questions/18871541/what-is-the-purpose-of-test-files-configuration-in-a-gemspec) stackoverflow discussion) and then we add the `lib` path to the `$LOAD_PATH` when the gem is activated.

These things are pretty standard and can be found in the other Shoes gems.

Next we specify some executables that we'd like Rubygems to make available from the command-line and we notice that `shoes` is quite absent, instead the familiar `shoes-stub` and `shoes-picker` are specified and we are left with only one thing to dig into.

One thing that has been mentioned and avoided TWO times in these tutorials.

One thing that we've been putting off but can no longer put off.

*The* **nitty-gritty**: `ext/Rakefile`

### The Rakefile that answered some questions

The file's not all that bad. It's really pretty short. Go ahead and [take a look](https://github.com/shoes/shoes4/blob/master/shoes-core/ext/install/Rakefile).

The core problem this file solves is [a policy of RubyGems](http://guides.rubygems.org/specification-reference/#executables) about executables.

> These files must be executable Ruby files. Files that use bash or other interpreters will not work.
Executables included may only be ruby scripts, not scripts for other languages or compiled binaries.

That's normally no big deal, except we need to pass in that JRuby option (start on first thread) if we're running on OS X, and that means we need the executable to be a shell script, *not* a ruby script. What to do?

Well RubyGems isn't against running a shell script called `shoes`, it's just against installing it via the `executables` command. So this [Rakefile](https://github.com/shoes/shoes4/blob/master/shoes-core/ext/install/Rakefile) gets run on `gem install` (that's what extensions are for) and all it does is figure out if we need the `shoes.bat` or `shoes-stub` and then copies the non-ruby executable to the RubyGems executables directory.

The result: I call `shoes` from the command line, RubyGems responds with a copy of `shoes-stub` (or `shoes.bat` if you're on windows) and away we go!

## Wrap up

Well that answered how `$ shoes testing/hello.rb` works, let's wrap up by finishing our tour of `shoes.gemspec` in the `shoes-swt` and `shoes-manual` gems.

### shoes-swt

The only thing new over in [`shoes-swt.gemspec`](https://github.com/shoes/shoes4/blob/master/shoes-swt/shoes-swt.gemspec) is the dependencies on `swt`, `after-do`, `shoes-core`, and `shoes-package`.

* [`swt`](https://github.com/danlucraft/swt) is the gem that let's JRuby talk to [SWT](https://www.eclipse.org/swt/) (more on SWT later)
* [`after-do`](https://github.com/PragTob/after_do) is a gem written by our own [@PragTob](https://github.com/PragTob) that allows code to be called *after* a method gets called (more on that later too)
* [`shoes-package`](https://github.com/shoes/shoes4/tree/master/shoes-package) is the gem that handles packaging (again, more on that in a different lesson).

### shoes-manual

The [`shoes-manual`](https://github.com/shoes/shoes-manual) gem is *not* in the primary `shoes4` directory. That's because we want it to stand independent of how it is read. Right now both Shoes and the Shoes website have copies of the manual and it'd be nice to see them both working from the same thing, hence, the manual gem.

Right now the manual (like the packaging) are things that we'd **love** to see working, but know we can't really get humming along until Shoes is on a more solid footing. So this gem (for now) is sorta sitting by, waiting for love.
