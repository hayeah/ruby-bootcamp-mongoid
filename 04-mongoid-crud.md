# Persistence And Mongoid CRUD

In this lesson you'll learn how Mongoid saves a record into MongoDB.

## Objectives

+ Learn BSON, Mongoid's serialization format.
+ Understand Moped, the Ruby MongoDB driver.

# Mongoid CRUD

Let's get familiar with Mongoid's CRUD first. Again, we'll use the following Github event object.

```ruby
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

And the following Event model,

```ruby
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

### Connecting To Mongo

Try getting the default database session:

```ruby
[1] pry(main)> Mongoid.default_session
Mongoid::Errors::NoSessionsConfig
```

You get an error, because Mongoid doesn't know how to connect to the database. Let's configure it:

```
CONFIG = {
  sessions: {
    default: {
      database: "my_mongoid",
      hosts: [ "localhost:27017" ]
    }
  }
}

Mongoid.configure do |config|
  config.load_configuration(CONFIG)
end
```

Try again:

```ruby
[5] pry(main)> Mongoid.default_session
=> <Moped::Session seeds=[<Moped::Node resolved_address="127.0.0.1:27017">] database=my_mongoid>
```

Looks ok!

### Checking DB Connection

Is our Ruby process actually connected to the Mongo database? How can we verify it?

1. Find the process id of our pry session.
2. Use the `lsof` tool to show all open files and connections for that process.

What's our Process id?

```
[25] pry(main)> Process.pid
=> 5152
```

It's 5152. We'll invoke the `lsof` command with the `-p` flag, to list only the open files of our process:

```
[24] pry(main)> system "lsof -p #{Process.pid}"
COMMAND  PID   USER   FD   TYPE             DEVICE  SIZE/OFF     NODE NAME
ruby    5152 howard  cwd    DIR                1,2       510 19879879 /Users/howard/workspace/rubybootcamp-mongoid
ruby    5152 howard  txt    REG                1,2      9448 19985444 /Users/howard/.rvm/rubies/ruby-1.9.3-p484/bin/ruby
ruby    5152 howard  txt    REG                1,2   2882936 19985446 /Users/howard/.rvm/rubies/ruby-1.9.3-p484/lib/libruby.1.9.1.dylib
ruby    5152 howard  txt    REG                1,2     13272 19985476 /Users/howard/.rvm/rubies/ruby-1.9.3-p484/lib/ruby/1.9.1/x86_64-darwin13.0.0/enc/encdb.bundle
... more
ruby    5152 howard  txt    REG                1,2 342928962 12522710 /private/var/db/dyld/dyld_shared_cache_x86_64
ruby    5152 howard    0u   CHR               16,2 0t2041644      687 /dev/ttys002
ruby    5152 howard    1u   CHR               16,2 0t2041644      687 /dev/ttys002
ruby    5152 howard    2u   CHR               16,2 0t2041644      687 /dev/ttys002
ruby    5152 howard    3   PIPE 0xe27ef47dd9d43cdb     16384          ->0xe27ef47dc6a87b2b
ruby    5152 howard    4   PIPE 0xe27ef47dc6a87b2b     16384          ->0xe27ef47dd9d43cdb
```

You can see that our process opens many files. We see the standard IO at the FD (file descriptor) 0,1,2. In my case, they read and write to the file `/dev/ttys002`.

```
ruby    5152 howard    0u   CHR               16,2 0t2041644      687 /dev/ttys002
ruby    5152 howard    1u   CHR               16,2 0t2041644      687 /dev/ttys002
ruby    5152 howard    2u   CHR               16,2 0t2041644      687 /dev/ttys002
```

However, there's no TCP connection in the lsof output. Let's try making a database query:

```ruby
[26] pry(main)> Mongoid.default_session.databases
=> {"databases"=>
  [{"name"=>"local", "sizeOnDisk"=>83886080.0, "empty"=>false},
   {"name"=>"admin", "sizeOnDisk"=>1.0, "empty"=>true}],
 "totalSize"=>1610612736.0,
 "ok"=>1.0}
```

It looks like the query succeeds. We got back a list of databases. Let's run `lsof` again (this time we use the `-i` flag to only list TCP connections):

```ruby
[27] pry(main)> system "lsof -a -p #{Process.pid} -i tcp"
COMMAND  PID   USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
ruby    5152 howard    5u  IPv4 0xe27ef47dcc8a3d33      0t0  TCP localhost:55767->localhost:27017 (ESTABLISHED)
```

Now we see a connection. Its FD is 5, its type is TCP, and it connects to port 27017. So Mongoid connects to the database only when you make a query.

Let's disconnect from Mongo:

```
[30] pry(main)> Mongoid.default_session.disconnect
=> true
```

Now, `lsof` again to verify that the connection is gone:

```ruby
[31] pry(main)> system "lsof -a -p #{Process.pid} -i tcp"
# empty output
```

`lsof` is a very useful tool to help you debug connection problems. For example:

+ Is my server up? Is it waiting for incoming connections?
+ How many clients are connected to my server?
+ Which processes are writing to a log file?
+ Which processes are listening to a port?


### CRUD

Create an Event:

