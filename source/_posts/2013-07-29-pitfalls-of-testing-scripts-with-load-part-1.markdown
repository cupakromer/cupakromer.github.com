---
layout: post
title: "Pitfalls of Testing Scripts With Load - Part 1"
date: 2013-07-29 13:29
comments: false
categories:
- ruby
- testing
- script
---

Previously it was shown how to [use load to test scripts](http://aaronkromer.com/blog/2013-07-11-testing-scripts-with-load.html). As with all techniques, there are some drawbacks to using `load`.

### Hidden Require Dependency

Take script:

```ruby
#!/usr/bin/env ruby
$:.unshift File.join File.dirname(__FILE__), "..", "lib"

require 'local_server'

logger = Logger.new($stderr)

LocalServer.new(logger).run
```

and passing spec:

```ruby
require 'spec_helper'
require 'logger'
require 'stringio'

describe 'Running the server locally' do
  it 'logs that it is running' do
    io = StringIO.new
    allow(Logger).to receive(:new).and_return(Logger.new io)

    expect{ load 'script/local-server' }.to change{ io.string }
      .to include 'SERVER: starting up'
  end
end
```

However, when the script is run standalone, it errors with:

```
uninitialized constant Logger (NameError)
```

Be aware that since [`load`](http://ruby-doc.org/core-2.0/Kernel.html#method-i-load)
happens in the current spec context, a missing `require` may not be noticed if
it is required by the spec.

Whenever possible have at least one test that shells out as a sanity check.
