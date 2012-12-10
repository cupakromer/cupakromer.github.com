---
layout: post
title: "How Basic RSpec Works - Simplified"
date: 2012-12-09 20:15
comments: false
categories:
- ruby
- rspec
---
Recently, I had the privilege of giving a talk regarding
[RSpec](http://rspec.info) at the
[Arlington Ruby](http://www.meetup.com/Arlington-Ruby/) meetup group.
The talk was about RSpec and how you can take some next steps with it
(slides here: [http://rspec-next-steps.herokuapp.com](http://rspec-next-steps.herokuapp.com).

The talk was targeted at intermediate RSpec users. There were several in
attendance whom were fairly new to RSpec. This made some of the talk
seem like "magic". Based on the questions I received, I wanted to take a
moment to address some of the general workings of RSpec, in order to
dispel any "magic" that may seem to be happening.

It is my hope to try to show how _some_ of this works. I won't be
covering any of the more advanced topics just yet, as the code can get a
bit complicated, and the point here is to simplify how RSpec works. So
bare with me and the overly simplistic implementation. The full code is
available in the following GitHub gist:
[https://gist.github.com/4247624](https://gist.github.com/4247624)

First, we need to setup some methods that define the basic usage of
RSpec: `describe`, `subject`, `before`, `after`, and `it`.

First up is the outer `describe`:

```ruby
def describe(object_under_test, &block)
  @obj_under_test = object_under_test
  @test_group     = block
end
```

In this example we are specifically only supporting a single `describe`
block with no nesting. The reason here is for simplicity.  In our case
`describe` is just a method that takes an object or description string
and block. Nothing special. Hopefully, this helps make it clear that we
are only dealing with a DSL for creating / setting up examples. We'll
store away the test object in an instance variable and keep a reference
to the block for later.

Next, we setup a method for gaining access to subject:

```ruby
def subject
  if @obj_under_test.is_a? Class
    @subject ||= @obj_under_test.new
  else
    @obj_under_test
  end
end
```

If our subject is a `class`, we create a new memoized instance of it.
Otherwise, we simply return the object itself.

Next is the commonly used `before` blocks and the associated `after`
friend:

```ruby
def before(&block) @before_hooks << block end
def after(&block)  @after_hooks  << block end
```

In this simplistic implementation, it is easy to see how they work. We
just keep a reference to all of the block in a normal array (in Ruby the
order of insertion is preserved) for use later. We do the same with
`after`.

Last up, is the real meat of the examples, the `it` block:

```ruby
def it(description = nil, &block)
  @examples << Example.new(description, block)
end

Example = Struct.new(:description, :test) do
  attr_accessor :result, :failure
  def call
    @result = if test
                begin
                  test.call
                  :passed
                rescue => e
                  @failure = e
                  :failed
                end
              else
                :pending
              end
  end
end
```

As with the `describe` method, this takes an optional description and a
block to execute (our actual test). We'll need access to both of these
later, and we'll have multiple examples, so we'll use a simple object to
keep track of each.  After creating the example object, we'll store it
in the queue.

At this point it is worth taking a moment to discuss `Example#call`. We
named it `call` so that accessing it is no different than a traditional
block. This makes it easier to change code later.

Inside `Example#call`, we attempt to pass `call` on the block that the
`Example` was created with. If this raises an error, we store the
exception for access later and mark the test as `failed`. Something to
note here is that the return value of the `test` block is ignored. A
test is marked as `passed` as long as it does not `raise` any errors.
This is also how RSpec behaves.

If no block was given when we created the `Example`, then we treat it as
`pending`.  I have omitted the `pending` method, common in RSpec due to
the complexity it would add to this example.

Something else to note, since this is an overly simplistic example, we
are doing everything in the global main namespace. RSpec does _not_
behave this way, but it helps keep our example simple. Due to this,
we'll need to setup some of our variables:

```ruby
@before_hooks = []
@after_hooks  = []
@examples     = []
```

Additionally, we don't have any matchers defined yet. To keep it simple,
I'll add a TestUnit style `assert` matcher.

```ruby
def assert(truthy)
  truthy or raise Error.new("Test failed.")
end
```

At this point, I hope you can start to get the picture of how our
simplified example will run.  We'll setup our `run` as follows:

```ruby
def run
  raise 'No object under test defined.' unless @obj_under_test

  puts @obj_under_test
  return unless @test_group

  # Find out what tests need to be run
  @test_group.call

  # Run the tests
  @examples.each_with_index do |example, index|
    @subject = nil
    puts "\n\nEXAMPLE #{index+1}:"
    begin
      @before_hooks.each(&:call)
      example.call
      puts "\n    #{example.description} => #{example.result.upcase}\n"
    rescue => e
      puts "\n    #{example.description} => failed\n"
    ensure
      @after_hooks.reverse_each(&:call)
    end
  end
end
```

If there is no object under test defined when we `run` (i.e. `describe`
was never called) then we raise an error. Otherwise, we output the
object under test. If this is a `class` (as is usual for a top level
`describe` block) then we will see the class name. Otherwise, the object
itself is output.  If it is a string, we'll get the string value,
otherwise, we'll get the object's `#to_s` representation.

_It should be noted that in the real RSpec this outputting is much more
complicated and left up to various output formatters._

Next we will run the `test_group` (the body of the `describe` block). This
in turn call all our `before`, `after`, and `it` methods, which set up our
environment and define the examples.

All that is left, is to iterate over the examples and run them. Here I'm
using `each_with_index` solely to be able to add some debugging output
to make it a bit clearer how things are running. Normally, this would be
a simple `each` iterator.

Before each test run, we make sure we have a new empty `subject`. We
then iterate through each of the `before` blocks in the order they were
defined. At this point we run the example. After the example runs, I'm
immediately
outputting the results to keep things simple. In the real RSpec, this is
handled by an output formatter. Finally, all of the `after` hooks are
run, but in reverse defined order.

It should be noted, that here, as in the real RSpec, if any of the
`before` blocks throw an exception the test fails. However, failures in
any `after` block are ignored.

That's it. We can then use this to define our sample spec:

```ruby
class Thing
  attr_reader :my_value
  def initialize
    @my_value = rand 5
  end
end

describe Thing do
  before { puts "BEFORE_BLOCK:: First before block" }
  after do
    print "AFTER_BLOCK:: should be called last!"
    print "    Reset @tmp_value(#{@tmp_value.inspect}) => "
    @tmp_value = nil
    puts "@tmp_value(#{@tmp_value.inspect})"
  end

  it 'has access to subject' do
    p subject
    assert subject.my_value < 5
  end

  it 'subject changes only between tests' do
    p subject
    assert subject.equal?(subject)
  end

  it "fails on error" do
    raise Error.new 'Sad face'
  end

  it 'works!' do
    assert @tmp_value == 'test'
  end

  before do
    @tmp_value = 'test'
    print "BEFORE_BLOCK:: Another before block"
    puts "     Set @tmp_value(#{@tmp_value.inspect})"
  end

  it 'is pending'

  after { puts "AFTER_BLOCK:: should be called first!!" }
end
```

I hope it is clear how this spec will end up running. This is not
exactly equivalent to how RSpec will treat things (notice that we need
to explicitly clear `@tmp_value` in an `after` block, where RSpec will
do that for us). This is due to how RSpec creates example classses
(which we are not using) and how it binds the blocks to different
scopes; we are strictly using the `global` namespace to keep the example
simple.

Check out the gist for the code and output of the sample spec:
[http://gist.github.com/4247624](http://gist.github.com/4247624)

Stay tuned for more on RSpec in the future.
