# Objectives

Mongoid is a mature project, with 289 contributors. Today we'll look at some of the tools that the project uses, as well as read some of the code to see how a large & complicated project is structured.

+ Setup the development environment for the Mongoid gem.
+ Familiarize with the tools used by the project.
+ Code dive into Mongoid::Document. Understand how concerns are separated into different modules, and each module in a different file.
+ Play with the Github Events API.

# Install Mongoid From Source (DIY)

Please do the following on your own:

+ Clone Mongoid from Github.
+ Checkout a branch called "explore-mongoid", at the commit tagged by "v4.0.0.beta1"
+ Run the full Rspec to see if all tests pass. (It takes about ~2 minutes on my machine)
+ Build and Install Mongoid gem from source.

# Causing A Fail Test (DIY)

Now you've ensured that all the tests pass, let's try making some changes to the code and break some tests.

+ Change `Mongoid.default_session` to return the string `"banana"`
+ Run the test again, to see that it fails.

You should see,

```
Mongoid
  .default_session
    returns the default session (FAILED - 1)
```

### Hint - Using Ack to Find Code Quickly

To find where the code `Mongoid.default_session` is? Try the `ack` tool.

````
> ack --help
Usage: ack [OPTION]... PATTERN [FILES OR DIRECTORIES]

Search for PATTERN in each source file in the tree from the current
directory on down.  If any files or directories are specified, then
only those files and directories are checked.
```

Let's search for `"def default_session"`

```
> ack "def default_session"
lib/mongoid.rb
58:  #   Mongoid.default_session
63:  def default_session
```

It's at line 63 of the file `lib/mongoid.rb`. Suppose I want to know all places where `default_session` appears,

```
> ack default_session
lib/mongoid/railtie.rb
117:          ::Mongoid.default_session.disconnect if ::Mongoid.configured?
124:            ::Mongoid.default_session.disconnect if forked

lib/mongoid.rb
58:  #   Mongoid.default_session
63:  def default_session

spec/mongoid_spec.rb
34:  describe ".default_session" do
37:      expect(Mongoid.default_session).to eq(Mongoid::Sessions.default)
```

and this search also tells me where the test for `default_session` is.

### Hint - Run Only Selected Specs

It takes a long time to run the full Rspec suite. Often when you are working on something, you only run specs related to what you are doing so it's faster. You can pass in command line options to run only the tests you want.

For example, to run only specs whose description string matches

```
> rspec -e default_session
```

or, to run only specs in a file,

```
> rspec spec/mongoid_spec.rb
```

or, even more specific,

```
> rspec spec/mongoid_spec.rb -e default_session
```

See `rspec --help` for details.

# Mongoid API Doc

Mongoid uses [Yardoc](http://yardoc.org) to generate its API documentation. You can see the generated API doc [HERE](http://rdoc.info/github/mongoid/mongoid).

For example, the `Mongoid.default_session` method has the following API doc,

```ruby
# Convenience method for getting the default session.
#
# @example Get the default session.
#   Mongoid.default_session
#
# @return [ Moped::Session ] The default session.
#
# @since 3.0.0
def default_session
  "banana"
end
```

View a few other source files, and notice how extensively documented Mongoid is. Even the internal API (those not exposed to Mongoid users) are documented.

Also notice that Mongoid uses the `@return` and `@param` annotations to document the types of API. Even though Ruby is a dynamic language, documenting the type information of internal API helps to,

1. Keep the API simple.
2. Make it easier for new contributors to get to know the codebase.
3. Declare contracts between modules as the codebase grows large.



# Explore Project Tools

There's a rich ecosystem of services for projects hosted on Github.

TravisCI - for continuous integration.
Coverall - for test converage report.
CodeClimate - for code health report (complexity & churn).

TravisCI is especially useful to make sure that the code is correct when there are many people contributing to the project. If you look at a pull request for Mongoid, [#3468](https://github.com/mongoid/mongoid/pull/3468), TravisCI would run the full test suite to make sure that it doesn't break anything. #3468 passes the test:

![aok](https://www.evernote.com/shard/s20/sh/048536db-b1f3-4c61-ab73-370c340df617/bf407ae6d028d8af8356bb827803f90f/res/e579a548-e370-47cc-8e0e-cce267712e96/skitch.png)

Another pull reqeust ([#3526](https://github.com/mongoid/mongoid/pull/3526)) has a failed test run:

![fail](https://www.evernote.com/shard/s20/sh/324ed8b3-8253-4cf1-833d-7e2763ccbfd4/86479c23441fc7cc28caf47f50468cc9/res/a29a349f-0ebc-4b58-95a4-e71f9a0def62/skitch.png)

TravisCI's configuration is in `.travis.yml`

```yaml
language: ruby
bundler_args: --without development
rvm:
  - 1.9.3
  - 2.0.0
  - 2.1.0
  - jruby
  - jruby-head
env: JRUBY_OPTS="--server -J-Xms512m -J-Xmx1024m"
matrix:
  allow_failures:
    - rvm: jruby-head
    - gemfile: gemfiles/rails41.gemfile
