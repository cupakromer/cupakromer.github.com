---
layout: post
title: "The Bang is for Surprise"
date: 2013-05-19 21:20
comments: false
categories:
- ruby
- rspec
---

**_Note:_** I'm using Ruby 2.0.0 and RSpec 2.13.1 for these samples. Behavior
may be slightly different in older versions. YMMV!

One of the more well known features of [RSpec](github.com/rspec/rspec-core) is
[`let`](https://www.relishapp.com/rspec/rspec-core/v/2-13/docs/helper-methods/let-and-let!).
It provides a way to create a variable as a
[bareword](http://devblog.avdi.org/2012/10/01/barewords/) which is lazy loaded
and memoized.

It also has a sibling `let!`. On the surface, `let!` is just a `let` without
the lazy loading. So any variable defined with a `let!` will always be created
before a test. This tool has a few nuances that should be know before you reach
for it.

Take the following sample. _What do you think the result will be?_

```ruby
describe "Which one wins?" do

  let!(:sample) { "using let bang!" }

  context "inside a context" do
    let(:sample) { "normal let" }

    it { expect(sample).to eq "normal let" }
  end

end
```

The test passes. This may or may not have been surprising. The rules for nested
`let`s, is that the `let` defined closet to the test in the hierarchy wins.
But, didn't I say `let!` always created the object which was then memoized?

I'll get to more about how `let` and `let!` are implemented in a minute. For
now, I want to point out a subtle surprise waiting for you; or possibly bring
to light that nagging itch at the back of your brain.

```ruby
describe "Using Conflicting Let and Let!" do

  let!(:user) { create :user, name: 'bob' }

  # LOTS OF CODE SO YOU CAN'T SEE THE ABOVE LINE

  context "inside a context" do
    let(:user) { create :user, name: 'alice' }

    # Will this pass?
    it do
      expect(User.count).to eq 0
    end
  end

end
```

Surprise! The test fails:

```text
  1) Which one wins? inside a context
     Failure/Error: expect(User.count).to eq 0

       expected: 0
            got: 1

       (compared using ==)
```

[**WAT?**](https://www.destroyallsoftware.com/talks/wat)

`user` was never explicitly referenced in our test or a `before` block. Above I
also stated that the `let` closest to the test _wins_. Theoretically, by these
rules one would naturally think the test would have passed. Yet, someone was
created in the database.

_Which user definition do you think was created?_

If we dump the `User` collection before the `expect` line we see:

```ruby
[<User id: 1, name: "alice", ...>]
```

Not only did the inner normal `let` block appear to override the outer, the
outer `let!` behavior took affect!

Let's try one more:

```ruby
describe "Using Conflicting Nested Let!" do

  let!(:user) { create :user, name: 'bob' }


  context "inside a context" do
    # Now we'll use the bang version here
    let!(:user) { create :user, name: 'alice' }

    # Will this pass?
    it do
      expect(User.count).to eq 1
    end
  end

end
```

Surprise! It passes:

```text
Using Conflicting Nested Let!
  inside a context
    should eq 1

Finished in 0.02185 seconds
1 example, 0 failures
```

Again, dumping the user created we see our good friend Alice:

```ruby
[<User id: 1, name: "alice", ...>]
```

If you're scratching your brain right now. Don't worry. I did too the first
time.  However, once we cover what is happening behind that curtain things will
make perfect sense.


## How `let` and `let!` Work

"**_Pay ~~[no]~~ attention_** to that man behind the curtain!"
- [The Wizard](http://en.wikiquote.org/wiki/The_Wizard_of_Oz#The_Wizard)

The reason for this behavior, remember I'm using RSpec 2.13, is that `let!`
just calls `let` and `before` [behind the scenes](https://github.com/rspec/rspec-core/blob/5baa615d/lib/rspec/core/memoized_helpers.rb#L257):

```ruby
def let!(name, &block)
  let(name, &block)
  before { __send__(name) }
end
```

And all
[`let`](https://github.com/rspec/rspec-core/blob/5baa615d/lib/rspec/core/memoized_helpers.rb#L195)
does is setup a memoized method based on the name and provided block:

```ruby
MemoizedHelpers.module_for(self).send(:define_method, name, &block)

define_method(name) do
  __memoized.fetch(name) { |k| __memoized[k] = super(&nil) }
end
```

### _"Using Conflicting Let and Let!"_ Explained

Going back to the example _"Using Conflicting Let and Let!"_ above, where both
`let!` and `let` were used. It should be a bit clearer what is really going on.

When the test runs, the `let!` has already created the `before` block, which
will send the message `:user`. However, the inner context's `let` created a new
method with the same name. Thus based on standard Ruby method lookup, when the
`before` block runs the inner method receives the message:

```ruby
describe "Using Conflicting Let and Let!" do

  # Expanding the following out:
  # let!(:user) { create :user, name: 'bob' }
  def user
    create :user, name: 'bob'
  end
  before{ user }

  context "inside a context" do
    # Expanding the following out:
    # let(:user) { create :user, name: 'alice' }
    def user
      create :user, name: 'alice'
    end

    # The outer `before` block will run before this example.
    # Due to the examples being objects, the inner
    # `def user` will receive the `:user` message sent
    # by `before`.
    it do
      expect(User.count).to eq 0
    end
  end

end
```

### _"Using Conflicting Nested Let!"_ Explained

It should also start to make sense what was going on with the _"Using
Conflicting Nested Let!"_ example:

```ruby
describe "Using Conflicting Nested Let!" do

  # Expanding the following out:
  # let!(:user) { create :user, name: 'bob' }
  def user
    create :user, name: 'bob'
  end
  before{ user }

  context "inside a context" do
    # Now we'll use the bang version here
    # Expanding the following out:
    # let!(:user) { create :user, name: 'alice' }
    def user
      create :user, name: 'alice'
    end
    before{ user }

    # The outer `before` block will run before this example.
    # Due to the examples being objects, the inner
    # `def user` will receive the `:user` message sent
    # by the outer `before`.
    #
    # The inner `before` block will run next, also sending
    # the message `:user`. This is also received by the
    # inner example object. However, since `let` is also
    # memoized, this doesn't actually execute the `:create`.
    # It just returns the already created object.
    it do
      expect(User.count).to eq 1
    end
  end

end
```

### It's Just a Method

I hope that helps demystify the behavior.

Since `let` is just a helper for setting up methods on the [example group object](http://interblah.net/how-rspec-works)
you can call `super` in it; though this is generally not an advised practice.

```ruby
describe "Just a Method" do

  let!(:sample) { "using let bang!" }

  context "using super()" do
    let(:sample) {
      p super()
      "normal let"
    }

    it { expect(sample).to eq "normal let" }
  end

end
```

```text
Just a Method
  using super()
"using let bang!"
    should eq "normal let"
```

## Avoiding Ambiguity

Since that tiny little `!` can be hard to see, especially in a sea of `let`
declarations, it is easy to miss it and get surprised. Additionally, seeing as
how mixing `let!` and `let` can lead to some surprises, it's fairly clear why
`let!` has started, rightly so in my opinion, to fall out of favor with some of
the RSpec crowd.

Luckily, there should be very few situations you should find yourself in where
you want to reach for `let!` over `let`. For me, this is usually a situation
where I'm creating some sort of persisted resource. For instance, the factory
example, or creating a test file on the system.

## Options For Preloading

If people are moving away from using `let!`, how should you preload variables?

### Reference in `before`

Call them just like RSpec does in a `before`:

```ruby
let(:bob)   { create :user, name: 'bob'   }
let(:alice) { create :user, name: 'alice' }

before do
  bob
  alice
end
```

To me this looks a bit odd. People I've talked to tend to have two reactions:

  1. Why are you referencing a _'variable'_ and not using it?
  2. Wouldn't it be a bit more explicit to show the creation using an `@var`?

By now you should know that the first response indicates a lack of understand on
how RSpec works. They aren't variables, they are actually bareword messages.

The second response is a valid point. The result would be:

```ruby
before do
  @bob   = create :user, name: 'bob'
  @alice = create :user, name: 'alice'
end
```

This goes back to preference and style. My preference is to reach for a
bareword whenever I can. One reason is that, when using an instance variables
you are now locked in to how both `@bob` and `@alice` are created. If you later
wanted to modify them, you could but at the expense of already having created
the persisted resource. Remember `before` blocks execute outside-in (this isn't
so much of an issue for lightweight objects). Or you have to roll your own
memoization scheme (not hard just duplication of work).

### Use a method

The next common thing I see done is people say: _"I'll just wrap it all up in a
method."_

```ruby
def create_users
  @bob   = create :user, name: 'bob'
  @alice = create :user, name: 'alice'
end

before { create_users }
```

Now the `before` looks better; it's explicit what is happening. However, you've
just added one level of indirection Also, the new `create_users` method looks
just like our old `before`. The main advantage here is if we need to change the
behavior we can just write a new `create_users` method in an inner context. We
could also use barewords by making our variables into methods:

```ruby
def bob
  @bob ||= create :user, name: 'bob'
end

def alice
  @alice ||= create :user, name: 'alice'
end

def create_users
  bob
  alice
end

before { create_users }
```

Though now we've duplicated the lazy loading and memoizing logic already
provided by `let`.

At this point, you'll probably say, we can make this a bit more explicit and
clean it up at the same time:

```ruby
def bob
  @bob ||= create :user, name: 'bob'
end

def alice
  @alice ||= create :user, name: 'alice'
end

def create_users(*users)
  users.each{ |user| public_send user }
end

before { create_users :bob, :alice }
```

This brings me to my next option.

### Explicit Preload

Now there's nothing inherently wrong with the above methods. However, to me
they add a lot of work, without adding much additional value. There are still
cases where I'll break out the generator method as it's a very useful tool. But
this section is about another option, so I'll get to it.

Having gone through the cycle of improvement the _"hard"_ way, it's time to
show you the shortcut. To me, this is reminiscent of high school calculus
class where the teacher made me do everything the difficult, time consuming
way, for a week before teaching how it's usually done with the shorter method.

Since pretty much everything in RSpec is already just a method, we can leverage
that to get our desired behavior. This was discussed in a [pull request](https://github.com/rspec/rspec-core/pull/815#issuecomment-14446077):

```ruby
module LetPreloadable
  def preload(*names)
    before do
      names.each { |name| __send__ name }
    end
  end
end

RSpec.configure do |rspec|
  rspec.extend LetPreloadable
end
```

You can place the module code anywhere you want (usually in `spec/support`).
Then you'll load it in a `RSpec.configure` block either in the same file or in
`spec_helper.rb`.

Our setup now looks like:

```ruby
let(:bob)   { create :user, name: 'bob'   }
let(:alice) { create :user, name: 'alice' }

preload :bob, :alice
```

Going back to our original example. There is now more context to what is
happening without the confusing mix of `let` and `let!`:

```ruby
describe "No More Conflicting Let and Let!" do

  let(:user) { create :user, name: 'bob' }

  preload :user

  context "inside a context" do
    let(:user) { create :user, name: 'alice' }

    it do
      expect(User.count).to eq 1
    end
  end

end
```

## Introducing [`Conjurer`](https://github.com/cupakromer/conjurer) Gem

I've started using this in enough new projects that I wanted an easy way to
just add it. I also wanted to be able to include any changes easily. Thus, I've
rolled it all up into a gem: [`conjurer`](http://rubygems.org/gems/conjurer)


**Happy RSpecing!!**
