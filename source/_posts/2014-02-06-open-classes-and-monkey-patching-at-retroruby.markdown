---
layout: post
title: "Discussion about open classes and monkey patching at #retroruby"
date: 2014-02-06 15:28
comments: false
categories:
- ruby
- retroruby
---

While at [RetroRuby 2014](http://retroruby.org) a very good question was asked
in the newbie track. The following code sample had just been shown on some
basic awesomeness of Ruby:

```ruby
"Words" * 2
# => "WordsWords"
```

One of the participants then asked why that worked but the inverse didn't:

```ruby
2 * "Words"
# TypeError: String can't be coerced into Fixnum
```

## Oops what just happened?

In Ruby, virtually everything is accomplished by sending a message to another
object. Above we would say:

> _Send the message `*` to `"Words"` with parameter `2`_

> _Send the message `*` to `2` with parameter `"Words"`_

"Sending a message" is the Java equivalent of calling a method. Another way to
write the above is:

```ruby
"Words".*(2)
# => "WordsWords"
```

Here we used the normal "method" calling syntax: `obj.method(args)`

You'll also probably see the following:

```ruby
"Words".send(:*, 2)
# => "WordsWords"
```

This time we explicitly sent the message: `obj.send(message, args)`

With [`send`](http://ruby-doc.org/core-2.1.0/Object.html#method-i-send) Ruby
doesn't check if the message you passed was supposed to be public or private.
Generally, what you wanted to do was dynamically send the message while still
making sure to respect the public API of the object. To do this you should use
[`public_send`](http://ruby-doc.org/core-2.1.0/Object.html#method-i-public_send)
instead: `obj.public_send(message, args)`.

So back to the original issue. Both
[`String`](http://ruby-doc.org/core-2.1.0/String.html) and
[`Fixnum`](http://ruby-doc.org/core-2.1.0/Fixnum.html) respond to the message
`*`.

  * [`String#*`](http://ruby-doc.org/core-2.1.0/String.html#method-i-2A)
  * [`Fixnum#*`](http://ruby-doc.org/core-2.1.0/Fixnum.html#method-i-2A)

However, `String`'s implementation knows what to do when the argument is a
`Fixnum`. When we reverse it, the `Fixnum` implementation doesn't understand
what to do with a `String` argument.

## How to fix this?

Well you probably shouldn't. But just for fun we'll use Ruby's open class
behavior to monkey patch `Fixnum`'s `*` implementation.

```ruby
class Fixnum # We just re-opened the class

  def *(arg) # We're redefining the * message - this destroys the
             # previous implementation!!!
    arg * self
  end

end

2 * "Words"
# => WordsWords
```

It worked!! Before you go doing this to everything, be aware we've now lost the
ability to do normal multiplication. Additionally, we'll overflow the stack
trying to multiply two `Fixnum`s.

```ruby
2 * 2
# SystemStackError: stack level too deep
```

## Wrap Up

In Ruby, most things are messages sent from one object to another. Some times
the language gives a little extra syntactic sugar to make sending the message
more readable.

Additionally, open classes and monkey patching have their uses. Just be aware
that with great power comes great responsibility. Always stop to ask yourself
if this is really a good idea; and try to never change existing behavior if you
can.


## Level Up!

So what if we just wanted to extend the existing functionality.  There are a
lot of ways to do this and covering each, along with the pros and cons is very
out of scope for this post. I'm just going to demonstrate one way to illustrate
how it could be done.

You'll need a new irb or Pry session so we get the original implementation for
`Fixnum` back.

```ruby
class Fixnum # We just re-opened the class

  # We need to store a reference to the original implementation One way to do
  # this is to create an alias for the original implementation so we can call
  # it later. Be sure to pick a descriptive name so that it isn't accidentally
  # overwritten by something else.
  alias :star_without_string_support :*

  def *(arg) # Redefine the message destroying the previous implementation!!!
    if arg.is_a? String
      arg * self
    else
      star_without_string_support arg  # use the original implementation
    end
  end

end
```
