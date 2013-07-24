---
layout: post
title: "Monitoring Specific Messages in RSpec Part 2"
date: 2013-07-24 10:11
comments: false
categories:
- ruby
- rspec
---

[Last time](http://aaronkromer.com/blog/2013-07-17-monitoring-specific-messages-in-rspec.html) it was demonstrated how it is possible to monitor only a desired
message expectations. While that technique is useful for a vast majority of
use cases, sometimes you need a bit more complexity.

Say for example, the desire is to verify the state of the provided parameter.
As in this very contrived example:

```ruby
class Factory
  def perform_maintenance
    widgets.select{ |widget| widget.broken? }
           .each{ |widget| mechanic.fix widget }
    # Other stuff
  end
end

describe Factory do
  context 'making sure current stock is ready' do
    it 'only fixes broken widgets' do
      mechanic = double('Mechanic')
      factory = Factory.new(
        widgets: [
          double('Widget', broken?: true),
          double('Widget', broken?: false),
        ],
        mechanic: mechanic
      )

      states = []
      expect(mechanic).to receive(:fix) do |widget|
        states << widget.broken?
      end

      expect{ factory.perform_maintenance }.to change{states}.to [true]
    end
  end
end
```

This leverages the [stub with substitute implementation](https://www.relishapp.com/rspec/rspec-mocks/v/2-14/docs/method-stubs/stub-with-substitute-implementation)
provided by RSpec mocks. This can also be used with `allow`.

While the above version has it's uses, it tends to hide some of the intent in
the closure manipulation. A slightly more expressive method is to just add
the state expectation in the block:

```ruby
expect(mechanic).to receive(:fix) do |widget|
  expect(widget).to be_broken
end
```

These techniques are great for setting up generic message stubs which require
that something with specific state is provided. By adding it to a generic
`allow`, it ensures when the contract is broken anywhere in the code under test,
the test will properly fail.
