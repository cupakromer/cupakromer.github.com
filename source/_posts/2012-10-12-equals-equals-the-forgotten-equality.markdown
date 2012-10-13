---
layout: post
title: "== the Forgotten Equality"
date: 2012-10-12 19:04
comments: true
categories:
- ruby
- equality
- comparison
---

Recently I was looking for a way to do a comparison on a `String` with
either another `String` or a `Regexp`. Most of the discussions on equality
focused on `==`, `eql?`, `equal?`. None of which would satisfy the requirement.
So I was left with this code:

```ruby
def matches(compare_with)
  if compare_with.is_a?(Regexp)
    @data_string =~ compare_with
  else
    @data_string == compare_with
  end
end
```

I was less than thrilled. So I did what everyone does, I asked the internet.
Thanks to Twitter, specifically James Edward Gray II
[@JEG2](https://twitter.com/jeg2) who btw completely rocks, I was pointed at
`===`. Though the documentation on `===` leaves something to be desired:

  {% blockquote Dave Thomas, Programming Ruby 1.9, page 128 http://pragprog.com/book/ruby/programming-ruby %}
  Used to compare each of the items with the target in the `when` clause of
  a `case` statement.
  {% endblockquote %}

  * The [String API](http://ruby-doc.org/core-1.9.3/String.html#method-i-3D-3D-3D)
    sneakily directs you to `==` but doesn't outright state they are the same
  * The [Regexp API](http://ruby-doc.org/core-1.9.3/Regexp.html#method-i-3D-3D-3D)
    states it as a synonym for [`Regexp#=~`](http://ruby-doc.org/core-1.9.3/Regexp.html#method-i-3D-7E)

The thing to remember is with `case` when you have the following:

```ruby
case thing
when other_thing
  # stuff
end
```

You are just saying `other_thing === thing`. The comparison is performed with
the `when` expression as the lvalue.

This means I could rewrite the `matches` method as:

```ruby
def matches(compare_with)
  compare_with === @data_string
end
```

This also means it's possible to be more flexible on the match:

```ruby
# @data_string = "coding for fun"
matches "oding"           # false
matches "coding for fun"  # true
matches /oding/           # true
matches String            # true
```

So, the next time you're thinking of writing some code that needs to
change based on class type or how something compares with something else, think
if a `case` statement applies. If it does, see if `===` works to produce better
code.
