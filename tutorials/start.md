---
layout: page
title: 1. Where to start?
category: executables
permalink: /start/
---

Let's start by learning everything there is to know about a simple program:

```ruby
Shoes.app do
  para "Hello World"
end
```

Oh wait! Let's do a different program, that one is too complicated.

Seriously.

In fact, we'll probably do text blocks last as [Davor](https://github.com/davorb) [once suggested](http://blog.davor.se/blog/2012/07/16/contributing-to-shoes4/).

OK, so what's simpler? How about something that doesn't actually touch the DSL. Something like

```ruby
Shoes.app do
  puts "Hello World"
end
```

Nope, still too complicated. I want to start deep down at the bottom and this has too much going on!


Let's say that this app was called `app.rb`, then I'd run it like this

    $ bin/shoes testing/app.rb

<hr>
**Sidenote**: I'm assuming you've cloned/forked the `shoes4` repo and are sitting in the root directory when you run your test apps. You should put example apps like this in a `testing` directory, as shown above, because the shoes4 `.gitignore` already ignores that directory.

If this sidenote doesn't make sense, don't despair! Just go read up a little bit on Github and how to contribute to an open source project. There are some great tutorials out there. Once you have got that down, come on back! For the rest of these tutorials I'll be assuming only very basic Ruby knowledge since that's all I have :)
<hr>

The important thing here is that `bin/shoes` call. One thing that's neat about Shoes is that it ships with its own executable `shoes` so that we don't muddy the app with any `require`s, `extend`s, or `include`s. That's neat for the end user and a bit magical, so that means it will take a little work to understand. Since it's the first Shoes thing we're running into here, it's the best place to start.

## The Shoes Executable

So the first program we'll look at will just be

```ruby
puts "Hello World"
```

and we'll run it with

```
$ bin/shoes testing/hello.rb
```

