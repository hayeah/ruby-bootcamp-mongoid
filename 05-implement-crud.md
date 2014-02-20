To improve:

+ better higher level implementation plan. answer "why" we are implementing this, and how it might tie into later steps
+ implementation notes: what you might need to know to actually implement this


# Implement MyMongoid CRUD

+ Create a singleton connection object to handle DB queries.
+ Implement CRUD for MyMongoid

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

We'll implement a very common configuration pattern in Ruby:

```ruby
MyMongoid.configure do |config|
  config.database = "my_mongoid"
  config.host = "localhost:27017"
end
```

Then we should be able to get the database session:

```ruby
MyMongoid.session
```

We won't implement multiple sessions in MyMongoid, so `MyMongoid.session` is a _singleton_ of `Moped::Session`. In later sections, the model will get a `Moped::Collection` from the session, and issue queries via the collection.



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

The benefit of having a configuration object over plain Ruby hash is that all the configurations are well defined methods. If you misconfigure, the configuration object would have a chance to report error. For example:

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

## Implementation

Implement `Mongoid.configure`.

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

Implement `Mongoid.session`.

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

Singleton is useful when you want only one thing in your program. For example:

+ Singleton database connections pool.
+ Singleton object memory cache
+ Singleton logger

# MyMongoid.create (DIY)

To create a new record we'll first use `MyMongoid.session` to get a model's collection, then issue the `insert` query to the model's collection. A model's collection is a `Moped::Collection`.

For the `Event` model, `Event.create(doc)` is roughly like:

```
event = Event.new(doc)
MyMongoid.session.collections[:events].insert(event.to_document)
event.new_record? # => false
```

Pass `rspec crud_spec.rb -e 'model collection'`

Hint: Use [ActiveSupport's tableize](http://api.rubyonrails.org/classes/ActiveSupport/Inflector.html#method-i-tableize) to convert a class name like "Event" to collection name "events". (Add version 3.0.0 as dependency to your gemspec.) `require "active_support/inflector"`
Hint: Use [`Moped::Session#[]`](https://github.com/mongoid/moped/blob/master/lib/moped/session.rb#L42-L51) to get a collection by name.

```
Should be able to create a record:
  model collection:
    Model.collection_name
      should use active support's titleize method
    Model.collection
      should return a model's collection
```

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

`#create` should initialize a new record, then call `#save`, then return the saved record.

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

If you try to save a record and it has no `_id`, then we should generate a random id before we insert it into the db.

Hint: use `BSON::ObjectId.new` to generate an [uuid](http://en.wikipedia.org/wiki/Universally_unique_identifier).

Pass: `rspec crud_spec.rb -e 'saving a record with no id'`

```
Should be able to create a record:
  saving a record with no id
    should generate a random id
```

Commit.

# Find

Event.find("abc") == Event.find({"_id" => "abc"})

# Update

event.update_attributes({}) # use $set

should do mass-assignment first

# Save

Model.dirty_fields
dirty_tracking. hook into write_attribute. reset @dirty_fields upon save.

# Delete

event.delete