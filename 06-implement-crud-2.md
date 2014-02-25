# Implement MyMongoid CRUD - Part 2

We'll complete CRUD by implementing record update and deletion.

# Objectives

+ Fields dirty tracking. Which fields had been written to?
+ '#save' for saving field changes to a record.
+ '#update_attributes' for updating fields directly.
+ '#delete' for removing a record.

# How Mongoid Updates A Record

When a record is new, `#save` should insert a new record into the database. Thus,

```ruby
person = Person.new(
  first_name: "Heinrich",
  last_name: "Heine"
)
person.save
```

should cause Moped to insert a record:

```ruby
collections[:people].insert({
  first_name: "Heinrich",
  last_name: "Heine"
})
```

If we make changes to the record, and save it for a second time,

```ruby
person.first_name = "Christian Johan"
person.save
```

then `Moped` should update the record in the database:

```ruby
collections[:people].find(...).
  update("$set" => { first_name: "Christian Johan" })
```

Only fields that changed are included in the `$set` query, which is sent to the database.

So to implement record udpate, we need to

1. Keep track of which fields had changed.
2. When `#save` is invoked, issue a Moped update query to the database.

We need to extend the `#save` method that we implemented earlier for `#create` to support record updating.

# How Mongoid Implements Save

We can find `#save` in `Mongoid::Persistable::Savable`:


```ruby
def save(options = {})
  if new_record?
    !insert(options).new_record?
  else
    update_document(options)
  end
end
```

Pretty straightforward. For a new record, it inserts a new record. For a persisted record, it calls `#update_document` to update the document in the database. Let's look at `#update_document`:

```ruby
def update_document(options = {})
  raise Errors::ReadonlyDocument.new(self.class) if readonly?
  prepare_update(options) do
    updates, conflicts = init_atomic_updates
    unless updates.empty?
      coll = _root.collection
      selector = atomic_selector
      coll.find(selector).update(positionally(selector, updates))
      conflicts.each_pair do |key, value|
        coll.find(selector).update(positionally(selector, { key => value }))
      end
    end
  end
end
```