```
[4] pry(main)> event = Event.create(data)
=> #<Event _id: 1978774765, type: "PushEvent", actor: {"id"=>382747, "login"=>"andrepl", "gravatar_id"=>"411d2b4791a8de51f98666e93e9f1fde", "url"=>"https://api.github.com/users/andrepl", "avatar_url"=>"https://gravatar.com/avatar/411d2b4791a8de51f98666e93e9f1fde?d=https%3A%2F%2Fa248.e.akamai.net%2Fassets.github.com%2Fimages%2Fgravatars%2Fgravatar-user-420.png&r=x"}, org: nil, payload: {"push_id"=>307955115, "size"=>1, "distinct_size"=>1, "ref"=>"refs/heads/master", "head"=>"9487ef037120e52091bd5f59d4e2fc983188035f", "before"=>"ceac7512abbc515f90797ebb5644b7ced52588be", "commits"=>[{"sha"=>"9487ef037120e52091bd5f59d4e2fc983188035f", "author"=>{"email"=>"andre.leblanc@webfilings.com", "name"=>"Andre LeBlanc"}, "message"=>"more style", "distinct"=>true, "url"=>"https://api.github.com/repos/andrepl/andrepl.github.io/commits/9487ef037120e52091bd5f59d4e2fc983188035f"}]}, public: true, repo: {"id"=>16670304, "name"=>"andrepl/andrepl.github.io", "url"=>"https://api.github.com/repos/andrepl/andrepl.github.io"}, created_at: "2014-02-13T03:20:37Z">
[5] pry(main)> Event.count
=> 1
```

Update it:

```
[12] pry(main)> event.created_at = Time.now.utc.xmlschema
=> "2014-02-18T10:25:08Z"
[13] pry(main)> event.save
=> true
```

Find it:

```
[15] pry(main)> Event.find("1978774765")
=> #<Event _id: 1978774765, type: "PushEvent", actor: {"id"=>382747, "login"=>"andrepl", "gravatar_id"=>"411d2b4791a8de51f98666e93e9f1fde", "url"=>"https://api.github.com/users/andrepl", "avatar_url"=>"https://gravatar.com/avatar/411d2b4791a8de51f98666e93e9f1fde?d=https%3A%2F%2Fa248.e.akamai.net%2Fassets.github.com%2Fimages%2Fgravatars%2Fgravatar-user-420.png&r=x"}, org: nil, payload: {"push_id"=>307955115, "size"=>1, "distinct_size"=>1, "ref"=>"refs/heads/master", "head"=>"9487ef037120e52091bd5f59d4e2fc983188035f", "before"=>"ceac7512abbc515f90797ebb5644b7ced52588be", "commits"=>[{"sha"=>"9487ef037120e52091bd5f59d4e2fc983188035f", "author"=>{"email"=>"andre.leblanc@webfilings.com", "name"=>"Andre LeBlanc"}, "message"=>"more style", "distinct"=>true, "url"=>"https://api.github.com/repos/andrepl/andrepl.github.io/commits/9487ef037120e52091bd5f59d4e2fc983188035f"}]}, public: true, repo: {"id"=>16670304, "name"=>"andrepl/andrepl.github.io", "url"=>"https://api.github.com/repos/andrepl/andrepl.github.io"}, created_at: "2014-02-18T10:25:08Z">
```

Delete it:

```
[17] pry(main)> event.delete
=> true
```

Trying to find it again, it would raise an error:

```
[15] pry(main)> Event.find("1978774765")
Mongoid::Errors::DocumentNotFound
```

# How Mongoid Implements CRUD

