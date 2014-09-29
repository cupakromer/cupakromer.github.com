---
layout: post
title: "Farewell JSON API gems"
date: 2014-09-29 11:51
comments: false
categories:
- ruby
- rspec
- json
---

In the past, testing JSON APIs tended to be a bit painful for me. Most of this
pain revolved around setting expectations on the response body.

If you treat the response as a raw string, attempting to use regular
expressions ends up being an exercise in how you handle frustration. While a
JSON body is a string, it has structure. Using regular expressions for parsing
them is akin to using a hammer on a screw. It'll get the job done, but it's the
wrong tool for the job.

Ruby gives us [`JSON.parse`](http://ruby-doc.org/stdlib-2.1.3/libdoc/json/rdoc/JSON.html#method-i-parse).
Which will convert a valid JSON string into a more familiar object structure.
Now comes the "fun" part of actually verifying that structure:

- Sometimes you only care about part of the response
- Sometimes you care about validating the entire response
- Sometimes the response is very complicated consisting of many smaller, more
  logically meaningful, structures
- Sometimes you only care about the general structure (e.g. this value must be
  a number, that value must be either an empty array or an array of strings,
  etc.)

It is possible to do all of these validations out of the box. In my experience,
writing them tended to be tedious. Often the resulting code left something to
be desired in terms of readability. This was especially true when validating the
general response structure.

I like to follow the "one expectation per spec" guideline. However, this lead
to writing many small specs. Normally, this is perfectly fine and something I
advocate you do. However, in terms of a JSON response, it means I need to have
more discipline to keep everything explicitly organized.

Naturally in the Ruby community, many gems have sprouted up to help with this
problem set. I've had a bit of success with some of those gems in the past.
However, with the release of RSpec 3, [several new
features](http://myronmars.to/n/dev-blog/2014/01/new-in-rspec-3-composable-matchers)
have eliminated my need for these JSON gems.

Expectations on a JSON response is a great fit for [composing
matchers](https://www.relishapp.com/rspec/rspec-expectations/v/3-0/docs/composing-matchers).
When I need to logically group checking several options, the [compound matchers](https://www.relishapp.com/rspec/rspec-expectations/v/3-0/docs/compound-expectations)
are the perfect tool.

Often people don't realize that the matcher messages (i.e. `exist`, `be`, `eq`,
`include`, etc) are just factories. They are just helper methods which create
the matcher object for you. That means, we can easily write our own using our
app's domain language.

Let's jump right into an example!

These examples are assuming a JSON structure like one of the ones listed on the
[jsonapi.org](http://jsonapi.org/format/#document-structure-compound-documents)
site. Though I am assuming integer value are represented as numbers and not
strings, since that is valid JSON and more meaningful:

```ruby
# Use common JSON helpers such as: `json_response`, `be_an_empty`, `all_match`
require 'support/json_api_helpers'

def be_kits_root_json
  be_kits_json.and(
    include(
      'meta' => {
        'first'   => anything,
        'last'    => anything,
        'current' => anything,
      }
    )
  )
end

def be_kits_json
  include(
    'version' => '1.0',
    'links'   => {
      'kits.beacons'       => "#{beacons_url}/{kits.beacons}",
      'kits.overlays'      => "#{overlays_url}/{kits.overlays}",
      'beacons.attributes' => "#{beacon_attributes_url}/{beacons.attributes}",
    },
    'kits'    => be_an_empty(Array).or(
      all_match(
        'id'        => Fixnum,
        'name'      => be_nil.or(be_a String),
        'api_token' => String,
        'account'   => be_nil.or(
          match(
            'id'   => Fixnum,
            'name' => be_nil.or(be_a String),
          )
        ),
        'links'     => {
          'self'     => /\A#{kits_url}\/\d+\z/,
          'beacons'  => be_an_empty(Array).or(all be_a Fixnum),
          'overlays' => be_an_empty(Array).or(all be_a Fixnum),
        },
      ),
    ),
  )
end

def include_linked_resources(*resources)
  resource_maps = resources.each_with_object({}) { |resource, mappings|
    mappings.store(resource.to_s, be_an(Array))
  }
  include('linked' => resource_maps)
end

context "a basic user", "with a kit having no beacons or maps" do
  # Setup world state

  describe "requesting the kits root" do
    it "conforms to the expected JSON structure" do
      get kits_path, *options
      expect(json_response).to be_kits_root_json
    end

    # More specific specs
  end

  describe "requesting a kit" do
    it "conforms to the expected JSON structure" do
      get kit_path(kit), *options
      expect(json_response).to be_kits_json
    end

    # More specific specs
  end
end

# More state specs

context "a developer user", "sending request with parameter 'include'" do
  # Setup world state

  describe "requesting the kits root" do
    it "conforms to the expected JSON structure with included resources" do
      get kits_path(include: "beacons,beacon_attributes"), *options
      expect(json_response).to be_kits_root_json.and(
        include_linked_resources(:beacons, :beacon_attributes)
      )
    end
  end

  describe "requesting a beacon" do
    it "conforms to the expected JSON structure with included resources" do
      get kit_path(kit, include: "beacons,beacon_attributes"), *options
      expect(json_response).to be_kits_json.and(
        include_linked_resources(:beacons, :beacon_attributes)
      )
    end
  end
end
```

The possibilities are fairly endless. We could improve this further by allowing
the factories to take model instances or attribute hashes. We can use those to
check specific content when available:

```ruby
def account_resource(account = nil, allow_nil: false)
  return nil unless account || !allow_nil
  if account
    {
      'id'   => account.id,
      'name' => account.name
    }
  else
    {
      'id'   => Fixnum,
      'name' => be_nil.or(be_a String),
    }
  end
end

def kit_resource(kit = nil, allow_nil: false)
  return nil unless kit || !allow_nil
  if kit
    {
      'id'        => kit.id,
      'name'      => kit.name,
      'api_token' => kit.api_token,
      'account'   => account_resource(kit.account, allow_nil: true),
    }
  else
    {
      'id'        => Fixnum,
      'name'      => be_nil.or(be_a String),
      'api_token' => String,
      'account'   => be_nil.or(match account_resource),
    }
  end
end

context "a basic user", "with a kit having no beacons or maps" do
  # Setup world state

  describe "requesting the kits root" do
    it "conforms to the expected JSON structure" do
      get kits_path, *options
      expect(json_response).to be_kits_root_json
    end

    it "has only the expected kit" do
      get kits_path, *options
      expect(json_response).to include 'kits' => [kit_resource(basic_users_kit)]
    end
  end

  describe "requesting a beacon" do
    it "conforms to the expected JSON structure" do
      get kit_path(kit), *options
      expect(json_response).to be_kits_json(basic_users_kit)
    end
  end
end
```

Happy RSpec'ing!
