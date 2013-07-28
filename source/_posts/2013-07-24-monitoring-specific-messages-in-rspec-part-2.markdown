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
As in this very contrived example (_Update: 2013-07-28 Paired down example to
only show test and not implementation_):

```ruby
describe Factory do
  context 'making sure current stock is ready' do
    it 'only fixes broken widgets' do
      # Setup code

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
provided by RSpec mocks. It can also be used with the `allow` syntax.

While the above version has it's uses, it tends to hide some of the intent in
the closure manipulation. A slightly more expressive method is just to add
the state expectation in the block:

```ruby
expect(mechanic).to receive(:fix) do |widget|
  expect(widget).to be_broken
end
```

This last technique is great for setting up generic message stubs which require
that something with specific state is provided. By adding it to a generic
`allow`, it ensures when the contract is broken anywhere in the code under test,
the test will properly fail.