Mongoid uses [Moped](https://github.com/mongoid/moped) to communicate with MongoDB. Moped is responsible of

+ Serialization of Ruby objects to [BSON (MongoDB's serialization format)](http://bsonspec.org)
+ Managing connections to one or many MongoDB databases
+ Creating queries to MongoDB, using the [MongoDB wire protocol](http://docs.mongodb.org/meta-driver/latest/legacy/mongodb-wire-protocol)


## Creating A Record

Let's investigate how Mongoid creates a document。 What happens when you call `Event.create(data)`? Eventually, the `create` call goes to [Mongoid::Persistable::Creatable#insert_as_root](https://github.com/hayeah/mongoid/blob/5b0f031992cbec66d68c6cb288a4edb952ed5336/lib/mongoid/persistable/creatable.rb#L78-L80):

```ruby
# Insert the root document.
#
# @api private
#
# @example Insert the document as root.
#   document.insert_as_root
#
# @return [ Document ] The document.
#
# @since 4.0.0
def insert_as_root
  collection.insert(as_document)
end
```

**Exercise**: Can you read the Mongoid source to follow it from `create` to `insert_as_root`? Use `ack` to find where methods are.

Let's look at the `#insert_as_root` method:

What's the `#collection` method?

```ruby
[18] pry(main)> event = Event.new(data)
=> #<Event ...>
[19] pry(main)> event.collection
=> #<Moped::Collection:0x007f98235000b0
 @database=
  #<Moped::Database:0x007f98235001f0
   @name="my_mongoid",
   @session=
    <Moped::Session seeds=[<Moped::Node resolved_address="127.0.0.1:27017">] database=my_mongoid>>,
 @name="events">
```

So `collection` is a `Moped::Collection`, connecting to the database "my_mongoid".

What does the `#as_document` method do?

```ruby
[27] pry(main)> doc = event.as_document
=> {"_id"=>"1978774765",
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
 "created_at"=>"2014-02-13T03:20:37Z"}
[28] pry(main)> doc.class
=> Hash
```

`#as_document` turns the `event` record into an ordinary Ruby hash. The only
`#thing special about this hash object is that it is "Mongo Friendly", which
`#means that it can be converted to BSON (the MongoDB serialization format).

You can try to convert a few objects to BSON:

```ruby
[2] pry(main)> 10.to_bson
=> "\n\x00\x00\x00"
[3] pry(main)> 20.to_bson
=> "\x14\x00\x00\x00"
[4] pry(main)> 1.to_bson
=> "\x01\x00\x00\x00"
[5] pry(main)> 2.to_bson
=> "\x02\x00\x00\x00"
[6] pry(main)> "Hello Mongo".to_bson
=> "\f\x00\x00\x00Hello Mongo\x00"
[8] pry(main)> ["Hello Mongo",1].to_bson
=> "\x1F\x00\x00\x00\x020\x00\f\x00\x00\x00Hello Mongo\x00\x101\x00\x01\x00\x00\x00\x00"
[10] pry(main)> {"Hello" => "Mongo"}.to_bson
=> "\x16\x00\x00\x00\x02Hello\x00\x06\x00\x00\x00Mongo\x00\x00"
```

When you call `create`, Mongoid passes to Moped a Ruby hash that can be
seralized into BSON. Then Moped communicates with MongoDB to insert the
data as a BSON document.

# Using Moped Directly

Using Mongoid is like using ActiveRecord, and using Moped is like using SQL.

Mongoid's [Documentation for Persistence](http://mongoid.org/en/mongoid/docs/persistence.html) has a table that corresponds Mongoid CRUD to Moped queries. For example:

`Model.create`

```ruby
# Mongoid
Person.create(
  first_name: "Heinrich",
  last_name: "Heine"
)

# Moped
collections[:people].insert({
  first_name: "Heinrich",
  last_name: "Heine"
})
```

`Model#update_attributes`

```ruby
# Mongoid
person.update_attributes(
  first_name: "Jean",
  last_name: "Zorg"
)

# Moped
person.update_attributes(
  first_name: "Jean",
  last_name: "Zorg"
)
```

`Model#delete`

```ruby
# Mongoid
person.delete

#  Moped
collections[:people].find(...).remove
```

# Playing With Moped

Let's try to interact with MongoDB using Moped.

Getting the events collection：

```
[25] pry(main)> events = Mongoid.default_session[:events]
=> #<Moped::Collection:0x007faa448eb7a0
 @database=
  #<Moped::Database:0x007faa44936598
   @name="my_mongoid",
   @session=
    <Moped::Session seeds=[<Moped::Node resolved_address="127.0.0.1:27017">] database=my_mongoid>>,
 @name="events">
```

Insert a record into Mongo:

```ruby
[54] pry(main)> data = {"_id" => "1", "payload" => "aaaa"}
[55] pry(main)> events.insert(data)
=> {"n"=>0, "connectionId"=>133, "err"=>nil, "ok"=>1.0}
```

To get the record from MongoDB:

```ruby
[57] pry(main)> events.find({"_id" => "1"}).first
=> {"_id"=>"1", "payload"=>"aaaa"}
```

Remember that `id` is a Mongoid alias for `_id`. It wouldn't work if you try to find a record with `id`:

```ruby
[70] pry(main)> events.find({"id" => "1"}).first
=> nil
```

To update the record we've created:

```ruby
[65] pry(main)> c.update("$set" => {"payload" => "bbbb"})
=> {"updatedExisting"=>true, "n"=>1, "connectionId"=>133, "err"=>nil, "ok"=>1.0}
[66] pry(main)> events.find({"_id" => "1"}).first
=> {"_id"=>"1", "payload"=>"bbbb"}
```

It should raise an error if you try to insert another record with the same `_id`:

```ruby
[61] pry(main)> data = {"_id" => "1", "payload" => "cccc"}
=> {"_id"=>"1", "payload"=>"cccc"}
[62] pry(main)> events.insert(data)
Moped::Errors::OperationFailure: The operation: #<Moped::Protocol::Command
  @length=74
  @request_id=40
  @response_to=0
  @op_code=2004
  @flags=[]
  @full_collection_name="my_mongoid.$cmd"
  @skip=0
  @limit=-1
  @selector={:getlasterror=>1, :w=>1}
  @fields=nil>
failed with error 11000: "E11000 duplicate key error index: my_mongoid.events.$_id_  dup key: { : \"1\" }"
```

To delete the record:

```ruby
[68] pry(main)> events.find({"_id" => "1"}).remove
=> {"n"=>1, "connectionId"=>133, "err"=>nil, "ok"=>1.0}
[69] pry(main)> events.find({"_id" => "1"}).count
=> 0
```