notifications:
  email: false
  irc:
    on_success: change
    on_failure: always
    channels:
      - "irc.freenode.org#mongoid"
services:
  - mongodb
gemfile:
  - gemfiles/rails40.gemfile
  - gemfiles/rails41.gemfile
```

For *EVERY* commit to the Mongoid project, TravisCI will run the full test suite, once each with different versions of Ruby (1.9.3, 2.0.0, 2.1.0, jruby, jruby-head).

Go take a look at Mongoid's [CodeClimate](https://codeclimate.com/github/mongoid/mongoid) and [Coverage Report](https://coveralls.io/r/mongoid/mongoid?branch=master).

# Explore Mongoid::Document

`Mongoid::Document` is a very complex module. What happens when you create a Mongoid document class like the following?

```ruby
class Person
  include Mongoid::Document
end
```

Let's do it step by step in a pry console. (`ls` command list the instance methods of an object in pry).

```ruby
> pry
[1] pry(main)> class Person
[1] pry(main)* end
[2] pry(main)> ls Person

[3] pry(main)> person = Person.new
=> #<Person:0x007f8562a00db8>
[4] pry(main)> ls person

```

So this is a `Person` class with no instance methods except those inherited from Object. Now let's include `Mongoid::Document` into the `Person` class:

```
[6] pry(main)> require "mongoid"
=> true
[7] pry(main)> Person.send :include, Mongoid::Document
=> Person
ActiveModel::Conversion::ClassMethods#methods: _to_partial_path
ActiveModel::Naming#methods: model_name
Mongoid::Attributes::Nested::ClassMethods#methods: accepts_nested_attributes_for
Mongoid::Attributes::Readonly::ClassMethods#methods: attr_readonly
Mongoid::Attributes::ClassMethods#methods: alias_attribute
Mongoid::Fields::ClassMethods#methods: attribute_names  database_field_name  field  replace_field  using_object_ids?
Mongoid::Indexable::ClassMethods#methods: add_indexes  create_indexes  index  index_specification  remove_indexes
Mongoid::Persistable::Creatable::ClassMethods#methods: create  create!
...
[11] pry(main)> ls person
ActiveModel::Conversion#methods: to_model  to_param  to_partial_path
ActiveModel::Serialization#methods: read_attribute_for_serialization
ActiveModel::Serializers::JSON#methods: as_json  from_json
ActiveModel::Serializers::Xml#methods: from_xml  to_xml
...
Mongoid::Matchable#methods: matches?
Mongoid::Persistable::Creatable#methods: insert
Mongoid::Persistable::Deletable#methods: delete  remove
Mongoid::Persistable::Destroyable#methods: destroy  destroy!
...
```

By including `Mongoid::Document`, the `Person` class now has many methods! Notice that pry tells you from which module which method comes from. For example, `Person#delete` comes from `Mongoid::Persistable::Deletable`, and `Person#destroy` comes from `Mongoid::Persistable::Destroyable`.

