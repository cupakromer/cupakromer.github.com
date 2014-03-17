---
layout: post
title: "Monitoring Specific Messages in RSpec"
date: 2013-07-17 11:34
comments: true
categories:
- ruby
- rspec
---

{% render_partial _posts/_monitor_specific_messages_posts.md %}

A common question I see asked in the #rspec IRC channel is:

> How do I verify only a specific message is received?

The context of this question is usually a method that sends the same message
multiple time, but with different arguments.

Take the following contrived example:

```ruby
def do_stuff(thing)
  thing.do :having_fun
  thing.do :drink_coffee
  thing.do :code
end

describe 'Coffee time!' do
  it 'drinks the coffee' do
    thing = double('Thing')

    expect(thing).to receive(:do).with(:drink_coffee)

    do_stuff thing
  end
end
```

Alas, it fails:

```text
Double "Thing" received :do with unexpected arguments
         expected: (:drink_coffee)
              got: (:having_fun)
```

The solution is to add a generic stub (a catchall) for the desired message:

```ruby
describe 'Coffee time!' do
  it 'drinks the coffee' do
    thing = double('Thing')

    allow(thing).to receive(:do)
    expect(thing).to receive(:do).with(:drink_coffee)

    do_stuff thing
  end
end
```

There is another solution which can also be used:
[`as_null_object`](https://www.relishapp.com/rspec/rspec-mocks/v/2-14/docs/method-stubs/as-null-object).

The issue with `as_null_object` is that it will happily hum along even for
invalid and unexpected messages. The double stub pattern above is
explicit in the test, matching the catchall only with the message expectation.

```ruby
describe 'Coffee time!' do
  it 'drinks the coffee' do
    thing = double('Thing').as_null_object

    expect(thing).to receive(:do).with(:drink_coffee)

    do_stuff thing
  end
end
```

Stay tuned for [RSpec3](http://myronmars.to/n/dev-blog/2013/07/the-plan-for-rspec-3)
when `as_null_object` doubles will be smart enough to only stub matching
messages.  Or check out one of the many plugins:
[`rspec-fire`](https://github.com/xaviershay/rspec-fire),
[`bogus`](https://github.com/psyho/bogus), and
[`spy`](https://github.com/ryanong/spy).
