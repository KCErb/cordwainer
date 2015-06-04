---
layout: page
title: Setup
order: 1
category: tdt-core
permalink: /tdt-core/setup/
---

With a terminal sitting in my `shoes4` repository, if I run `rake spec`, the following command will be created and executed

```
jruby --debug --1.9 -Ispec  -S rspec --tty -rshoes-core/spec/spec_helper  
shoes-core/spec/shoes/animation_spec.rb
shoes-core/spec/shoes/app_spec.rb
shoes-core/spec/shoes/arc_spec.rb
shoes-core/spec/shoes/background_spec.rb
...
```

So before we can start going through each of the specs, let's start off by looking at the work [`spec_helper`](https://github.com/shoes/shoes4/blob/master/shoes-core/spec/spec_helper.rb) does to set everything up.

## Code Climate

The first thing that we do is pull in a top-level `code_coverage` [file](https://github.com/shoes/shoes4/blob/master/spec/code_coverage.rb).
This file pulls in a general purpose library called [simplecov](https://github.com/colszowka/simplecov) that tracks which lines of code get hit when a test suite runs. There are a number of frameworks that can hook up to this machinery, and the one that we've chosen for Shoes is [Code Climate](https://codeclimate.com/).

Code Climate keeps track of code complexity, duplication, and security vulnerabilities on every commit. It creates a report card with a GPA so that we can tell on each commit how that contribution improved (or worsened) the quality of code in the repo. You can see the Shoes 4 report card [here](https://codeclimate.com/github/shoes/shoes4).

So in this bit of code we start up `simplecov` and then hook up `CodeClimate` to it.

## Helpers

After that we require essential tools for the tests like RSpec, Pry, File Utilities, the Shoes Core codebase, and some helpers. All of these will be used at one point or another during the tests so we'll talk about them then (for example `WebMock` is used for testing methods like `download` that use an internet connection).

## Shared Examples

The last thing we do is `require` all of the [`shared_examples`](https://github.com/shoes/shoes4/tree/master/shoes-core/spec/shoes/shared_examples).

These examples are specs that we want to require *before* we run the specs. As RSpec puts it

> Shared examples let you describe behaviour of classes or modules. When declared,
a shared group's content is stored. It is only realized in the context of
another example group, which provides any context the shared group needs to
run.

You can read more about shared examples [here](https://www.relishapp.com/rspec/rspec-core/docs/example-groups/shared-examples).

## Wrap-up

And that's it for getting things set up. At this point, with the core library loaded, the shared-examples loaded, and a small pile of helpful gems ready to go, we can start executing each of the tests. I'll run through these mostly alphabetically but I'll do some classes out of order if that makes more sense, as usual, if you see a better a way to write or organize this information please let me know :)
