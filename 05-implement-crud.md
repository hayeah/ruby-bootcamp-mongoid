# Implement MyMongoid CRUD

+ Create a singleton connection object to handle DB queries.
+ Implement `#create` and `#find`

Please follow TDD when you do this assignment. You may reuse the assignment's test cases in your test suite. Remember, you should fail a test first before you code.

## Working With Your Partner

Fork your study partner's my_mongoid repo, and do your assignment there.

+ If you notice bugs, you should fix them and create pull requests (each bug should be fixed in a branch, and have its own pull request).
+ You should negotiate with your partner to decide how much tests you'll write.
+ Follow the [Github Ruby styleguide](https://github.com/styleguide/ruby)
+ Don't read your partner's code until you are done.
+ Review your partner's pull request. If it's good, accept it. If it's not good enough, say why and ask your partner to fix it.
+ **Be kind and helpful** if your partner's code is terrible :)

# Add Moped As Gem Dependency

Add Moped to `my_mongoid.gemspec` as a dependency, so when somebody installs your gem, Moped would also be installed:

```
# my_mongoid.gemspec
spec.add_dependency("moped", ["~> 2.0.beta6"])
```

Do bundle install to make sure that all your development gems are up to date:

```
> bundle
Resolving dependencies...
Using rake (10.1.1)
Using bson (2.2.0)
Using bundler (1.5.3)
Using connection_pool (1.2.0)
Using diff-lcs (1.2.5)
Using optionable (0.2.0)
Using moped (2.0.0.beta6)
Using my_mongoid (0.0.1) from source at .
Using rspec-support (3.0.0.beta1)
Using rspec-core (3.0.0.beta1)
Using rspec-expectations (3.0.0.beta1)
Using rspec-mocks (3.0.0.beta1)
Using rspec (3.0.0.beta1)
Your bundle is complete!
Use `bundle show [gemname]` to see where a bundled gem is installed.
```

Commit your change to gemspec. We are ready to start.

# Configuring Database Session

The first thing we need to do is to provide a way to configure MyMongoid, so it knows how it can connect to MongoDB.

We'll configure MyMongoid this way:

```ruby
MyMongoid.configure do |config|
  config.database = "my_mongoid"
  config.host = "localhost:27017"
end
```

# MongoDB Session Configuration (DIY)

We'll implement session configuration in this section. It'll look like:

```ruby
MyMongoid.configure do |config|
  config.database = "my_mongoid"
  config.host = "localhost:27017"
end
```

This is very commonly seen in Ruby projects. RSpec uses the same configuration mechanism:

```ruby
RSpec.configure do |config|
  config.include Mongoid::SpecHelpers
  config.raise_errors_for_deprecations!

  # Drop all collections and clear the identity map before each spec.
  config.before(:each) do
    Mongoid.purge!
  end

  # Filter out MongoHQ specs if we can't connect to it.
  config.filter_run_excluding(config: ->(value){
    return true if value == :mongohq && !mongohq_connectable?
  })
end
```

The benefit of having a configuration object over plain Ruby hash is that all the configurations are well defined methods. If your user misconfigures, the configuration object would have a chance to report error. For example:

```
MyMongoid.configure do |config|
  config.unknown_config = "crazy value"
end
# => raise NoMethod
```

After MyMongoid is configured, we should be able to get the database session:

```ruby
MyMongoid.session
```

We won't implement multiple sessions in MyMongoid. In later sections, a `MyMongoid` model will get a `Moped::Collection` from the session, and issue queries via the collection.

## Implement Configuration

**Implement** `Mongoid.configure`.

Hint: use the [`Singleton` module](http://ruby-doc.org/stdlib-2.1.0/libdoc/singleton/rdoc/Singleton.html) in Ruby stdlib. See the section "Singleton Object" below.

Pass `rspec crud_spec.rb -e 'Should be able to configure MyMongoid'`

```
Should be able to configure MyMongoid:
  MyMongoid::Configuration
    should be a singleton class
    should have #host accessor
    should have #database accessor
  MyMongoid.configuration
    should return the MyMongoid::Configuration singleton
  MyMongoid.configure
    should yield MyMongoid.configuration to a block
```
Commit your work.

**Implement** `Mongoid.session`.

Pass `rspec crud_spec.rb -e 'Should be able to get database session'`

Hint: Look at the API doc for [`Moped::Session`](https://github.com/mongoid/moped/blob/09b9d91f9204486c56b0e2c69377dffad56dc852/lib/moped/session.rb#L11-L32)
Hint: [The Basics of Ruby Memoization](http://gavinmiller.io/2013/basics-of-ruby-memoization)

```
Should be able to get database session:
  MyMongoid.session
    should return a Moped::Session
    should memoize the session @session
    should raise MyMongoid::UnconfiguredDatabaseError if host and database are not configured
```

Commit your work.

#### Singleton Object

**Singleton**: the [singleton pattern](http://en.wikipedia.org/wiki/Singleton_pattern) is a design pattern that restricts the instantiation of a class to one object.

Often your code might need a global object that is shared by everybody. A common use case for such a global object is a configuration object. The simplest thing to do is:

```ruby
CONFIG = {
  "foo" => 1
  "bar" => 2
}
```

Sometimes you might want something fancier, instead of using a plain hash for configuration, you might have a specialized configuration class:

```
class AppConfig
  # fancy configuration stuff
end
CONFIG = AppConfig.new
```

But there's nothing that prevents you from creating another `AppConfig` instance:

```
CONFIG2 = AppConfig.new
```

Having two instances of `AppConfig` violates the singleton pattern. Ruby provides a the [`Singleton`](http://www.ruby-doc.org/stdlib-1.9.3/libdoc/singleton/rdoc/Singleton.html) module to implement the singleton pattern.

```
class AppConfig
  include Singleton
end
```

You can get the `AppConfig` singleton with the `AppConfig.instance` method:

```
[4] pry(main)> AppConfig.instance
=> #<AppConfig:0x007fb47c834658>
[5] pry(main)> AppConfig.instance
=> #<AppConfig:0x007fb47c834658>
```

However, calling `new` would be an error:

```
[6] pry(main)> AppConfig.new
NoMethodError: private method `new' called for AppConfig:Class
```

Singleton is useful when you want one and only one instance in your program. For example:

+ Singleton database connections pool.
+ Singleton object memory cache
+ Singleton logger

Exercise: read the source code of [`singleton.rb`](https://github.com/ruby/ruby/blob/ba5ed845b30c81fbf92c052b83f54198cd272bbd/lib/singleton.rb)

# Implement Create

To create a new record we'll first use `MyMongoid.session` to get a model's collection, then issue the `insert` query to the model's collection. A model's collection is a `Moped::Collection`.

For the `Event` model, what `Event.create(doc)` would do is roughly:

```
# This is pseudo code!
event = Event.new(doc)
MyMongoid.session.collections[:events].insert(event.to_document)
event.new_record? # => false
```

**Implement** `Model#collection` to return the collection for a model.

Pass `rspec crud_spec.rb -e 'model collection'`

Hint: Use [ActiveSupport's tableize](http://api.rubyonrails.org/classes/ActiveSupport/Inflector.html#method-i-tableize) to convert a class name like "Event" to collection name "events". (Add version 4.0.3 as dependency to your gemspec.) `require "activesupport/inflector"`
Hint: Use [`Moped::Session#[]`](https://github.com/mongoid/moped/blob/master/lib/moped/session.rb#L42-L51) to get a collection by name.

```
Should be able to create a record:
  model collection:
    Model.collection_name
      should use active support's titleize method
    Model.collection
      should return a model's collection
```

**Implement** `#to_document` to "serialize" a model, so it can be saved in MongoDB.

As you've seen before, the `insert` query takes an ordinary Ruby hash:

```ruby
collections[:people].insert({
  first_name: "Heinrich",
  last_name: "Heine"
})
```

So `#to_document` just need to return `#attributes`; it doesn't need to do anything special. We assume that `#attributes` can be converted to bson by calling `#to_bson`.

Pass `rspec crud_spec.rb -e '#to_document'`

```
Should be able to create a record:
  #to_document
    should be a bson document
```

**Implement** `#create` to initialize a new record, then call `#save`, then return the saved record.

Pass `rspec crud_spec.rb -e 'Model#save'`

```
Should be able to create a record:
  Model#save
    successful insert:
      should insert a new record into the db
      should return true
      should make Model#new_record return false
```

Then pass `rspec crud_spec.rb -e 'Model.create'`

```
Should be able to create a record:
  Model.create
    should return a saved record
```

Commit.

## Automatically Generated ObjectId

If you try to save a record and it has no `_id`, then we should generate an id before we insert it into the db.

**Implement** automatically generate an id.

Hint: use `BSON::ObjectId.new` to generate an [uuid](http://en.wikipedia.org/wiki/Universally_unique_identifier).

Pass: `rspec crud_spec.rb -e 'saving a record with no id'`

```
Should be able to create a record:
  saving a record with no id
    should generate a random id
```

Commit.

# Model.find

Recall how Mopeds finds a record by id:

```ruby
collection.find({"_id" => "1"})
```

For our `find` method, we'd want to be able to find a document by using a query:

```
Model.find({"_id" => "abc"})
```

Or using a shorthand:

```
Model.find("abc")
```

## Mongoid::Document#instantiate

Before we implement our own `find`, let's take a look at Mongoid's code. When
Mongoid fetches a record from the database, it doesn't invoke `Model.new` to
get an instance. Instead, Mongoid invokes `Model.instantiate`.

Compare `Model.instantiate` with `Model#initialize` below.

`Model.instantiate`:

```ruby
# Instantiate a new object, only when loaded from the database or when
# the attributes have already been typecast.
#
# @example Create the document.
#   Person.instantiate(:title => "Sir", :age => 30)
#
# @param [ Hash ] attrs The hash of attributes to instantiate with.
# @param [ Integer ] selected_fields The selected fields from the
#   criteria.
#
# @return [ Document ] A new document.
#
# @since 1.0.0
def instantiate(attrs = nil, selected_fields = nil)
  attributes = attrs || {}
  doc = allocate
  doc.__selected_fields = selected_fields
  doc.instance_variable_set(:@attributes, attributes)
  doc.apply_defaults
  yield(doc) if block_given?
  doc.run_callbacks(:find) unless doc._find_callbacks.empty?
  doc.run_callbacks(:initialize) unless doc._initialize_callbacks.empty?
  doc
end
```

`Model#initialize`

```
# Instantiate a new +Document+, setting the Document's attributes if
# given. If no attributes are provided, they will be initialized with
# an empty +Hash+.
#
# If a primary key is defined, the document's id will be set to that key,
# otherwise it will be set to a fresh +BSON::ObjectId+ string.
#
# @example Create a new document.
#   Person.new(:title => "Sir")
#
# @param [ Hash ] attrs The attributes to set up the document with.
#
# @return [ Document ] A new document.
#
# @since 1.0.0
def initialize(attrs = nil)
  _building do
    @new_record = true
    @attributes ||= {}
    with(self.class.persistence_options)
    apply_pre_processed_defaults
    apply_default_scoping
    process_attributes(attrs) do
      yield(self) if block_given?
    end
    apply_post_processed_defaults
    # @todo: #2586: Need to have access to parent document in these
    #   callbacks.
    run_callbacks(:initialize) unless _initialize_callbacks.empty?
  end
end
```

+ `#initialize`'s self is the model instance. `instantiate`'s self is the model class.
+ `#initialize` sets `@new_record` to be true.
+ `#initialize` uses `#process_attributes`, and `instantiate` sets `@attributes` directly.
+ Both invokes the `initialize` lifecycle callback. (Is this surprising?)
+ The two methods handle default values differently (we'll ignore that).
+ `instantiate` may return a document whose selected fields is only a subset of the complete document. (maybe you want 3 out of 10 fields).

Why does Mongoid have two methods to do essentially the same thing? Would it be better to unify the API this way:

```ruby
def initialize(attrs,selected_files=nil,is_find=false)
  if is_find
    ...
  else
    ...
  end
end
```

It's probably not better, because `initialize` is a "public API", exposed to Mongoid users. `instantiate` is a "private API", used only within the Mongoid project. If the public API and the private API are the same method, a Mongoid user who accidentally uses `Model.new` the wrong way might be surprised instead of getting an error:

```ruby
Event.new({"id" => "1000"},{"user" => "clearly didn't read the documentation"})
# no error
```

Whereas by separating the private API into its own the user gets an error,

```ruby
Event.new({"id" => "1000"},{"user" => "clearly didn't read the documentation"})
# error
```

## Implement Find

**Implement** `Model.instantiate(attributes)` to return a model instance. Later `find` will query the database to find the attributes for a document.

Pass: `rspec crud_spec.rb -e 'Model.instantiate'`

```
Should be able to find a record:
  Model.instantiate
    should return a model instance
    should return an instance that's not a new_record
    should have the given attributes
```

**Implement** `Model.find`. It looks up the database for attributes, and use instantiate to return a model instance.

Pass: `rspec crud_spec.rb -e 'Model.find'`

```
Should be able to find a record:
  Model.find
    should be able to find a record by issuing query
    should be able to find a record by issuing shorthand id query
    should raise Mongoid::RecordNotFoundError if nothing is found for an id
```

# Pull Request

Push your code to your own clone, and create a pull request to your partner's.