Instead of having one huge module that holds all the methods, Mongoid [separates different concerns](http://programmers.stackexchange.com/a/32614) into different modules. This helps to manage the complexity of the project.

You can see all the modules that are mixed into Person:

```ruby
[16] pry(main)> Person.included_modules
=> [Mongoid::Document,
 Mongoid::Composable,
 Mongoid::Equality,
 Mongoid::Stateful,
 Mongoid::Reloadable,
 Mongoid::Inspectable,
 Mongoid::Evolvable,
 ActiveModel::Naming,
 ActiveModel::ForbiddenAttributesProtection,
 Mongoid::Copyable,
 ...
 ]
```

Finally, the Person class is registered to `Mongoid.models`

```
[11] pry(main)> Mongoid.models
=> [Person]
```


# Code Layout of Mongoid::Document (DIY)

Now we know that Mongoid::Document is actually a super module that includes many other modules, let's see how it's actually done.

```ruby
# lib/mongoid/document.rb
module Mongoid
  module Document
    extend ActiveSupport::Concern
    include Composable

    included do
      Mongoid.register_model(self)
    end
```

+ What's the difference between `extend` and `include`?
+ Which source file defines `Mongoid::Composable`?
+ Check that modules included into the `Person` class are included by `Mongoid::Composable`.
+ Are there other modules that include `Mongoid::Composable`? (Hint: use `ack` to search for `Composable`)

`ActiveSupport::Concern` comes from Rails. It's basically the `Module.included` hook with some additional magic. It runs when the module is included into another class. If you want to learn more about it,

+ `ActiveSupport::Concern` [documentation](http://api.rubyonrails.org/classes/ActiveSupport/Concern.html)
+ [Ruby Mixins & ActiveSupport::Concern](http://engineering.appfolio.com/2013/06/17/ruby-mixins-activesupportconcern/)

# Github Event API

We'll take a break from Mongoid and look at the [Github Events API](https://developer.github.com/v3/activity/events). It return an array of Event objects in JSON. There are many [different types of events](https://developer.github.com/v3/activity/events/types/).

You can get the API data by making a HTTP request with curl.

```
> curl https://api.github.com/events
[
  {
    "id": "1978774765",
    "type": "PushEvent",
    "actor": {
      "id": 382747,
      "login": "andrepl",
      "gravatar_id": "411d2b4791a8de51f98666e93e9f1fde",
      "url": "https://api.github.com/users/andrepl",
      "avatar_url": "https://gravatar.com/avatar/411d2b4791a8de51f98666e93e9f1fde?d=https%3A%2F%2Fa248.e.akamai.net%2Fassets.github.com%2Fimages%2Fgravatars%2Fgravatar-user-420.png&r=x"
    },
    "repo": {
      "id": 16670304,
      "name": "andrepl/andrepl.github.io",
      "url": "https://api.github.com/repos/andrepl/andrepl.github.io"
    },
    "payload": {
      "push_id": 307955115,
      "size": 1,
      "distinct_size": 1
...
```

It's also interesting to see what the Github API returns in the HTTP header.

```
> curl -I https://api.github.com/events
HTTP/1.1 200 OK
Server: GitHub.com
Date: Fri, 14 Feb 2014 11:20:53 GMT
Content-Type: application/json; charset=utf-8
Status: 200 OK
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 56
X-RateLimit-Reset: 1392380080
Cache-Control: public, max-age=60, s-maxage=60
Last-Modified: Fri, 14 Feb 2014 11:20:51 GMT
ETag: "245d7d2b5f19b1f34f306879f8f87abc"
X-Poll-Interval: 60
Vary: Accept
X-GitHub-Media-Type: github.beta
Link: <https://api.github.com/events?page=2>; rel="next"
X-Content-Type-Options: nosniff
Content-Length: 70859
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: ETag, Link, X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, X-OAuth-Scopes, X-Accepted-OAuth-Scopes, X-Poll-Interval
Access-Control-Allow-Origin: *
X-GitHub-Request-Id: 6ABA158E:22DD:52D7770:52FDFC13
Vary: Accept-Encoding
```

From the API documentation, you can see how Etag is used:

```
$ curl -I https://api.github.com/users/tater/events
HTTP/1.1 200 OK
X-Poll-Interval: 60
ETag: "a18c3bded88eb5dbb5c849a489412bf3"

# The quotes around the ETag value are important
$ curl -I https://api.github.com/users/tater/events \
    -H 'If-None-Match: "a18c3bded88eb5dbb5c849a489412bf3"'
HTTP/1.1 304 Not Modified
X-Poll-Interval: 60
```

+ What are `X-RateLimit-Limit` and `X-RateLimit-Remaining`? Do they change each time you make an API request?
+ What's the curl `-H` flag for? How would a web browser respond to 304? What's the Etag header for?
+ What might cause X-Poll-Interval to change?
+ How do you get the second page of events?

# Modelling Github Event (DIY)

Having seen the Event API, let's model it with Mongoid. Instead of using the live data, let's use [events.json](https://gist.github.com/hayeah/8970697/raw/dac2fabc176d9da0fe7dcde1ab4b61d908b74b23/events.json) so we all have the same data. Before we start modelling the data, it's useful to take a quick look at `events.json` to see what it's like.

We'll use the `jq` command for our initial data exploration.

([jq](http://stedolan.github.io/jq/manual/) is a tool that lets you query a JSON document. Here we use it to find the unique types of our events. If you are a command-line addict like me, you might be interested in [7 command-line tools for data science](http://jeroenjanssens.com/2013/09/19/seven-command-line-tools-for-data-science.html).)

What Event types do we have?

```
> cat events.json | jq '[.[].type] | unique'
[
  "CreateEvent",
  "GollumEvent",
  "IssueCommentEvent",
  "PullRequestEvent",
  "PushEvent"
]
```

How many events of each type do we have?

```sh
> cat events.json | jq -c 'group_by(.type) | .[] | [.[0].type,length]'
["CreateEvent",1]
["GollumEvent",2]
["IssueCommentEvent",2]
["PullRequestEvent",3]
["PushEvent",22]
```

Do all events have the same fields?

```
> cat events.json | jq '[.[] | keys] | unique'
[
  [
    "actor",
    "created_at",
    "id",
    "org",
    "payload",
    "public",
    "repo",
    "type"
  ],
  [
    "actor",
    "created_at",
    "id",
    "payload",
    "public",
    "repo",
    "type"
  ]
]
```

No. Sometimes an event might have the "org" field.

What fields does the payload of a PushEvent have?

```
> cat events.json | jq '[.[] | select(.type == "PushEvent")] | .[0].payload | keys'
[
  "before",
  "commits",
  "distinct_size",
  "head",
  "push_id",
  "ref",
  "size"
]
```

Check that this agree with Github's [PushEvent spec](https://developer.github.com/v3/activity/events/types/#pushevent).

**Optional exercise**: understand the `jq` examples above.