# Objectives

+ Use Mongoid to model Github Event
+ Read Mongoid code
+ Implement a simple Document model for MyMongoid.

# Model Github Event With Mongoid

Let's model the following Github event:

```
data = {
 "id"=>"1978774765",
 "type"=>"PushEvent",
 "actor"=>
  {"id"=>382747,
   "login"=>"andrepl",
   "gravatar_id"=>"411d2b4791a8de51f98666e93e9f1fde",
   "url"=>"https://api.github.com/users/andrepl",
   "avatar_url"=>
    "https://gravatar.com/avatar/411d2b4791a8de51f98666e93e9f1fde?d=https%3A%2F%2Fa248.e.akamai.net%2Fassets.github.com%2Fimages%2Fgravatars%2Fgravatar-user-420.png&r=x"},
 "repo"=>
  {"id"=>16670304,
   "name"=>"andrepl/andrepl.github.io",
   "url"=>"https://api.github.com/repos/andrepl/andrepl.github.io"},
 "payload"=>
  {"push_id"=>307955115,
   "size"=>1,
   "distinct_size"=>1,
   "ref"=>"refs/heads/master",
   "head"=>"9487ef037120e52091bd5f59d4e2fc983188035f",
   "before"=>"ceac7512abbc515f90797ebb5644b7ced52588be",
   "commits"=>
    [{"sha"=>"9487ef037120e52091bd5f59d4e2fc983188035f",
      "author"=>
       {"email"=>"andre.leblanc@webfilings.com", "name"=>"Andre LeBlanc"},
      "message"=>"more style",
      "distinct"=>true,
      "url"=>
       "https://api.github.com/repos/andrepl/andrepl.github.io/commits/9487ef037120e52091bd5f59d4e2fc983188035f"}]},
 "public"=>true,
 "created_at"=>"2014-02-13T03:20:37Z"
}
```

It's very simple:

```
class Event
  include Mongoid::Document

  field "type"
  field "actor"
  field "org"
  field "payload"
  field "public"
  field "repo"
  field "created_at"
end
```

Notice that "_id" is automatically declared:

```ruby
[16] pry(main)> Event.fields.keys
=> ["_id", "type", "actor", "org", "payload", "public", "repo", "created_at"]
```

And that "id" is an alias for "_id", so the `id` of data is actually assigned to `_id`:

```ruby
[1] pry(main)> Event.aliased_fields
=> {"id"=>"_id"}
[2] pry(main)> event = Event.new("id" => "abc")
=> #<Event _id: abc, type: nil, actor: nil, org: nil, payload: nil, public: nil, repo: nil, created_at: nil>
[3] pry(main)> event.id
=> "abc"
[4] pry(main)> event._id
=> "abc"
[5] pry(main)> event.id == event._id
=> true
```

To instantiate a new event record (without saving to database):

```ruby
event = Event.new(data) do |event|
  event.created_at = Time.parse(data["created_at"])
end
```

We'll implement the same thing.

# Mongoid Implementation

