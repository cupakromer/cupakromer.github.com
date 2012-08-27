---
layout: post
title: "Everyone Should Learn metaprogramming"
date: 2012-08-14 17:54
comments: true
categories:
- ruby
- meta
- metaprogramming
---

A few weeks ago was the first [Steel City Ruby conference](http://steelcityrubyconf.org/).
It was a great conference and I would highly recommend you attend next
year. While there, I had the opportunity to give my first lightning talk.

[{% img right /images/scrc12_lightning_talk.jpg 391 287 "SCRC'12 Lightning Talk" 'An image of me presenting at SCRC12' %}](http://www.flickr.com/photos/charliekilo/7717397572/in/photostream)
Now I'm not one for public speaking, in fact it's terrifying for me.
However, the Ruby community is awesome, friendly, and very encouraging.
After thinking about it for a while, talking with several people, and
reflecting on my time participating in [Scrappy Academy](http://scrappyacademy.com/),
I decided that it was important to speak up about metaprogramming.

I titled my talk: "_Demystifying Metaprogramming_." The intent was to
encourage those people who are afraid of "metaprogramming" to give it a
try. I strongly believe if you have been programming Ruby (or Rails -
this is just Ruby with a fun web framework) for more than two months,
you owe it to yourself to learn metaprogramming.

To most people, this included me, they hear about "metaprogramming" as
this advanced technique that should be avoided by all but the most
wizardly among us. Frankly, this big scary stigma attached to
metaprogramming just isn't warranted.

{% pullquote %}
I'll let you in on the big secret, {" "metaprogramming" is just
programming "}. Yep. It's not scary. It may be a bit complicated, but at
the end of the day it is well worth your time to learn. You may even find
it to be fun.
{% endpullquote %}

To help ease you into the topic, we'll start with a simple contrived
example to demonstrate that you already know everything needed to get
started.

```ruby
def say( phrase )
  puts phrase
end
```

Yep. That's a really basic method. It simply `puts` the object that is
passed in. To use this method in [irb](http://en.wikipedia.org/wiki/Interactive_Ruby_Shell)
or [pry](https://github.com/pry/pry) we could simply:

```ruby
> def say( phrase )
*  puts phrase
* end
=> nil
> say "Steel City Ruby I heard you like programming."
Steel City Ruby I heard you like programming.
=> nil
```

This shouldn't be anything shocking so far. In fact, all we've done is
create a method that calls a method (`puts`) and pass it a parameter. In
fact, this concept is at the heart of the "meta" programming.

All you really are doing is writing methods, that call another method,
and pass it a parameter. It just so happens that the parameter you pass,
tends to be a block that defines a new method.

{% blockquote %}
Write a method, that calls a method, that creates a method.*
{% endblockquote %}

So to extend our example and add the "meta" part to it, we'll just:

  * Wrap our method in another method
  * Pass our method as the parameter to another method

```ruby
def just_programming
  class_eval do
    def say( phrase )
      puts phrase
    end
  end
end
```

In this case, we're using the special [`class_eval`](http://www.ruby-doc.org/core-1.9.3/Module.html#method-i-class_eval)
method. In a nutshell, this will change the value of `self` and then
execute the block in this context. In our case, we provide it a block
which contains the method definition we wish to dynmically create. The
trick here is that class definitions and methods are active code. So the
call to `def say( phrase )` is run just as if we had typed it directly
in the original class definition.

To be able to use this in a context more familiar, we'll just wrap this
in a module. We can then `extend` that module in our class and
dynamically create our method by using our class method
`just_programming`.

```ruby
module Meta
  def just_programming
    class_eval do
      def say( phrase )
        puts phrase
      end
    end
  end
end

class Foo
  extend Meta
  just_programming
end

f = Foo.new
f.say "Steel City Ruby I heard you like programming."
=> "Steel City Ruby I heard you like programming."
```

I hope this has helped illustrate that "metaprogramming" is only
programming. It isn't anything to be afraid of. Sure, it can get
complicated at times, but then, what code concepts beyond the bare
basics can't?

You owe it to yourself to learn these nifty and fun coding techniques.
They will demystify manly things you think are "magic", provide a
deeper understanding of the Ruby language, and add tools to your
belt enabling you to write better code.

There are many resources on these topics if you do a search.
Additionally, [The Pragmatic Bookshelf](http://pragprog.com/) has a
good set of resources on Ruby and metaprogramming. I found the
videos series on [The Ruby Object Model and Metaprogramming](http://pragprog.com/screencasts/v-dtrubyom/the-ruby-object-model-and-metaprogramming)
very enlightening.

Happy programming!

\* Technically, this should really be "Write a method, that _sends a
message_, that creates a method." However, I wanted to emphasise the
progression and went with this version.
