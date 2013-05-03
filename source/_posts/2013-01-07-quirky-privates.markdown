---
layout: post
title: "Quirky Privates"
date: 2013-01-07 20:35
comments: true
categories:
- ruby
- note
---
Just one of those little Ruby language quirks. Normally, you have to use
the implicit receiver when sending a `private` message. This can be
demonstrated as follows:

```ruby
class Quirky
  def send_private_explicitly
    self.say_hello    # => NoMethodError: private method 'say_hello'
  end

  def send_private_implicitly
    say_hello         # => "Hi there friend!"
  end

  private

  def say_hello
    puts "Hi there friend!"
  end
end
```

However, this is not true when sending a message that ends in `=`. In
this case you _must_ use the explicit receiver `self`.

```ruby
class Quirky
  def send_private_setter_explicitly
    self.store_value = 'friend'     # => @private_value='friend'
    store_value                     # => "Let's get our private value"
  end

  def send_private_setter_implicitly
    store_value = 'friend'          # => @private_value is not set
    store_value                     # => 'friend'
  end

  private

  def store_value=(value)
    @private_value = value
  end

  def store_value
    puts "Let's get our private value"
  end
end
```

In fact, without the explicit reciever `self` when sending a private
message ending in `=`, you will instead create a local variable with
that name. This is then the reference point for the rest of the method
body.
