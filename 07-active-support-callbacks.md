# ActiveSupport::Callbacks And Cross-Cutting Concerns

In this lesson we'll learn about how Mongoid uses `ActiveSupport::Callbacks` to implement [lifecycle callbacks](http://mongoid.org/en/mongoid/docs/callbacks.html#callbacks).

## Objectives

+ Look at how `Mongoid::Interceptable` uses ActiveSupport to implement callbacks.
+ Understand `ActiveSupport::Callbacks`
+ Understand how ActiveRecord uses `ActiveModel::Callbacks` to implement callbacks.
+ Practice using `ActiveSupport::Callbacks`

# Mongoid::Interceptable

`Mongoid::Interceptable` declares all the lifecycle callback hooks for Mongoid.

```ruby
included do
  extend ActiveModel::Callbacks
  include ActiveModel::Validations::Callbacks

  define_model_callbacks :build, :find, :initialize, :touch, only: :after
  define_model_callbacks :create, :destroy, :save, :update, :upsert

  attr_accessor :before_callback_halted
end
```

`ActiveModel::Callbacks` provides the [`define_model_callbacks`](http://api.rubyonrails.org/classes/ActiveModel/Callbacks.html#method-i-define_model_callbacks) method, which is then used to declare callback hooks for `build`, `find`, `create`, `destroy` etc.

Notice that for the first `define_model_callbacks` (`build`,`find`,`initialize`, `touch`), the `:only => :after` option is specified, so only the after callbacks are declared.

If you look at the list of callbacks in the Mongoid doc:

```
after_initialize
after_build
before_validation
after_validation
before_create
around_create
after_create
after_find Since 3.1.0
before_update
around_update
after_update
before_upsert
around_upsert
after_upsert
before_save
around_save
after_save
before_destroy
around_destroy
after_destroy
```

There's `after_initialize`, but no `before_initialize` or `around_initialize`. The `create` callback hook declared by the second `define_model_callbacks`, in contrast, created `before_create`, `around_create`, and `after_create`.

The [`run_callbacks`](http://api.rubyonrails.org/classes/ActiveSupport/Callbacks.html#method-i-run_callbacks) method is used to run callbacks added to a hook. We can see how `run_callbacks` is used in Mongoid:

```
# With the -C flag, ack shows 2 lines above and below the lines that match a patern.
> ack -C 2 run_callbacks lib
lib/mongoid/persistable/upsertable.rb
44-      def prepare_upsert(options = {})
45-        return false if performing_validations?(options) && invalid?(:upsert)
46:        result = run_callbacks(:upsert) do
47-          yield(self)
48-          true

lib/mongoid/relations/builders.rb
69-            _building do
70-              child = send("#{name}=", document)
71:              child.run_callbacks(:build)
72-              child
73-            end

lib/mongoid/relations/embedded/many.rb
84-          doc.apply_post_processed_defaults
85-          yield(doc) if block_given?
86:          doc.run_callbacks(:build) { doc }
87-          doc
88-        end

lib/mongoid/relations/referenced/many.rb
84-          doc.apply_post_processed_defaults
85-          yield(doc) if block_given?
86:          doc.run_callbacks(:build) { doc }
87-          doc
88-        end

lib/mongoid/relations/touchable.rb
34-          _root.collection.find(selector).update(positionally(selector, touches))
35-        end
36:        run_callbacks(:touch)
37-        true
38-      end

lib/mongoid/reloadable.rb
29-      apply_defaults
30-      reload_relations
31:      run_callbacks(:find) unless _find_callbacks.empty?
32:      run_callbacks(:initialize) unless _initialize_callbacks.empty?
33-      self
34-    end
```

**Exercise** Read the following docs:

+ [ActiveSupport::Callbacks](http://api.rubyonrails.org/classes/ActiveSupport/Callbacks.html)
+ [ActiveSupport::Callbacks::ClassMethods](http://api.rubyonrails.org/classes/ActiveSupport/Callbacks/ClassMethods.html)
+ [ActiveModel::Callbacks](http://api.rubyonrails.org/classes/ActiveModel/Callbacks.html)

# Understanding ActiveModel::Callback

`ActiveModel::Callback` is more convenient way to define callbacks. It's an abstraction built on top of `ActiveSupport::Callback`.

**Exercise** Read [Getting to Know ActiveSupport::Callbacks](http://thomasmango.com/2011/09/02/getting-to-know-active-support-callbacks/). Look at the section "Life Cycle Callbacks in ActiveRecord".

For the rest of this lesson, we'll focus on `ActiveSupport::Callbacks` only.

# Cross-Cutting Concerns

Callbacks are good for doing things that are unrelated to the main business logic of a class. To say that in a more technical way, `ActiveSupport::Callbacks` is a way to handle [cross-cutting concerns](http://en.wikipedia.org/wiki/Cross-cutting_concern).

Let's illusrate the idea with a hypothetical banking application.

The banking app has the `Account` class that controls the money that a client has,

```ruby
class Account
  attr_reader :balance
  def initialize(balance)
    @balance = balance
  end

  def deposit(amount)
    @balance += amount
  end

  def withdraw(amount)
    @balance -= amount
  end

  def valid_access?
    true
  end
end
```

Furthermore, there's the `ATM` class that allows a client (let's keep it very simple for now):

```ruby
class ATM
  def initialize(account)
    @account = account
    @password = password
  end

  def withdraw(amount)
    account.withdraw(amount)
    -amount
  end

  def deposit(amount)
    account.deposit(amount)
    amount
  end
end
```

The `ATM` class simply takes money or put money into a specified account. Suppose we want to add logging to `ATM` so we have a record of everything it does to an account, then we might change the code this way:

```ruby
class ATM
  def initialize(account)
    @account = account
  end

  def withdraw(amount)
    log "before: #{account.balance}"
    account.withdraw(amount)
    log "after: #{account.balance}"
    -amount
  end

  def deposit(amount)
    log "before: #{account.balance}"
    account.deposit(amount)
    log "after: #{account.balance}"
    amount
  end
end
```

If we want to authenticate the account before we allow the ATM to do anything, we might change it this way:

```ruby
class ATM
  def initialize(account)
    @account = account
  end

  def withdraw(amount)
    return unless account.valid_access?
    log "before: #{account.amount}"
    account.withdraw(amount)
    log "after: #{account.amount}"
    -amount
  end

  def deposit(amount)
    return unless account.valid_access?
    log "before: #{account.amount}"
    account.deposit(amount)
    log "after: #{account.amount}"
    amount
  end
end
```

We might want to send a text message to the account owner if there is a transaction made to his account:

```ruby
class ATM
  def initialize(account)
    @account = account
  end

  def withdraw(amount)
    return unless account.valid_access?
    log "before: #{account.balance}"
    account.withdraw(amount)
    log "after: #{account.balance}"
    send_sms "your new balance is #{account.balance}"
    -amount
  end

  def deposit(amount)
    return unless account.valid_access?
    log "before: #{account.balance}"
    account.deposit(amount)
    log "after: #{account.balance}"
    send_sms "your new balance is #{account.balance}"
    amount
  end
end
```

You see that logging, authentication, and notification by sms are repeated in both `ATM#withdraw` and `ATM#deposit`. All three of these features don't have anything to do with the business logic (e.g. logging is unrelated to taking money from an account), and they don't have anything to do with each other (e.g. logging is unrelated to authentication).

> [Cross-cutting concerns](http://en.wikipedia.org/wiki/Cross-cutting_concern) are parts of a program that rely on or must affect many other parts of the system. They form the basis for the development of aspects."

In the `ATM`, logging, authentication, and sms notification are cross-cutting concerns. In the next section, we'll use `ActiveSupport::Callbacks` to separate these concerns into separate modules.

# Separating ATM Into Concerns

We'll use `ActiveSupport::Callbacks` to separate concerns into modules. First, we have the `ATM` class:

```ruby
class ATM
  attr_reader :account
  def initialize(account)
    @account = account
  end
end
```

Then we have the main business logic for withdrawl and deposit (this is called the "core-concern"):

```ruby
module ATM::Commands
  def withdraw(amount)
    account.withdraw(amount)
    -amount
  end

  def deposit(amount)
    account.deposit(amount)
    amount
  end
end
```

Each concern should have its own module:

```
module ATM::Authentication
  extend ActiveSupport::Concern

  def valid_access?
    @account.valid_access?
  end
end

module ATM::Logging
  extend ActiveSupport::Concern

  def log(msg)
    puts msg
  end
end

module ATM::SMSNotification
  extend ActiveSupport::Concern

  def send_sms(msg)
    # fake send
  end
end
```

Finally, we create the `ATM::Concerns` module to tie it all together:

```
module ATM::Concerns
  extend ActiveSupport::Concern

  included do
    include ActiveSupport::Callbacks
    include Commands
    include Authentication
    include Logging
    include SMSNotification
  end
end
ATM.include(ATM::Concerns)
```

## Implement ATM Concerns (DIY)

We'll practice using `ActiveSupport::Callbacks`.

**Exercise** Use `define_callbacks`, `set_callback`, `run_callbacks` to implement the full `ATM` functionality as separate concerns as outlined in the previous section. Define the `:command` hook that runs callbacks when `ATM#withdraw` or `ATM#deposit` is invoked.

Download the gist [atm.rb](https://gist.github.com/hayeah/709ffc88135dc9c2fd6b) and put it in the `my_mongoid_spec` directory.

Refer to:

+ [ActiveSupport::Callbacks](http://api.rubyonrails.org/classes/ActiveSupport/Callbacks.html)
+ [ActiveSupport::Callbacks::ClassMethods](http://api.rubyonrails.org/classes/ActiveSupport/Callbacks/ClassMethods.html)
+ [ActiveModel::Callbacks](http://api.rubyonrails.org/classes/ActiveModel/Callbacks.html)

Pass: `rspec atm_spec.rb`

```ruby
ATM
  should declare callback hooks
    should be able to register a :command callback
  should run callbacks when #deposit or #withdraw is invoked
    should run the :command callbacks when #deposit is invoked
    should run the :command callbacks when #withdraw is invoked
  logging concern
    should log around #deposit
  text notification concern
    should invoke #send_sms after #deposit
  authentication concern
    account.valid_access? returns true
      should call Account#deposit
    account.valid_access? returns false
      should cancel #deposit
      should cancel after callbacks
```