This looks scary. A lot of the complexity comes from [document embedding](http://mongoid.org/en/mongoid/docs/relations.html#embeds_one). Since we are not doing embedding yet, we could be simplify `#update_document` quite a bit:

```ruby
def update_document
  # get the field changes
  updates = atomic_updates

  # make the update query
  unless updates.empty?
    selector = { "_id" => self.id }
    collection.find(selector).update(updates)
  end
end
```

The simplified version:

+ Removes the `readonly` check
+ Assumes that we won't have conflicting atomic updates (we'll make sure MyMongoid's design avoids conflicting updates)
+ Assumes that the selector is always for the document itself (this is ok, until we add embedded document)
+ Assumes that the collection we query with is the model (this is ok, until we add embedded document)

Now we just need to figure ot what `atomic_updates` does.

```
[28] pry(main)> event = Event.create(DATA)
=> #<Event _id: 1978774765, type: "PushEvent" ...>
[31] pry(main)> event.public = false
=> false
[32] pry(main)> event.type = "AnotherType"
=> "AnotherType"
[33] pry(main)> event.changes
=> {"public"=>[true, false], "type"=>["PushEvent", "AnotherType"]}
[34] pry(main)> event.atomic_updates
=> {"$set"=>{"public"=>false, "type"=>"AnotherType"}}
```

As you can see `#atomic_updates` generates the Mongo [`$set` update operation](http://docs.mongodb.org/manual/reference/operator/update/set/).

**Exercise** Think about how you might implement document embedding.

# Implement Document Update

We'll break this feature into two parts:

1. Keep track of which fields had changed.
2. When `#save` is invoked, issue a Moped update query.

## Implement Dirty Tracking

Dirty tracking keeps a record of what attributes had changed by keeping the original unchanged values in a hash. In the next part, we'll use this to generate the `$set` update operation.

**Implement** `#changed_attributes` returns a hash of the original values of fields that had changed.

Pass: rspec crud_spec.rb -e '#changed_attributes'

```
Should be able to update a record
  #changed_attributes
    should be an empty hash initially
    should track writes to attributes
    should keep the original attribute values
    should not make a field dirty if the assigned value is equaled to the old value
```

**Implement** `#changed?` to return true if a field changed.

Pass: rspec crud_spec.rb -e '#changed?'

```
Should track changes made to a record
  #changed?
    should be false for a newly instantiated record
    should be true if a field changed
```

**Note**: An odd behaviour of Mongoid's `#changed_attributes` is that it considers attributes passed to '#initialize' as changes:

```ruby
[23] pry(main)> Event.new({"public" => true, "id" => "1"})
=> #<Event _id: 1, type: nil, actor: nil, org: nil, payload: nil, public: true, repo: nil, created_at: nil>
[24] pry(main)> e = Event.new({"public" => true, "id" => "1"})
=> #<Event _id: 1, type: nil, actor: nil, org: nil, payload: nil, public: true, repo: nil, created_at: nil>
[25] pry(main)> e.changed_attributes
=> {"_id"=>nil, "public"=>nil}
[26] pry(main)> e.changes
=> {"_id"=>[nil, "1"], "public"=>[nil, true]}
```

Because Mongoid generates an id for a new record, `changed?` is true for a new record:

```ruby
[30] pry(main)> e = Event.new
=> #<Event _id: 530b537a68792d4cb8010000, type: nil, actor: nil, org: nil, payload: nil, public: nil, repo: nil, created_at: nil>
[31] pry(main)> e.changed?
=> true
```

But because it is a `new_record`, `atomic_updates` returns an empty hash:

```ruby
[27] pry(main)> e.atomic_updates
=> {}
```

## Reset Dirty Fields

Immediately after saving a record, we should reset `#changed_attributes` because the changes had already been made. Implementing this ensures that `Model.create(data)` returns a record that returns false for '#changed?'.

**Modify** `#save` to reset changed attributes to empty hash after saving.

Pass: `rspec crud_spec.rb -e "updating database: #save should have no changes"`

```
Should be able to update a record:
  updating database:
    #save
      should have no changes right after persisting
```

## Implement Record Update

We'll implement record updating by completing the simplified `update_document`:

```ruby
def update_document
  # get the field changes
  updates = atomic_updates

  # make the update query
  unless updates.empty?
    selector = { "_id" => self.id }
    collection.find(selector).update(updates)
  end
end
```

When an update occurs, the resulting query is something like:

```ruby
field_changes = {"field1" => 1, "field2" => 2}
collection.find({"id" => id}).update({"$set" => field_changes})
```

**Implement** `#atomic_updates` to generate $set update operation.

Hint: `#atomic_updates` should ignore changes to `_id`. You can't change the id of a Mongoid record.

```
[47] pry(main)> e = Event.find("1978774765")
=> #<Event _id: 1978774765, type: "PushEvent" ...>
[48] pry(main)> e.id = "23"
=> "23"
[49] pry(main)> e.changes
=> {"_id"=>["1978774765", "23"]}
[50] pry(main)> e.atomic_updates
=> {}
[51] pry(main)> e.save
=> true
[52] pry(main)> e = Event.find("23")
Mongoid::Errors::DocumentNotFound:
```

Pass: `rspec crud_spec.rb -e "#atomic_updates"`

```
Should be able to update a record
  #atomic_updates
    should return {} if nothing changed
    should return {} if record is not a persisted document
    should generate the $set update operation to update a persisted document
```

**Implement** `#update_document` to update the document in database

Pass: `rspec crud_spec.rb -e "#update_document"`

```
Should be able to update a record:
  updating database:
    #update_document
      should not issue query if nothing changed
      should update the document in database if there are changes
```

**Modify** `#save` to save the changes of a persisted document (before it's only used for `#create`).

Pass: `rspec crud_spec.rb -e "updating database: #save"`

```
Should be able to update a record:
  updating database:
    #save
      should save the changes if a document is already persisted
```

**Implement** `#update_attributes` (see [Persistence](http://mongoid.org/en/mongoid/docs/persistence.html))

```
Should be able to update a record:
  updating database:
    #update_attributes
      should change and persiste attributes of a record
```

# Delete

**Implement** `#delete`

Pass: `rspec crud_spec.rb -e "#delete"`

```
Should be able to delete a record:
  #delete
    should delete a record from db
    should return true for deleted?
```

# Bonus

+ Mongoid's `add_field` class method invokes `create_dirty_methods` to create methods related to dirty tracking. Read Mongoid's source code to read what those methods are, and implement them.
+ Implement `#reload` to fetch attributes from database and reload the object.