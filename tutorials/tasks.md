---
layout: page
title: 3. The Tasks
category: executables
permalink: /tasks/
---

In the [previous](http://kcerb.github.io/cordwainer/start/) [tutorials](http://kcerb.github.io/cordwainer/gems/) we learned how to run a Shoes program from both the development executable `bin/shoes` and the gem executable `shoes`. In both cases we end up in the command-line interface (CLI) and [this line](https://github.com/shoes/shoes4/blob/master/shoes-core/lib/shoes/ui/cli.rb#L58) runs the app!

I'd like to dig into a real app, frontend, backend etc but first let's wrap up this section on Shoes executables by talking about the rake tasks.

## Rakefile

Let's take a look at [the Rakefile](https://github.com/shoes/shoes4/blob/master/Rakefile).

If you're not familiar with Rakefiles then please take a look at the [Rake's README](https://github.com/ruby/rake). If you're not familiar with `rake/clean` [these comments](https://github.com/ruby/rake/blob/master/lib/rake/clean.rb#L1-L12) should be enough for our purposes. And lastly you should know that `rspec/core/rake_task` pulls in some nice rake tasks from RSpec that we'll be using later on.

OK now, looking on in the file, we see a bunch of files from the `tasks` directory getting pulled in. Let's go through those one by one.

### Changelog

[`changelog`](https://github.com/shoes/shoes4/blob/master/tasks/changelog.rb) is a neat little task that [@wasnotrice](https://github.com/wasnotrice) cooked up for keeping track of changes between versions.

We want Shoes developers to tag every commit with a `Changelog` tag (see [this wiki page](https://github.com/shoes/shoes4/wiki/Merging-a-Pull-Request)). The purpose is so that at each release we can automatically have a little log of the changes. For an example of what the sum of all these little `Changelog` tags outputs take a look at [the Changelog](http://shoesrb.com/2015/01/04/shoes-4pre3-release.html) that was blogged after pre-release 3.

So how does it work? Well the code starts out in `Changelog#generate` which finds the most recently tagged commit (i.e. the last release, take a look at `git tag` if, like me, you're new to this) and stores a range string i.e. "v4.0.0.pre3..master"

<hr>
**Sidenote**
From [this StackOverflow answer](http://stackoverflow.com/a/24186641/3221576) I learned that there is a bit more to a `commit range` than meets the eye. The upshot is that technically the syntax `a..b` means all commits in `b` which aren't in `a`. Assuming a nice linear set of commits on `master` this just amounts to a simple range of commits, but I thought it was interesting that a `git` range may be non-linear.
<hr>

#### changelog_header

This method generates the header which states how many commits occurred between releases. It makes use of `git log`, `git rev-list` and `git describe`. I'll show how `git` commands like these work over in `categorize_commits` in detail since that method is a bit more complicated. But the gist of it is that with `git` commands we can do something like list out commits, count them, and get simple descriptive names.

#### categorize_commits

Next `categorize_commits` goes through each hash in the `CATEGORY_MAPPING` array, and looks for commits (searched for via `git log`) that contain a certain `:pattern`. The key to understanding this is seeing how `log_command` is built [here](https://github.com/shoes/shoes4/blob/master/tasks/changelog.rb#L50-L54). An example of a fully built command is:

```
git log --regexp-ignore-case --grep 'Changelog: improvement' --format='%s<--BODY-START-->%b<--BODY-END--> [%h]<--COMMIT-->' v4.0.0.pre3..master
```

The result is a nicely formatted version of `git log` that delineates the body of the message like so:

```
Merge pull request #1028 from shoes/hidden-style<--BODY-START-->Support the hidden style on all elements

* Changelog: improvement<--BODY-END--> [63afeca]<--COMMIT-->
Merge branch 'master' into move-default-args<--BODY-START-->Move defaulting out of dsl.rb into DSL classes

Fixes #968

Changelog: improvement
<--BODY-END--> [81092b9]<--COMMIT-->
Merge pull request #1032 from shoes/text-block-painter-guard-636<--BODY-START-->Move guard against empty text segments further up in the chain

Changelog: improvement<--BODY-END--> [f121cac]<--COMMIT-->
Merge pull request . . .
```

After some formatting touches supplied by `uniform_change_log` we get two outputs: the commits which are put into `categorized_commits` which look like this for my sample:

```
["* Support the hidden style on all elements\r [63afeca]", "* Merge branch 'master' into move-default-args [81092b9]", "* Move guard against empty text segments further up in the chain\r [f121cac]"
```
and `change` which is a formatted version produced by `changes_under_heading` that looks like this for the improvement changes I'm demonstrating

```
[nil, "Improvements (5)\n----------------\n\n* Support the hidden style on all elements\r [63afeca]\n* Merge branch 'master' into move-default-args [81092b9]\n* Move guard against empty text segments further up in the chain\r [f121cac]\n . . .
```

Lastly the `categorized_commits` method hands back `change` after adding one more category "Miscellaneous" which is the difference between the `categorized_commits` array and the `changes` variable, just a catch all for leftover changelog entries.

#### Contributors

Next the contributors are extracted via `git shortlog`. Here's an example input and output

**input**
```
git shortlog --numbered --summary v4.0.0.pre3..master
```

**output**
```
48  Jason R. Clark
19  Tobias Pfeiffer
12  KC Erb
 6  Eric Watson
 5  Thomas Graves
 2  David English
 1  bx10000
```
This output gets put under a new heading via the same heading method from before and before the string is handed back it is `compact`ed (to remove `nil`s) and joined.

And that's it for the `changelog`. I know its kind of a quick run through but hopefully its enough to get you familiar with the basic ideas.

### Console

[`console`](https://github.com/shoes/shoes4/blob/master/tasks/console.rb) is a little rake task just for running a pry session with Shoes loaded. Such a great thing! If you don't know why that is great then you might not know what `pry` is and should [check it out](http://pryrepl.org/)! Also you should watch [this](http://www.confreaks.com/videos/3365-railsconf-debugger-driven-developement-with-pry) great RailsConf talk on DDD - Debugger Driven Development. Very cool things!

### Gem

[`gem`](https://github.com/shoes/shoes4/blob/master/tasks/gem.rb) is a set of tasks for building and releasing gems.

<hr>
**Sidenote**:
When I first came across this file, I'd never built or released a gem before and had a lot of questions. So I'm going to jot down the things I learned here in a bit more detail than the average ruby developer would need. The source for much of my commentary is [this discussion](https://github.com/shoes/shoes4/issues/1042) I had with other cordwainers.
<hr>

#### install build release

The first line of `gem.rb` pulls in some standard rake tasks that bundler provides: `install`, `build`, and `release`. You can find the definition of these tasks [here](https://github.com/bundler/bundler/blob/master/lib/bundler/gem_helper.rb#L34-L66). `install` installs a built gem and `release` puts the built gem up on [rubygems](https://rubygems.org/) so the important question is: what does `build` do?

I like the way [@PragTob](https://github.com/PragTob) put it:

>Build takes the gemspec (or `build:all` takes all gemspecs) and packages up the gems according to them, i.e. it produces a `.gem` file that contains all the files - so that file is an archive, zip or something. This file can then be distributed. You can try it! run `rake build:all` and it should create a bunch of files in the `pkg` directory. You could then do `gem install pkg/gem_name.gem` to install the gem.

#### empty tasks

The next few lines contain task declarations with no blocks attached. For example:

```ruby
desc 'Build all gems'
task 'build:all'
```

These tasks are getting initialized so that we can attach other tasks to them.

`shoes` is a meta-gem, it's just a container for the actual Shoes gems `shoes-core` etc. So we attach the actual work of building, installing etc. of each gem to an encompassing `build:all`. That's what the `each` loop in [the next few lines](https://github.com/shoes/shoes4/blob/master/tasks/gem.rb#L15-L37) is doing.

The following [lines](https://github.com/shoes/shoes4/blob/master/tasks/gem.rb#L39-L47) handle the pure `shoes` meta-gem. This one gets special attention because the default rake `build` task (and others) is already set-up to work as expected, so there's no need to `cd` into the directory and then run `rake build`.

One more thing about those empty tasks. The correct way to describe a line like this:

```ruby
task "build:all" => "build:shoes"
```

is

>when `build:shoes` gets *invoked* (in Rake's terminology), then *before* you execute the `build:shoes` task, make sure that the `build` task has been run. Then run `build:shoes`.

Thanks to [@wasnotrice](https://github.com/wasnotrice) for [explaining this](https://github.com/shoes/shoes4/issues/1042#issuecomment-72544807) to me!

#### update_versions

Lastly, each of the gems has a `version.rb` file that looks [like this](https://github.com/shoes/shoes4/blob/master/shoes-core/lib/shoes/version.rb):

```ruby
class Shoes
  VERSION = "4.0.0.pre3"
end
```

it would be a hassle to update all of these by hand, so this task will update these based on the root [`VERSION`]((https://github.com/shoes/shoes4/blob/master/VERSION)) file.

OK, so that does it for the `gem` tasks.

### Sample

[`sample`](https://github.com/shoes/shoes4/blob/master/tasks/sample.rb) provides some tasks under the namespace `samples`. If like me you're not familiar with Rake's `namespace` feature, how about you skip down and take a look at the [last three lines](https://github.com/shoes/shoes4/blob/master/tasks/sample.rb#L66-L68). Defining tasks like `samples:random` is made possible by defining them in the `namespace` block.

In this file, the block gathers up all the samples from the `SAMPLES_DIR` and puts them into two categories: those that are listed as working in `samples/README` and those that aren't. Depending on which ones you want to run (go ahead and look around at all the options), the code here calls the samples one at a time via . . .

```ruby
def run_sample(sample_name, index, total)
   puts "Running #{sample_name} (#{index + 1} of #{total})...quit to run next sample"
   system "bin/shoes #{sample_name}"
 end
```

yup, `bin/shoes`.

### Yard

And last but not least (before moving onto rspec that is) the [yard task](https://github.com/shoes/shoes4/blob/master/tasks/yard.rb) uses [the `yard` gem](http://yardoc.org/) to auto-generate documentation. To my knowledge we're not actively using yard-based documentation so I'm not going to dig into an "all about yard" aside here. Let me know if you'd rather I did though :)

## RSpec

If you're totally new to RSpec go on over to [the website](http://rspec.info/) and take a look. (Also, I would recommend checking out the [specs section](https://github.com/shoes/shoes4#running-specs) of the README if you haven't seen it.) The purpose of [this task](https://github.com/shoes/shoes4/blob/master/tasks/rspec.rb) is to get our tests up and running, but it's actually a pretty complicated job. Let's dig in and find out why.

### A little about JRuby and Rake

The first thing we do is set the `runRubyInProcess` configuration variable of the *currently running* JRuby runtime to `false`. We do this because we're going to launch the specs in a sub-jruby and if we don't, the current VM will ignore arguments given to the second one, and we will definitely want to be able to pass arguments in! For (a little) more info see this [JRuby wiki page](https://github.com/jruby/jruby/wiki/FAQs#how-can-i-increase-the-heapmemory-size-when-launching-a-sub-jruby).

After some method definitions, we define the default task (invoked by `$ rake`) to be this one (on [line 47](https://github.com/shoes/shoes4/blob/master/tasks/rspec.rb#L47)). Then we do something new: pass in `[:module] => "spec:all"` as an argument to the task method. This syntax let's us pass arguments into the spec. This is what lets us just run the `Shape` specs as described in [the README](https://github.com/shoes/shoes4#running-specs). To get a feel for this try running the following file named `Rakefile`:

```ruby
task default: :spec

task :spec, [:module] => "spec:all"

namespace :spec do
  task "all", [:module] do |t, args|
    puts t
    puts args
  end
end
```

by `$ rake spec:all[Shape] --trace` you should get back:

```
** Invoke spec:all (first_time)
** Execute spec:all
spec:all
{:module=>"Shape"}
```

We'll be using this kind of syntax a lot here so it's good to keep in mind what's going on. The `all` task of `spec`, for example, invokes each of the `shoes`, `swt`, and `package` specs and passes to each one a `String` identifying the DSL element that's to be tested.

The `spec` namespace fundamentally has 3 tasks: `swt` (which has 3 of it's own subtasks), `core` (which is aliased as both `shoes` and `dsl`), and `package`.

### Integration and Isolation, Frontend and Backend

One of the key things to know about Shoes is that it is a DSL for writing GUIs. The DSL is small and powerful and as a cordwainer it should be your bread and butter. The goal in creating Shoes is to implement that DSL on a bunch of platforms: Windows, Linux and Mac OS X to name a few. The pure DSL elements (`Flow`, `Stack`, `Button`, etc.) with no actual implementation are the **frontend**. The actual nuts and bolts that convert syntax like `para "Hi"` into text on the screen is the backend (which is sometimes called the gui). Right now Shoes development is focused on one backend: SWT. But we'd like to support many backends. For example someday there may be a backend for mobile devices.

Both the `shoes-core` and `shoes-swt` directories contain a set of specs. The `shoes-core` specs are supposed to spec out the frontend, and the `shoes-swt` specs are supposed to spec out the SWT backend. You might think we would just call each of these specs in isolation, but the trouble is the front-end *needs* a backend. So it can be called with either the Mock backend, or the SWT backend. The Mock backend is meant to be minimal so that the `shoes-core` specs are in isolation when they run, but it's important to remember that these are still specs that **integrate** the DSL (frontend) *with* a backend.

What this boils down to is that the frontend specs can be run two ways: with the Mock backend *or* the SWT backend. The backend specs can only be run one way. Therefore we call the frontend specs integration-specs and the backend specs isolation-specs.

### The :swt namespace

The [`:swt`](https://github.com/shoes/shoes4/blob/master/tasks/rspec.rb#L72-L95) namespace can run the frontend (isolation) specs, the backend (integration) specs, or both. Hence the three tasks: `all`, `isolation` and `integration`.

All three of these specs run essentially the same way:
* `swt_args(args)`
* stick all of the appropriate file paths into an array
* `jruby_rspec(files, args)`

So let's take a look at the first and last methods defined above

First `swt_args` sets up a hash of arguments to be passed to the specs called `argh`. If an argument was passed in, it is attached to the `:module` key. Here's an example `argh` produced on my system by running

```
$ rake spec:swt[Shape]
```

`argh` will be

```ruby
{:module=>"Shape", :swt=>true, :require=>"shoes-swt/spec/spec_helper", :excludes=>[:no_swt, :fails_on_osx]}
```

Next, `jruby_rspec` takes the `:swt` key and creates an `rspec_opts` from the remaining keys with appropriate mappings (such as `:require` to `-r`) the resultant `rspec_opts` output from the hash above is:

```
-e ::Shape -rshoes-swt/spec/spec_helper --tag ~no_swt --tag ~fails_on_osx
```

Next this gets combined with the files array into an RSpec command via the `rspec` method. The output is

```
rspec --tty -e ::Shape -rshoes-swt/spec/spec_helper --tag ~no_swt --tag ~fails_on_osx  shoes-swt/spec/shoes/cli_spec.rb shoes-swt/spec/shoes/swt/animation_spec.rb shoes-swt/spec/shoes/swt/app_spec.rb shoes-swt/spec/shoes/swt/arc_spec.rb ... a/bunch/more/files
```

Finally `jruby_run` runs the command with a new `jruby` instance passing in the command (and the start on first thread option if applicable).

### spec:swt:integration vs spec:core

The primary difference between the way the `swt` specs run the `shoes-core` specs and the way `core` runs them is which `spec_helper` file gets included. After that the rest is the same. The next tutorial will dig into what these `spec_helper` files do and how they specify which backend to test the frontend with so I'll leave off on that for now.

## Conclusion

So we took a quick spin through a lot of code that supports getting Shoes up and running in a development environment, with an app given, and for testing.

Since these tutorials are for Shoes developers and Shoes is meant to be some kind of TDD / BDD / DDD, the next set of tutorials will walk through the Shoes frontend class by class by looking at how these specs desribe it. 
