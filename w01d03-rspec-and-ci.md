# Writing RSpec

RSpec is the test framework that I've seen most often in different projects.
It's harder to learn than others test frameworks. But since many projects use
it, you'd have to learn to read and write RSpec whether you like it or not.

[@dhh doesn't like RSpec](https://twitter.com/dhh/status/52807321499340800)

If you want something simpler, you might try

[Test::Unit](http://www.ruby-doc.org/stdlib-2.1.0/libdoc/test/unit/rdoc/Test/Unit.html)

Which is a Ruby standard library, and used by the Rails project.

## Objectives

You'll learn:

+ How RSpec files in the `spec` directory is organized.
+ How an RSpec file is structured.
+ Write your own RSpec.
+ Setup TravisCI for your project.
+ Setup Coverall for your project.

Read [An Introduction To RSpec](http://blog.teamtreehouse.com/an-introduction-to-rspec) before you continue.

Note that the `should` syntax is deprecated in RSpec 3. Use expectation syntax: [rspec-expectation](https://github.com/rspec/rspec-expectations).

If you are familiar with RSpec 2, you might want to read about how how RSpec 3 is different: [The Plan for RSpec 3](http://myronmars.to/n/dev-blog/2013/07/the-plan-for-rspec-3).

# How Spec Files Are Organized

When you run the `rspec` command, it looks in the `spec` directory for all files that matches the pattern `*_spec.rb`.

What spec files are there?

```sh
> find spec -name "*_spec.rb"
spec/mongoid/atomic/modifiers_spec.rb
spec/mongoid/atomic/paths/embedded/many_spec.rb
spec/mongoid/atomic/paths/embedded/one_spec.rb
spec/mongoid/atomic/paths/root_spec.rb
spec/mongoid/atomic/paths_spec.rb
spec/mongoid/atomic_spec.rb
spec/mongoid/attributes/nested_spec.rb
spec/mongoid/attributes/readonly_spec.rb
spec/mongoid/attributes_spec.rb
spec/mongoid/changeable_spec.rb
spec/mongoid/composable_spec.rb
...
```

The `rspec` command loads these files automatically.

What non-spec files are there?

```sh
> find spec -not -name "*_spec.rb" -not -type d
spec/app/models/account.rb
spec/app/models/acolyte.rb
spec/app/models/actor.rb

... many other files

spec/config/mongoid.yml
spec/helpers.rb
spec/spec_helper.rb
```

These files aren't automatically loaded by RSpec. The file `spec_helper.rb` is
responsible to load all these extra stuff, and do any other test
configuration. Then each spec file just `require "spec_helper"`. Like this:

```ruby
require "spec_helper"

describe Mongoid::Document do
end
```

The files in `spec/app/models` are models used in tests. For example, `spec/app/models/account.rb` defines the `Account` model:

```ruby
class Account
  include Mongoid::Document

  field :_id, type: String, overwrite: true, default: ->{ name.try(:parameterize) }

  field :number, type: String
  field :balance, type: String
  field :nickname, type: String
  field :name, type: String
  field :balanced, type: Mongoid::Boolean, default: ->{ balance? ? true : false }

  field :overridden, type: String

  embeds_many :memberships
  belongs_to :creator, class_name: "User", foreign_key: :creator_id
  belongs_to :person
  has_many :alerts, autosave: false
  has_and_belongs_to_many :agents
  has_one :comment, validate: false

  validates_presence_of :name
  validates_presence_of :nickname, on: :upsert
  validates_length_of :name, maximum: 10, on: :create

  def overridden
    self[:overridden] = "not recommended"
  end
end
```

`Account` is used in `spec/mongoid/reloadable_spec.rb`:

```
spec/mongoid/reloadable_spec.rb
33:        Account.create(name: "bank", number: "1000")
37:        Account.find(account.id).tap do |acc|
```

Mongoid does a very good job at breaking implementation into modules. The organization of spec files mirrors the source files by a naming convention. The spec file for "foo.rb" should correspond to "foo_spec.rb". For examples:

+ `lib/mongoid/document.rb` and `spec/mongoid/document_spec.rb`
+ `lib/mongoid/attributes/readonly.rb` and `spec/mongoid/attributes/readonly_spec.rb`


# Spec Helper

Let's look at what spec_helper does.

First, it sets up the load path so it can `require "mongoid"` and load test models in `spec/app/models`.

```
$LOAD_PATH.unshift(File.dirname(__FILE__))
$LOAD_PATH.unshift(File.join(File.dirname(__FILE__), "..", "lib"))

MODELS = File.join(File.dirname(__FILE__), "app/models")
$LOAD_PATH.unshift(MODELS)
```

Later, sets up autoload for all test models:

```ruby
# Autoload every model for the test suite that sits in spec/app/models.
Dir[ File.join(MODELS, "*.rb") ].sort.each do |file|
  name = File.basename(file, ".rb")
  autoload name.camelize.to_sym, name
end
```

Autoloading loads `account.rb` the first time Account is used. This avoids loading all the test models if you only want to run one test.

Before each running each test, delete everything in Mongo:

```ruby
RSpec.configure do |config|
  # Drop all collections and clear the identity map before each spec.
  config.before(:each) do
    Mongoid.purge!
  end
end
```

The `spec_helper` also sets up coverage report when it's running on TravisCI:

```
if ENV["CI"]
  require "coveralls"
  Coveralls.wear! do
    add_filter "spec"
  end
end
```

When build succeeds, the report is sent to [Coveralls](https://coveralls.io). For build [#4019.1](https://travis-ci.org/mongoid/mongoid/jobs/18037371#L90-L94), we can see this happening in the log:

```
[Coveralls] Submitting to https://coveralls.io/api/v1
[Coveralls] Job #4019.1
[Coveralls] https://coveralls.io/jobs/1113261
Coverage is at 98.69%.
Coverage report sent to Coveralls.
```

View the [coverage report](https://coveralls.io/jobs/1113261).

# document_spec.rb

Let's take a closer look at a spec file for `Mongoid::Document`.

We can display the structure of the spec in a human-friendly format:

```
> rspec spec/mongoid/document_spec.rb --dry-run
Run options: exclude {:config=>#<Proc:./spec/spec_helper.rb:101>}

Mongoid::Document
  does not respond to _destroy
  .included
    when Document has been included in a model
      .models should include that model
    before Document has been included
      .models should *not* include that model
    after Document has been included
      .models should include that model
    after Document has been included multiple times
      .models should include that model just once
  ._types
    when the document is subclassed
      includes the root
      includes the subclasses
    when the document is not subclassed
      returns the document
    when ._types had been called before class declaration
      should clear descendants' cache
  #attributes
    returns the attributes with indifferent access
... more
```

Notice the tree structure of the spec. All the tests for ".included" are grouped under that branch, and the tests for "._types" under its own branch. What this structure looks like in code:

```ruby
describe Mongoid::Document do
  describe ".included" do
    context "when Document has been included in a model" do
      it ".models should include that model" do
        expect(models).to include(klass)
      end
    end
  end
end
```

It is good practice to organize the structure of a spec to mirror the structure of your module. You can see that there are groups of tests for every method:

```
> grep describe spec/mongoid/document_spec.rb
describe Mongoid::Document do
  describe ".included" do
  describe "._types" do
  describe "#attributes" do
  describe "#cache_key" do
  describe "#identity" do
  describe "#hash" do
  describe "#initialize" do
  describe ".instantiate" do
  describe "#model_name" do
  describe "#raw_attributes" do
  describe "#to_a" do
  describe "#as_document" do
  describe "#to_key" do
  describe "#to_param" do
  describe "#frozen?" do
  describe "#freeze" do
  describe ".logger" do
  describe "#logger" do
  describe "#initialize" do
  describe "#becomes" do
    describe Marshal, ".dump" do
    describe Marshal, ".load" do
    describe ActiveSupport::Cache do
      describe "#fetch" do
```

This structure makes code sharing very easy. The tests for a method often need the same data. For the `.included` group you see a few `let` declarations:

```ruby
describe ".included" do
  let(:models) do
    Mongoid.models
  end

  let(:new_klass_name) do
    'NewKlassName'
  end

  # ... more
end
```

The values declared by these `let` expressions can be used by all the tests within the `.included` block.

Let's look at the `#cache_key` group. It is a little more complicated:

```ruby
describe "#cache_key" do
  let(:document) do
    Dokument.new
  end

  context "when the document is new" do
  end

  context "when persisted" do
    before do
      document.save
    end
  end
end
```

In this case, `document` is available over all the context blocks, but `document.save` only runs for tests within the "when persisted" block.

RSpec's nesting contexts gives you a lot of control in managing scopes. As a result, rspec tests themselves often are very short. Notice how most of the specs are just one line long:

```
it "returns the id as a string" do
  expect(person.to_param).to eq(person.id.to_s)
end

it "copies attributes" do
  expect(person.title).to eq('Sir')
end

it "keeps the same object id" do
  expect(person.id).to eq(manager.id)
end

it "copies the embedded documents" do
  expect(person.addresses.first).to eq(address)
end

it "copies the embedded documents only once" do
  expect(person.reload.addresses.length).to eq(1)
end
```

# RSpec Summary

+ The organization of spec files in `spec` mirrors the organization of source files in the `lib` folder.
+ Within a spec file, the test groups mirror the structure of the module.
+ By using nesting context to manage scope, your spec can be very [DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself).

# Writing Spec for MyMongoid

## Init RSpec

Add rspec to your `Gemfile`:

```
# Add in Gemfile
group :test do
  gem "rspec", "~> 3.0.0.beta1"
end
```

Run `bundle install`

Init rspec for your project:

```
> rspec --init
  create   spec/spec_helper.rb
  create   .rspec
```

Run the empty rspec suite:

```
> rspec
No examples found.


Finished in 0.00006 seconds
0 examples, 0 failures
```

Commit your work.

## Creating RSpec Files (DIY)

We'll organize our rspec files the same way as Mongoid. Even though all our code is in `lib/my_mongoid.rb`, we'll break down our spec files to match independent modules.

First, create two files.

```ruby
# spec/my_mongoid/document_spec.rb
describe MyMongoid::Document do
  it "is a module" do
    expect(MyMongoid::Document).to be_a(Module)
  end
end
```

and

```ruby
# spec/my_mongoid/field_spec.rb
describe MyMongoid::Field do
  it "is a module" do
    expect(MyMongoid::Field).to be_a(Module)
  end
end
```

Now run rspec:

```sh
> rspec
/Users/howard/workspace/rubybootcamp-mongoid/my_mongoid/spec/my_mongoid/document_spec.rb:1:in `<top (required)>': uninitialized constant MyMongoid::Document (NameError)
        from /Users/howard/.rvm/gems/ruby-1.9.3-p484/gems/rspec-core-3.0.0.beta1/lib/rspec/core/configuration.rb:886:in `load'
        from /Users/howard/.rvm/gems/ruby-1.9.3-p484/gems/rspec-core-3.0.0.beta1/lib/rspec/core/configuration.rb:886:in `block in load_spec_files'
        from /Users/howard/.rvm/gems/ruby-1.9.3-p484/gems/rspec-core-3.0.0.beta1/lib/rspec/core/configuration.rb:886:in `each'
        from /Users/howard/.rvm/gems/ruby-1.9.3-p484/gems/rspec-core-3.0.0.beta1/lib/rspec/core/configuration.rb:886:in `load_spec_files'
        from /Users/howard/.rvm/gems/ruby-1.9.3-p484/gems/rspec-core-3.0.0.beta1/lib/rspec/core/command_line.rb:22:in `run'
        from /Users/howard/.rvm/gems/ruby-1.9.3-p484/gems/rspec-core-3.0.0.beta1/lib/rspec/core/runner.rb:90:in `run'
        from /Users/howard/.rvm/gems/ruby-1.9.3-p484/gems/rspec-core-3.0.0.beta1/lib/rspec/core/runner.rb:17:in `block in autorun'
```

+ Fix the error by loading "my_mongoid" in spec_helper.rb

It should now work:

```
> sh
rspec --format doc
MyMongoid::Document
  is a module

MyMongoid::Field
  is a module

Finished in 0.00102 seconds
2 examples, 0 failures
```

## Complete the Spec

Now you should finish writing the spec on your own. Remember, your spec should follow the structure of your module. So `document_spec.rb` might look like:

```ruby
describe MyMongoid::Document do
  describe ".new"
  describe "#read_attribute"
  describe "#write_attribute"
  describe "#process_attributes"
  describe "#new_record?"
end
```

You could start from scratch, or adapt [my_mongoid_spec](https://github.com/hayeah/my_mongoid_spec).

# Setup TravisCI (DIY)

Look at [Getting Started](http://docs.travis-ci.com/user/getting-started/) to learn how to setup TravisCI for your project.

Create `.travis.yml` at the root of your project. We'll use the following configuration:

```
# .travis.yml
language: ruby
bundler_args: --without development
rvm:
  - 1.9.3
  - 2.0.0
  - 2.1.0
script: "rspec"
```

Commit and push to Github.

Go to your [profile page](https://travis-ci.org/profile) and toggle the `my_mongoid` project to "On".

Add the build status image to your `README.md`:

```
[![Build Status]](https://travis-ci.org/{your_github_account}/my_mongoid)
```

Commit and push. TravisCI should start building.

# Bonus

+ Setup Coverall for your project.
+ Go for 100% coverage.
