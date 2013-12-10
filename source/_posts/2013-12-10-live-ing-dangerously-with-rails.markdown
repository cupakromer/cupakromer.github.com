---
layout: post
title: "Live-ing Dangerously with Rails"
date: 2013-12-10 13:08
comments: true
categories:
- ruby
- rails
---
Recently at work, we've been spinning up a bunch of demo apps on Heroku. This
has worked out well for us so far. However, some of these apps require
background data crunching. This hasn't been a problem as we just add a Rake
task, or two, and set the Heroku scheduler to take care of things.

Then comes the day when the sales team says:

> "Oh hey. Can we re-run all the numbers but with different configurations?"

Sure, that's not a problem you think:

```bash
$ heroku run rake redo_all_the_things[config.yml]
```

However, there's a few issues with this:

  1. The `config.yaml` needs to be on Heroku
  2. This counts against your active dyno time

See the thing is, Heroku [doesn't let you create files](https://devcenter.heroku.com/articles/read-only-filesystem)
that aren't already in your VC system from outside the app. It's now
[sorta possible on Cedar](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem),
but not really for this purpose.

Additionally, if these tasks turn out to be long running, that'll quickly eat
up the overhead on the 'free' dyno. For demo apps of non-paying customers
that's not always an option.

Enter the `live` environment.

Just add a new `live` environment to the Rails application. This allows us to
run Rake locally, but against the production database. Additionally, allowing
for custom configuration files to just be stored on our local system. And we
don't spin up a new dyno on Heroku.  Win, win!

To setup this up:

* Update the `Gemfile`

```ruby
# Gemfile

# We use dotenv-rails to help manage environment configs locally.
# Add our new environment:
gem 'dotenv-rails', groups: [:development, :test, :live]
```

* Add the live specific sample file: `.env.live-example`

```ruby
# .env.live-example

# Rename this file to `.env.live` and it will be automatically sourced by
# rails. Uncomment the settings for them to be picked up.
#
# Grab the settings from the output of:
#
#  $ heroku config
#
# Running against the Heroku production DB directly
# BE CAREFUL!! YOU HAVE BEEN WARNED!!
#DATABASE_NAME=WARNING
#DATABASE_HOST=DANGER
#DATABASE_PORT=WILL
#DATABASE_USER=ROBINSON
#DATABASE_PASSWORD=SERIOUSLY
```

* Update the `database.yml` file

```yaml
# config/database.yml

live:
  adapter: postgresql
  encoding: unicode
  database: <%= ENV['DATABASE_NAME'] %>
  host: <%= ENV['DATABASE_HOST'] %>
  port: <%= ENV['DATABASE_PORT'] %>
  username: <%= ENV['DATABASE_USER'] %>
  password: <%= ENV['DATABASE_PASSWORD'] %>
```

* Copy `config/environments/production.rb` to `config/environments/live.rb`.
  If you wish to symlink it instead [be wary of how git treats that](http://stackoverflow.com/questions/954560/what-does-git-do-to-files-that-are-a-symbolic-link).


Now running our tasks is as simple as:

```bash
$ RAILS_ENV=live rake redo_all_the_things[config-fun-edition.yml]
```
