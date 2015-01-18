---
layout: page
title: Where to start?
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

The important thing here is that `bin/shoes` call. One thing that's neat about Shoes is that it ships with it's own executable `shoes` so that we don't muddy the app with any `require`s, `extend`s, or `include`s. That's neat for the end user and a bit magical, so that means it will take a little work to understand. Since it's the first Shoes thing we're running into here, it's the best place to start.

## The Shoes Executable

So the first program we'll look at will just be

```ruby
puts "Hello World"
```

and we'll run it with

    $ bin/shoes testing/hello.rb

First let's go find that executable. It's in a directory called [`/bin`](https://github.com/shoes/shoes4/blob/master/bin/shoes).

<hr>
**Sidenote**:The tutorials are pretty tightly bound to the way Shoes is written *right now* so if you are reading this and it doesn't match up, then either let me know or fix it! I want these to be up-to-date and accurate and that means they'll need updating by conscientious readers!
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

First this script jumps into the `bin` directory and then exports a shell variable that holds the current working directory. Lastly it jumps back to the main `shoes4` directory and executes the `shoes` executable over in `shoes-core/bin/shoes`. The use of `$@` here means this script passes all the arguments that were given to it onto the next one untouched.

So let's go find that script!

<hr>
The `shoes` script over in `shoes-core` is the *real* shoes script, so that means this script is setting up our environment (ENV) for purposes that will be clear later on.
<hr>

## The *Real* Shoes Executable

OK, so if we head over to `/shoes-core/bin/shoes` then we'll see a sym-link to another script `shoes-stub` which is in the same directory. This is the *real* script: [`/shoes-core/bin/shoes-stub`](https://github.com/shoes/shoes4/blob/master/shoes-core/bin/shoes-stub).

We're going to go through this file a few lines at a time to understand it all. Let's start with the comment.

The first few lines tell us that if we want to learn why `shoes-core` uses both `shoes` and `shoes-stub` we'll need to go into the `ext/install/Rakefile` and see it in the context of installation. Since I'd like to hold installation off for another installation, let's keep on moving and trust that this was a wise choice (for now).

The next lines define two functions: `mac_move_to_link_dir` and `mac_readlink_f` we'll come back to them when they get called.

Next we come to a `case` statement. It says `SCRIPT` is either `mac_readlink_f` or just `readlink -f` depending on whether the script is run on a mac (darwin). Since I want to stay somewhat focused I'm going to leave as an exercise to the reader how the `mac_readlink_f` function works and instead focus on what `readlink -f` means.

After reading the [stackoverflow post mentioned in the comment](http://stackoverflow.com/questions/242538/unix-shell-script-find-out-which-directory-the-script-file-resides/1638397#1638397) it looks like we're just setting the `SCRIPT` variable to be the path to the `shoes-stub` script we're looking at, then finding the directory of that script as `SCRIPTPATH` and finally adding `shoes-backend`. To make this clear I added some `echo`s after the backend_file definition like so:
```sh
BACKEND_FILE="$SCRIPTPATH/shoes-backend"
echo $SCRIPT
echo $SCRIPTPATH
echo $BACKEND_FILE
```

and got the following output when running the script alone (on my mac I just double clicked the `shoes-stub` file).

```sh
/Users/KC/Programming/shoes4/shoes-core/bin/shoes-stub
/Users/KC/Programming/shoes4/shoes-core/bin
/Users/KC/Programming/shoes4/shoes-core/bin/shoes-backend
```


The next thing we do is see if a `shoes-backend` script actually exists in the directory, if so do nothing, if not then you need to run `shoes-picker` and hand it `SCRIPTPATH`.
<hr>
**Sidenote** The if statement here takes option `-e`. I found a guide at [tldp.org](http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_07_01.html)
<hr>

## Shoes-Picker

Ah, the first piece of Ruby code our little program has encountered so far. It is short and sweet

```ruby
#!/usr/bin/env ruby
lib_directory = File.expand_path('../../lib', __FILE__)
$LOAD_PATH << lib_directory

require 'shoes/ui/picker'
Shoes::UI::Picker.new.run
```

This little bit of code modifies our `LOAD_PATH` so that we can use a simple `require` and then `run` an instance of `Shoes::UI::Picker`. The definition of this class is over in [shoes-core/lib/shoes/ui/picker](https://github.com/shoes/shoes4/blob/master/shoes-core/lib/shoes/ui/picker.rb).
