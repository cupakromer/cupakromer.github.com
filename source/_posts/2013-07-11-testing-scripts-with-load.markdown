---
layout: post
title: "Testing Scripts with Load"
date: 2013-07-11 12:44
comments: false
categories:
- ruby
- testing
- script
---

A standard method for testing Ruby scripts is to shell out to the script using
[<code>Kernel#\`</code>](http://ruby-doc.org/core-2.0/Kernel.html#method-i-60)
or [`Kernel#system`](http://ruby-doc.org/core-2.0/Kernel.html#method-i-system).
Both of these follow a [`fork`](http://ruby-doc.org/core-2.0/Kernel.html#method-i-fork)
and [`exec`](http://ruby-doc.org/core-2.0/Kernel.html#method-i-exec) pattern to
run the command.

However, there are instances where it is necessary to mock/stub something in
the script.  The above technique fails in this situation due to `exec`
replacing the current process; the stubs simply cease to exist.

The solution: use [`load`](http://ruby-doc.org/core-2.0/Kernel.html#method-i-load)

```ruby
# Script
#!/usr/bin/env ruby
$:.unshift File.join File.dirname(__FILE__), "..", "lib"

require 'logger'
require 'monitor'
require 'pathname'
require 'temperature_api'
require 'tracker'

logger    = Logger.new($stderr)
file_path = Pathname.new(ARGV[0])
tracker   = Tracker.new(backup: file_path)
api       = TemperatureApi.new('api.server.com')

Monitor.new(tracker, api, logger).run
```

```ruby
# Spec
describe 'Monitoring cpu heat' do

  it 'uploads the temperature to the server' do
    stub_const('ARGV', ['monitor-integration.bak'])
    stub_request(:put, 'https://api.server.com/temp').to_return(status: 200)

    expect(load 'script/monitor')
      .to have_requested(:put, 'https://api.server.com/temp')
      .with(body: /\A{"temp":\d+}\z/)
  end

end
```

In conclusion, no more excuses for not have integration tests for your scripts.