First let's go find that executable. It's in a directory called [`/bin`](https://github.com/shoes/shoes4/blob/master/bin/shoes).

<hr>
**Sidenote**:The tutorials are pretty tightly bound to the way Shoes is written *right now* so if you are reading this and it doesn't match up with the actual code, or you find a broken link, then either let me know or fix it! I want these to be up-to-date and accurate and that means they'll need updating by conscientious readers!

Also I recommend that you open links to code (like the one above) in a new tab so that you can take a good look at the file I'm talking about **and** read my comments on it.
<hr>

This is one of many shell scripts in the Shoes project. If you don't know anything about shell scripts that's OK, I didn't either at the beginning of writing this tutorial. After writing it I've learned a lot. There's a neat tutorial about shell scripts at [linuxcommand.org](http://linuxcommand.org/writing_shell_scripts.php) and I recommend you briefly peruse it, at least the very first page to get an idea, and then maybe learn what a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) is too.

OK, so with some of those preliminaries down let's look at the script.

```sh
#!/usr/bin/env sh

# This is NOT the primary shoes that's installed--just a helper for local
# development purposes
cd ./bin
export SHOES_PICKER_BIN_DIR=`pwd -P`

cd ..
shoes-core/bin/shoes $@
```

First this script jumps into the `bin` directory and then exports a shell variable that holds the current working directory (we'll need that later). Finally it jumps back to the main `shoes4` directory and executes the `shoes` executable over in `shoes-core/bin/shoes`. The use of `$@` here means this script passes all the arguments that were given to it onto the next one untouched.

So let's go find that script!

<hr>
The `shoes` script over in `shoes-core` is the *real* `shoes` script, so that means this script is setting up our environment (ENV) for purposes that will be clear later on.
<hr>

## The *Real* Shoes Executable

OK, so if we head over to `/shoes-core/bin/shoes` then we'll see a sym-link to another script `shoes-stub` which is in the same directory. This is the *real* script: [`/shoes-core/bin/shoes-stub`](https://github.com/shoes/shoes4/blob/master/shoes-core/bin/shoes-stub).

We're going to go through this file a few lines at a time to understand it all. Let's start with the comment.

The first few lines tell us that if we want to learn why `shoes-core` uses both `shoes` and `shoes-stub` we'll need to go into the `ext/install/Rakefile` and see it in the context of installation. Since I'd like to hold installation off for another tutorial, let's keep on moving and trust that this was a wise choice (for now).

The next lines define two functions: `mac_move_to_link_dir` and `mac_readlink_f`.

Next we come to a `case` statement. It says `SCRIPT` is either `mac_readlink_f` or just `readlink -f` depending on whether the script is run on a mac (darwin). Since I want to stay somewhat focused, I'm going to leave as an exercise to the reader how the `mac_readlink_f` function works, and instead focus on what `readlink -f` means.

After reading the [stackoverflow post mentioned in the comment](http://stackoverflow.com/questions/242538/unix-shell-script-find-out-which-directory-the-script-file-resides/1638397#1638397) it looks like we're just setting the `SCRIPT` variable to be the path to the `shoes-stub` script we're looking at, then finding the directory of that script as `SCRIPTPATH` and finally adding `shoes-backend`. To make this clear, let's add some `echo`s after the `BACKEND_FILE` definition like so:

```sh
BACKEND_FILE="$SCRIPTPATH/shoes-backend"
echo $SCRIPT
echo $SCRIPTPATH
echo $BACKEND_FILE
```

we get the following output when running the script alone (on my mac I just double clicked the `shoes-stub` file).

```sh
/Users/KC/Programming/shoes4/shoes-core/bin/shoes-stub
/Users/KC/Programming/shoes4/shoes-core/bin
/Users/KC/Programming/shoes4/shoes-core/bin/shoes-backend
```


The next thing we do is see if a `shoes-backend` script actually exists in the directory, if so do nothing, if not then you need to run `shoes-picker` and hand it `SCRIPTPATH`.
<hr>
**Sidenote** The if statement here takes option `-e` which checks if a file exists. I found a guide at [tldp.org](http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_07_01.html)
<hr>

Let's take a look at what [`shoes-picker`](https://github.com/shoes/shoes4/blob/master/shoes-core/bin/shoes-picker) does before finishing this script.

## Shoes-Picker

Ah, the first piece of Ruby code our little program has encountered so far. It is short and sweet

```ruby
#!/usr/bin/env ruby
lib_directory = File.expand_path('../../lib', __FILE__)
$LOAD_PATH << lib_directory

require 'shoes/ui/picker'

# On Windows getting odd paths with trailing double-quote
bin_dir = ARGV[0].gsub('"', '')

Shoes::UI::Picker.new.run(bin_dir)
```

This little bit of code modifies our `LOAD_PATH` so that we can use a simple `require` and then `run` an instance of `Shoes::UI::Picker`. The definition of this class is over in [shoes-core/lib/shoes/ui/picker](https://github.com/shoes/shoes4/blob/master/shoes-core/lib/shoes/ui/picker.rb).

When the `run` method of `Picker` is called it
  * "bundles"
  * gets a generator file
  * writes the generator file

Let's talk about each of those steps.

#### **bundle**

I was a little confused about this bit of code, and the fantastic [@jasonrclark](https://github.com/jasonrclark) answered my question like so

> The key thing is what bundler/setup actually does, and that's setting up the load paths for your gems so that only things in your Gemfile are available. This is super important for local dev because our Gemfile forces everything to use the source copy rather than any gem-installed copies of Shoes.

So the point is that at this stage of development, the picker is mostly about getting the development environment set up. It's all primed to select backends, but that's not really the point right now.

In [this helpful explanation](https://github.com/shoes/shoes4/issues/1034#issuecomment-70397736) of the picker, Jason goes on to explain that bundler next requires the *correct* gems from the Gemfile and avoids anything that's installed.

#### **select\_generator**

This chunk of code first finds candidates via `Gem` which is provided by [rubygems](https://rubygems.org/) and searches through each of the gems on the load path. Right now that means `shoes-core`, `shoes-package`, and `shoes-swt`. The only one of those that contains a `generate-backend.rb` is swt and the method returns the path to [that file](https://github.com/shoes/shoes4/blob/master/shoes-swt/lib/shoes/swt/generate-backend.rb).

That means for now the only line of the `if/elsif` that gets used is 'candidates.one?'. But you can see that some mechanics are built in for the future when we'll have multiple backends.

#### **write\_backend**

The first thing we do is define a function for generating the backend by requiring the generator file. For SWT that function looks like this:

```ruby
require 'rbconfig'

def generate_backend(path)
  if RbConfig::CONFIG["host_os"] =~ /darwin/
    options = "-J-XstartOnFirstThread"
  end

  "jruby --1.9 #{options} #{path}/shoes-swt"
end
```

The output of this function is a string which will later be run in a shell script. That string will contain the string `"-J-XstartOnFirstThread"` if the host is using darwin architecture.

We do this because SWT needs to create a `Display` widget, and it can't do that on the OSX / Mac / darwin architecture unless it's on the `main` thread. JRuby has an option for doing just that: `-J-XstartOnFirstThread` and so it must be passed in when jruby tries to run the swt files. For another approach to learning about this problem check out [this chunk of the wiki](https://github.com/shoes/shoes4/wiki/SWT---JRuby-bootstrap-guide#a-little-swt-program).

After defining that function with the require, back in `Picker` we go through a little work (note that we use the `SHOES_PICKER_BIN_DIR` here) to get the exact path to the top-level bin directory and hands it over to `generate_backend`. The end result is that a file gets written in the correct place called `shoes-backend`. On my machine this file contains the following line of text:

```
jruby --1.9 -J-XstartOnFirstThread /Users/KC/Programming/shoes4/bin/shoes-swt
```

:tada: - Now we've got an executable to the backend!

## Back to `shoes-stub`

OK, so we left [`shoes-stub`](https://github.com/shoes/shoes4/blob/master/shoes-core/bin/shoes-stub#L60) to find out what `$SCRIPTPATH/shoes-picker $SCRIPTPATH` does. The answer was it writes out a file that contains a shell command starting with `jruby` and ending with a path to the `shoes-swt` script.

What follows next is that the last piece of `shoes-stub` just runs that file (the `cat` command means: read the file) using the correct path (`$SCRIPTPATH`) and passing along the arguments (`$@`).

So that's it for `shoes-stub`. All that this script did (essentially) was point at the `shoes-swt` binary and execute it. It might seem like a lot of work for such a simple task, but the complexity is necessary because of the division between running an app from source (like a developer does) and running it from the gem (which we haven't discussed yet).


## shoes-swt

Just like the top-level `shoes` script, this one is pointing us to the script over in the gem's `bin` directory: [`shoes-swt/bin/shoes-swt`](https://github.com/shoes/shoes4/blob/master/shoes-swt/bin/shoes-swt)

The first half of this duplicates the work that bundler did back in `Picker` in case that script didn't get called (remember it's a one time operation), and the last bit is:

```ruby
require 'shoes/ui/cli'
Shoes::CLI.new.run ARGV
```

So let's dive into the CLI a bit

## Shoes::CLI

CLI stands for Command Line Interface. That means this file is the one that is supposed to handle calls to the `shoes` command-line app runner.

So let's take a look at [the `run` method](https://github.com/shoes/shoes4/blob/master/shoes-core/lib/shoes/ui/cli.rb#L61-L74) and remember that ARGV has only 1 argument, the (relative) file path.

<hr>
**Sidenote**: The initialize here sets up the backend and the packager. Since I don't want to get into packaging let's take a quick look at the backend work done here.

First it `unshift`s the current directory from the `PATH` and requires the backend file, in our case `shoes/swt.rb`. Next it uses some mechanics `Shoes.load_backend` (`Shoes` got pulled in at the top of this file) and `initialize_backend` to setup the backend. Since for now Shoes has only 1 backend I'll not get into these either.
<hr>

`run` does the following:

1. parses the arguments
2. handles the case where 0 arguments are given
3. runs the app with the packager or the `execute_app` method.

#### 1. Parse the Arguments

This step uses ruby's built-in `OptionParser` to simultaneously create an options summary and define what the CLI should do when encountering the different options (see the [docs](http://ruby-doc.org/stdlib-2.2.0/libdoc/optparse/rdoc/OptionParser.html)). The intent then is for the explanation of what the opts do and the implementation to be unified, so instead of me explaining this step I'll let you just read it.

After the setup, it actually parses the args and returns the `OptionParser` object.

#### 2. Handle `args.empty?`

Next, if there are no arguments, as in

```
$ bin/shoes
```

then we exit after outputting the banner and program name like so:

```
Usage: shoes [-h] [-p package] file
Try 'shoes --help' for more information
```

#### 3. Package or Run

Since in this example we are not packaging, I'll ignore what `@packager.run` does and just look at `execute_app`. This method has one simple and key function: `load`. This method grabs the app (in our case `puts "Hello World"`). And then it **actually runs the app**.

## Conclusion

In this tutorial we walked through all of the code necessary to run a small bit of Ruby code. A lot happened, but most of the work was spent setting up and running the development environment. In the end we defined the entire Shoes library and *then* ran the app.
