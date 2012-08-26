---
layout: post
title: "Travis CI Setup for Plain Old Ruby"
date: 2012-07-18 01:11
comments: false
categories:
- ruby
- travis ci
---

I recently wanted to setup some automated testing for a mini project I
was working on. I was using regular Ruby and wanted to try working with
[Travis CI](http://travis-ci.org/). For public projects, this is a great _free_
resource which I cannot recommend enough.

They have a decent amount of [documentation](http://about.travis-ci.org/docs/)
on how to setup your project. However, it took me longer than I thought
necessary to get a basic Ruby project configured.

In the end, it came down to needing three things:

* The `rake` gem present in either a `Gemfile` or `*.gemspec` file

* A `.travis.yml` file

```yaml .travis.yml
language: ruby
rvm:
  - 1.9.2
  - 1.9.3
```

* A `Rakefile` with a default task to run the tests

```ruby Rakefile
#!/usr/bin/env rake
require "bundler/gem_tasks"

require 'rspec/core/rake_task'

desc 'Default: Run specs.'
task default: :spec

desc 'Run specs'
RSpec::Core::RakeTask.new do |task|
  task.rspec_opts = "--format doc"
end
```

Just drop those two files into your project directory and you are ready
to experience the fun of CI.
