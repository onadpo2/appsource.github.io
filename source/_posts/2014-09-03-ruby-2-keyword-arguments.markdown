---
layout: post
title: "Ruby 2 Keyword Arguments"
date: 2014-08-03 23:44:33 +0800
author: Nad Flores
comments: true
categories: ["Ruby"]
has_image: false
---
====================
Ruby 2.0 introduced first-class support for keyword arguments:

``` ruby
def foo(bar: 'default')
  puts bar
end

foo # => 'default'
foo(bar: 'baz') # => 'baz'
```

In Ruby 1.9, we could do something similar using a single Hash parameter:

```ruby
def foo(options = {})
  bar = options.fetch(:bar, 'default')
  puts bar
end

foo # => 'default'
foo(bar: 'baz') # => 'baz'
```

Ruby 2.0 blocks can also be defined with keyword arguments:

```ruby
define_method(:foo) do |bar: 'default'|
  puts bar
end

foo # => 'default'
foo(bar: 'baz') # => 'baz'
```

Again, to achieve similar behavior in Ruby 1.9, the block would take an options hash, from which we would extract argument values.

**Required keyword arguments**

Unfortunately, Ruby 2.0 doesn’t have built-in support for required keyword arguments. Luckily, Ruby 2.1 introduced required keyword arguments, which are defined with a trailing colon:

```ruby
def foo(bar:)
  puts bar
end

foo # => ArgumentError: missing keyword: bar
foo(bar: 'baz') # => 'baz'
```

If a required keyword argument is missing, Ruby will raise a useful ArgumentError that tells us which required argument we must include.

**Keyword arguments vs options hash**

With first-class keyword arguments in the language, we don’t have to write the boilerplate code to extract hash options. Unnecessary boilerplate code increases the opportunity for typos and bugs.

With keyword arguments defined in the method signature itself, we can immediately discover the names of the arguments without having to read the body of the method.

Note that the calling code is syntactically equal to calling a method with hash arguments, which makes for an easy transition from options hashes to keyword arguments.

**Keyword arguments vs positional arguments**

Assume we have a method with positional arguments:

```ruby
def mysterious_total(subtotal, tax, discount)
  subtotal + tax - discount
end

mysterious_total(100, 10, 5) # => 105
```

This method does its job, but as a reader of the code using the mysterious_total method, I have no idea what those arguments mean without looking up the implementation of the method.

By using keyword arguments, we know what the arguments mean without looking up the implementation of the called method:

```ruby
def obvious_total(subtotal:, tax:, discount:)
  subtotal + tax - discount
end

obvious_total(subtotal: 100, tax: 10, discount: 5) # => 105
```

Keyword arguments allow us to switch the order of the arguments, without affecting the behavior of the method:

```ruby
obvious_total(subtotal: 100, discount: 5, tax: 10) # => 105
```

If we switch the order of the positional arguments, we are not going to get the same results, giving our customers more of a discount than they deserve:

```ruby
mysterious_total(100, 5, 10) # => 95
```

**Connascence and trade-offs**

*Connascence between two software components A and B means either 1) that you can postulate some change to A that would require B to be changed (or at least carefully checked) in order to preserve overall correctness, or 2) that you can postulate some change that would require both A and B to be changed together in order to preserve overall correctness. - Meilir Page-Jones, What Every Programmer Should Know About Object-Oriented Design*

When one Ruby method has to know the correct order of another method’s positional arguments, we end up with connascence of position.

If we decide to change the order of the parameters to `mysterious_total`, we must change all callers of that method accordingly. Not only that, but our mental model of how to use this method must change as well, which isn’t as simple as a find/replace.

Like most things, keyword arguments have their trade-offs. Positional arguments offer a more succinct way to call a method. Usually, the code clarity and maintainability gained from keyword arguments outweigh the terseness offered by positional arguments. I would use positional arguments if I could easily guess their meanings based on the method’s name, but I find this rarely to be the case.
