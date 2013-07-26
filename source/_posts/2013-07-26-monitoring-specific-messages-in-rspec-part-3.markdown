---
layout: post
title: "Monitoring Specific Messages in RSpec Part 3"
date: 2013-07-26 10:13
comments: false
categories:
- ruby
- rspec
---

Continuing the series (
[part 1](http://aaronkromer.com/blog/2013-07-17-monitoring-specific-messages-in-rspec.html),
[part 2](http://aaronkromer.com/blog/2013-07-24-monitoring-specific-messages-in-rspec-part-2.html)
) on matching messages in RSpec, the next logical step is custom argument
matchers.

```ruby
expect(mechanic).to receive(:fix).with something_broken
```

Using the RSpec [matcher DSL](https://www.relishapp.com/rspec/rspec-expectations/v/2-14/docs/custom-matchers)
this could simply look like:

```ruby
RSpec::Matchers.define :something_broken do
  match do |thing|
    thing.broken?
  end
end
```

That's all there is to it. Now it can be used as both a regular matcher and as
an argument matcher.

If the matcher needs to be created from
[scratch](http://rubydoc.info/gems/rspec-expectations/RSpec/Matchers), a
`matches?` method must be defined instead:

```ruby
class SomethingBroken
  def matches?(target)
    target.broken?
  end
end

def something_broken
  SomethingBroken.new
end
```

This works just fine as a normal matcher, however, when used as an argument
matcher, it will always fail. The reason is that argument matchers are invoked
with the `==` operator, which by [default](http://ruby-doc.org/core-2.0/BasicObject.html#method-i-3D-3D),
verifies if the objects are the same object.

Attempting to use a normal matcher with the
[`change`](https://www.relishapp.com/rspec/rspec-expectations/v/2-14/docs/built-in-matchers/expect-change)
expectation also
[oddly](https://github.com/rspec/rspec-expectations/issues/276) fails, due to
`change` invoking the
[`===`](http://aaronkromer.com/blog/2012-10-12-equals-equals-equals-the-forgotten-equality.html)
message, not `matches?`. Since the [default `===` behavior is
`==`](http://ruby-doc.org/core-2.0/Object.html#method-i-3D-3D-3D), the existing
argument matchers currently work with it.

There is active talk / changes
[happening](https://github.com/rspec/rspec-expectations/issues/280) to
standardize the matchers to `===`. This will allow for a more consistent and
composable interface. It also has the added benefit of allowing the matchers to
be used in more complex conditionals using `case` statements.

To fix the class based matcher simply add the necessary alias(es):

```ruby
class SomethingBroken
  def matches?(target)
    target.broken?
  end
  alias_method :==, :matches?
  alias_method :===, :matches?  # Not technically necessary due to default ==
end
```

Note that with such a simple matcher, there is no reason it cannot be created
as a simple composed method using an existing matcher:

```ruby
def something_broken
  be_broken   # Note: Not all built in matchers have == aliases yet
end
```