+ Model initialization is in [`Mongoid::Document`](https://github.com/hayeah/mongoid/blob/5b0f031992cbec66d68c6cb288a4edb952ed5336/lib/mongoid/document.rb#L103)
  + in particular, look at [`process_attributes`](https://github.com/hayeah/mongoid/blob/5b0f031992cbec66d68c6cb288a4edb952ed5336/lib/mongoid/attributes/processing.rb#L8-L18)
+ The `field` keyword is implemented in [`Mongoid::Fields`](https://github.com/hayeah/mongoid/blob/5b0f031992cbec66d68c6cb288a4edb952ed5336/lib/mongoid/fields.rb)
+ Attributes setters/getters are in [`Mongoid::Attributes`](https://github.com/hayeah/mongoid/blob/5b0f031992cbec66d68c6cb288a4edb952ed5336/lib/mongoid/attributes.rb)

# Implementation Plan

Here's a high-level overview of what we'll do.

+ Use the `included` hook to add ORM functionality to a plain Ruby class.
+ Store record attributes as a hash, in the instance variable `@attributes`.
+ Implement the `field` keyword as a class method.
+ The `field` class method should generate setters and getters for attributes.

I've broken the work down into small steps, and each step has a corresponding RSpec test to make that sure you got it right.

We'll do today's work test-first. Before you start working on a step, you should run the corresponding test to see that it fails. Then you should work on it to make the test pass. Move on to the next step after the test passes.

For now, we won't do refactoring. Just go through the steps as simply as you can.

# Running The Test Suite

First clone the test suite into your `my_mongoid` project directory:

```
# In your project directory
> git clone https://github.com/hayeah/my_mongoid_spec.git
```

You need to run the spec from within the `my_mongoid/my_mongoid_spec` directory, so cd into it first.

```
> cd my_mongoid_spec
```

Run a spec to see if it's working:

```bash
> rspec version_spec.rb
MyMongoid Version:
  is a string

Finished in 0.00089 seconds
1 example, 0 failures
```

# Declare a model

In this section, we'll implement the module include hook:

```
class Foo
  include MyMongoid::Document
end
```

In this step, you'll implement the `MyMongoid::Document` mixin to turn a class into a MyMongoid model.

+ Create two empty modules: `MyMongoid::Document` and `MyMongoid::Document::ClassMethods`
  + run `rspec -e 'Document modules'`
+ A class should includes `MyMongoid::Document` to become a MyMongoid model.
+ `is_mongoid_model?` should be true for all MyMongoid models.
  + run `rspec -e 'Event is a mongoid model'`
+ `MyMongoid.models` should be a method that returns all MyMongoid models.
  + run `rspec -e 'maintains a list of models'`

Commit your work.

# Model attributes and new

In this section, we'll make it possible to initilize an object with attributes:

```
class Foo
  include MyMongoid::Document
end
foo = Foo.new({"a" => 10, "b" => 20})
```

Let's implement attributes for a model.

+ Should be able to initialize a Mongoid model with attributes, which is a Hash object.
  + `rspec -e 'can instantiate a model with attributes'`
+ Should throw `ArgumentError` if the argument for "#new" is not a Hash object.
  + `rspec -e 'throws an error if attributes it not a Hash'`
+ Should be able to get the attributes object with the `#attributes` method.
  + `rspec -e 'can read the attributes of model'`
+ Should be able to get an attribute with the method `#read_attribute(name)`
  + `rspec -e "can get an attribute with #read_attribute"`
+ Should be able to set an attribute with the method `#write_attribute(name,value)`
  + `rspec -e "can set an attribute with #write_attribute"`

When we instantiate a model, it should be a new record.

+ `#new_record?` should return true for a newly initialized record.
  + `rspec -e "is a new record initially"`

Since there is no way to `save` or `find` yet, a record would always be a new_record. Just make "#new_record?" return `true`, so there is the minimum amount of code possible to pass the test.

Commit your work.

# Declare fields

In this section, we'll implement the `field` keyword:

```
class Foo
  include MyMongoid::Document
  field :a
  field :b
end
foo = Foo.new({"a" => 10, "b" => 20})
```

+ Should be able to declare a field with the DSL "field :foo".
  + `rspec declare_field_spec.rb -e "can declare a field using the 'field' DSL"`
+ Declaring a field `foo` should generate the accessors "#foo", "#foo=" to get/set the attribute "foo".
  + `rspec declare_field_spec.rb -e "declares getter for a field"`
  + `rspec declare_field_spec.rb -e "declares setter for a field"`
+ `FooModel.fields` should return a Hash, where the keys are field names, and the values are `MyMongoid::Field` objects.
  + `rspec declare_field_spec.rb -e 'maintains a map fields objects'`
+ `MyMongoid::Field#name` should return the field name as `String`.
  + `rspec declare_field_spec.rb -e "returns a string for Field#name"`
+ Should raise `MyMongoid::DuplicateFieldError` if a field is declared twice.
  + `rspec declare_field_spec.rb -e "raises MyMongoid::DuplicateFieldError if field is declared twice"`

Run the above altogether: `rspec declare_field_spec.rb -e "Declare fields"`.

Commit your work.

Mongoid automatically declares the "_id" field. Let's do that.

+ All models should have the "_id" field automatically declared.
  + `rspec declare_field_spec.rb -e "automatically declares the '_id' field"`

Commit your work.

# Mass-Assignment & Initialize

In this section, we'll generate setters and getters for declared fields:

```
class Foo
  include MyMongoid::Document
  field :a
  field :b
end
foo = Foo.new({"a" => 10, "b" => 20})
foo.a # => 10
foo.a = 1
foo.a # => 1
```

Now that we can declare fields, let's implement mass-assignment by using field setters and getters, and then change `initialize` to use mass-assignment. (Take a look at the code for [`Mongoid::Attributes::Processing#process_attributes`](https://github.com/hayeah/mongoid/blob/5b0f031992cbec66d68c6cb288a4edb952ed5336/lib/mongoid/attributes/processing.rb#L8-L18).)

+ The `#process_attributes(attributes)` method should take a Hash object, and invoke field setter for each key & value.
  + `rspec declare_field_spec.rb -e "use field setters for mass-assignment"`
+ Should raise `MyMongoid::UnknownAttributeError` if the attributes Hash contains undeclared fields.
  + `rspec declare_field_spec.rb -e 'raise MyMongoid::UnknownAttributeError if the attributes Hash contains undeclared fields.'`
+ Make `#attributes=` an alias of `#process_attributes`
  + `rspec declare_field_spec.rb -e 'aliases #process_attributes as #attribute='`
+ Change `#initialize` to use `#process_attributes` instead of assigning to attribute directly.
  + `rspec declare_field_spec.rb -e 'uses #process_attributes for #initialize'`

`#process_attributes` is like [ActiveRecord::Base#assign_attributes](http://api.rubyonrails.org/classes/ActiveRecord/AttributeAssignment.html#method-i-assign_attributes).

Commit your work.

# Declare field aliases

In this section, we'll implement aliases for fields:

```
class Foo
  include MyMongoid::Document
  field :apple, :as => :a
end

foo = Foo.new({"apple" => 10})
foo.a # => 10
foo.a = 1
foo.apple # => 1
```

In Mongoid, if you set the "id" attribute of a model, it actually sets the "_id" attribute. This is accomplished by field aliases. By default, a Mongoid model would have "id" as an alias of "_id". (see [Mongoid::Fields](https://github.com/hayeah/mongoid/blob/5b0f031992cbec66d68c6cb288a4edb952ed5336/lib/mongoid/fields.rb#L42))

+ The `field` DSL should accept a hash as options.
  + `rspec declare_field_spec.rb -e 'accepts hash options for the field keyword'`
+ Store the field options in the Field object as `MyMongoid::Field#options`.
  + `rspec declare_field_spec.rb -e "stores the field options in Field object"`
+ Make it possible to alias a field by specifying the ":as" option. e.g. `field :_id, :as => "id"`.
  + `rspec declare_field_spec.rb -e "aliases a field with the :as option"`
+ For all MyMongoid models, make "id" the alias for "_id".

Commit your work.

# Bonus

+ Can you add default values for fields?

  `field :difficulty, :default => "normal"`

+ Can you add type declaration for fields? Raise an error if type mismatches.

  `field :time, :type => Time`

+ Can you add an initialization block?

    ```ruby
    event = Event.new(data) do |event|
      event.created_at = Time.parse(data["created_at"])
    end
    ```
