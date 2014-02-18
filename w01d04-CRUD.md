# Persistence And CRUD

We've created a simple document model in Ruby. In this lesson, you'll learn how to save that object into MongoDB.

You'll implement Create, Read, Update, Delete for MyMongoid.

## Objectives

+ Learn BSON, Mongoid's serialization format.
+ Understand Moped, the Ruby MongoDB driver.
+ Create a singleton connection object to handle DB queries.
+ Implement CRUD for MyMongoid

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

# BSON



#

{just focus on find, create, update_attribute, delete. ignore upsert.}

+ create MyMongoid.setup(options)
+ configure spec_helper
+ use lsof to find port of mongo, and use tcpdump to listen to it.
+ to_bson
+ add moped as dependency to Gemfile
+ configuring the db connection
+ create connection to mongo. use it directly for experiments.

+ show source code from Moped, detailing the wire protocol.
++ good to illustrate the use of metaprogramming to define DSL

+ show binary data of to_bson, then show it on the wire

+ compare & contrast with redis' plain text protocol
++ cite http as an example of plain text protocol. why plain text is awesome.
++ cite joe armstrong's binary format, why it's awesome.


+ dissect bson format. compare to other serialization format.
+ eavedrop on mongo wire protocol, tcpdump


+ dirty tracking (don't think this is super interesting, maybe as an optional exercise)
+ I think I'd like to keep this

{implementation wise, this is actually pretty easy. just use moped}

+ what if a record is a new_record, and we call update_attributes?
+ what's the difference between upsert and create?

