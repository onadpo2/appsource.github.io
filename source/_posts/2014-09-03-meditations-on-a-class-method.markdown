---
layout: post
title: "Meditations on a Class Method"
author: Nad Flores
date: 2014-09-03 23:26:04 +0800
comments: true
has_image: false
categories: [Good Code, Ruby]
---

I keep a file of code I like. When looking for inspiration, I read through the file. It’s short; I often re-write the samples to be nothing but the minimally inspiring thought.
<!--more-->

Here is the first snippet:

``` ruby
def self.run(user)
  new(user).run
end
```
Think on it for yourself before I explain what it means to me. I’d like to hear what it means to you — leave a long comment here before you keep reading.

To me, it’s a reminder of how to write a beautiful class method: instantiate the class then call a method on the instance.

Look at it from the perspective of the person calling the class method. When you call a class method you want one of two things: either you want to construct an instance of the class itself (`.new`, or perhaps `.new_from_file`, `.new_for_widescreen`, and `.new_from_json`), or you want convenience.

Think of the class methods you’ve seen, or have written. If they are not in the above style, they might look more like this:

``` ruby
class ImageUploader
  def self.run(xpm)
    @@dimensions = geometry_for(xpm)
    @@color_palette = colors_for(xpm)
    svg = generate_svg(xpm)
  end

  def self.geometry_for(xpm)
    # ...
  end

  def self.colors_for(xpm)
    # ...
  end

  def self.generate_svg(xpm)
    # ...
  end
end
```

What a mess of an object. An abuse of the singleton pattern, where it wasn’t even intended. Class variables being used in an especially not-thread-safe way, plus a jumble of code that is all exposed. It is daunting to extend because it is a tricky thought process to understand the full implications of even using it.

When dealing with an object you want a small interface. As a user you want the fewest number of options, and the one with the best abstraction; as a developer you want to hide as much of the implementation as possible, giving you full freedom to change the internals. The best object is one with no exposed methods at all.

The above pattern gives you that. You call .run and pass a user, and it takes care of the rest. If the default constructor changes its arity, the instance method (#run) changes its name, or the object is re-written in C and needs to do pointer arithmetic first: you are protected.

The snippet has explicit names for things: run and user. This brings to mind the command pattern, and especially a command pattern for dealing with users. Perhaps something to kick off the backend signup process. The command pattern is a quick way to start reducing a god class (PDF); pushing various bits of User into the SignUp class will help simplify both.

The simplicity of the snippet is a reminder to use abstractions on the same “level”. Create an instance and call a method on that; perhaps in the instance’s #run method, it will instantiate a few more objects and call a method on those; and so on. Short methods all the way down, explained with clear but concise names.

This snippet happens to be in Ruby, an inspiration unto itself. A part of the power behind the command pattern is in Ruby’s duck typing. Let’s say this is a class method on SignUp. I know that I can pass SignUp itself to something that expects to call run, passing a user. In doing so, I know that I can fake it in a test with any object that responds to run:

``` ruby
class FakeSignup
  def initialize(should_succeed = true)
    @should_succeed = should_succeed
  end

  def run(user)
    unless @should_succeed
      raise "I am supposed to fail"
    end
  end
end
```

The idea of passing SignUp around makes me think of queues: you can add the SignUp class to a background job runner to get an asynchronous workflow from within Rails. You could spawn a co-routine, passing SignUp and a user object. Once you’ve been inspired by the snippet, a world of concurrency opens up.

So that’s what I think of when I see my first snippet from my collection of inspirational code. What do you see?
