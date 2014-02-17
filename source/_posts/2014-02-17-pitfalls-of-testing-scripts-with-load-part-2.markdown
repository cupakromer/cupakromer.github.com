---
layout: post
title: "Pitfalls of Testing Scripts With Load - Part 2"
date: 2014-02-17 10:15
comments: false
categories:
- ruby
- testing
- script
---

{% render_partial _posts/_pitfalls_of_test_load_posts.md %}

## Scope Smashing

Take the following script and spec:

```ruby
#!/usr/bin/env ruby

def current_user
  'Bob'
end
```

```ruby
require 'spec_helper'

def current_user
  'Alice'
end

describe 'Running the server locally' do
  it 'logs that it is running' do
    load 'script/current-user-smash'

    expect(current_user).to eq 'Alice'
  end
end
```

Alas, the spec fails with:

```
expected: "Alice"
     got: "Bob"
```

This is due to how [`load`](http://ruby-doc.org/core-2.1.0/Kernel.html#method-i-load) works:

> If the optional `wrap` parameter is `true`, the loaded script will be
> executed under an anonymous module, protecting the calling program’s global
> namespace. In no circumstance will any local variables in the loaded file be
> propagated to the loading environment.

While it is easy to spot the issue this time, that's not normally the case.
Say if the additional method is define by a gem or in supporting file. Or if
you are testing multiple scripts that each define the same top-level methods.
These conditions will result in very strange and difficult to debug failures.
Of course, it's always a good idea to not define top-level methods to begin
with.

Instead always pass the additional `wrap` parameter. Here I've named it as a
descriptive inline variable to reveal it's intent:

```ruby
it 'logs that it is running' do
  load 'script/current-user-smash', _in_sandbox = true

  expect(current_user).to eq 'Alice'
end
```
