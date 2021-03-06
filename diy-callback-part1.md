# DIY Callbacks Part 1 - Minimal Implementation

We'll implement `ActiveSupport::Callbacks` in a series of lessons. In this lesson we'll start with a simplified version.

# ActiveSupport Implementation Overview

To implement the callbacks feature, `ActiveSupport::Callbacks` does the following:

+ `.define_callbacks(name)` creates a *class_attribute* called `_{name}_callbacks`
  + [code](https://github.com/rails/rails/blob/409fbffc92dfe612db129d5bb4e99a8807854958/activesupport/lib/active_support/callbacks.rb#L729-L730)
+ `.set_callback` adds a callback to an instance of `CallbackChain`
  + [code](https://github.com/rails/rails/blob/409fbffc92dfe612db129d5bb4e99a8807854958/activesupport/lib/active_support/callbacks.rb#L468)
+ Represent a callback with the `Callback` class
  + [code](https://github.com/rails/rails/blob/409fbffc92dfe612db129d5bb4e99a8807854958/activesupport/lib/active_support/callbacks.rb#L338)
+ `#run_callbacks` runs the callbacks stored in the `CallbackChain`
  + [code](https://github.com/rails/rails/blob/409fbffc92dfe612db129d5bb4e99a8807854958/activesupport/lib/active_support/callbacks.rb#L86)

A callback might be specified as a symbol (e.g. `before_save`). The `Callback` handles the logic of invoking that symbol as a method. The `Callback` class [supports many types of callbacks](https://github.com/rails/rails/blob/409fbffc92dfe612db129d5bb4e99a8807854958/activesupport/lib/active_support/callbacks.rb#L421).

The `CallbackChain` class is basically a fancy `Array` that transforms a number of added callbacks into lambdas, and calls them in order. `CallbackChain#compile` [builds a callable lambda](https://github.com/rails/rails/blob/409fbffc92dfe612db129d5bb4e99a8807854958/activesupport/lib/active_support/callbacks.rb#L511-L513) that optimizes the callback chain.

We'll do `#compile` later. Ignore it for now.

# Implementation Overview

`ActiveSupport::Callbacks` is pretty complicated. To keep it simple

+ We'll only support symbols as callbacks
+ We won't implement callbacks chain compilation (we'll do that in a later lesson)

In this lesson we'll support only the before callback. Just enough to make the following work:

```ruby
class Model
  define_callbacks :save
  set_callback :save, :before, :before_save

  def save
    run_callbacks(:save) do
      save_to_db
    end
  end

  def before_save
    "before save"
  end
end

model = Model.new
model.save # => "before save"
```

# define_callbacks

It should declare a class attribute to store the callbacks in a `CallbackChain` object.

**Implement** Define `.define_callbacks(name,opts={})`.

Hint: `ActiveSupport::Callbacks`'s API `.define_callbacks(*names)` allows multiple names to define multiple hooks at the same time. Let's just allow one name at a time.

Hint: Use [class_attribute](http://api.rubyonrails.org/classes/Class.html#method-i-class_attribute). Once defined, it can be used as instance reader as well.

Pass: `rspec callbacks_spec.rb -e '.define_callbacks'`

```
MyMongoid::MyCallbacks
  .define_callbacks
    should declare the class attribute #{name}_callbacks
    should initially return an instance of CallbackChain
```

# Callback Class

When we `set_callback` with a method name like this:

```
klass.set_callback(:save,:before,:before_save)
```

We don't want to store the callback directly in `CallbackChain` as a symbol. We will use a `Callback` class to provide a layer of abstraction.

**Implement** The `MyMongoid::MyCallbacks::Callback` class.

The method signature for `#initialize` is:

```
# @param [Symbol] filter The callback method name.
# @param [Symbol] kidn The kind of callback (eg :before,:around,:after)
Callback#initialize(filter,kind)
```

Example:
```
# The Callback instance is created by set_callback.
cb = Callback.new(:before_save,:before)
# Later when run_callbacks runs the callback, it gets the object as the target.
cb.invoke(object)
```
Pass: `rspec callbacks_spec.rb -e 'MyMongoid::MyCallbacks::Callback'`

```
MyMongoid::MyCallbacks
  MyMongoid::MyCallbacks::Callback
    should have the #kind attr_reader
    should have the #filter attr_reader
    should call the target object's method when #invoke is called
```

# CallbackChain

We need to be able to add multiple callbacks to the CallbackChain. When it is invoked, it calls all the callbacks in its chain. Also, the CallbackChain is responsible of calling `run_callbacks`'s block.

**Implement** `MyMongoid::Callbacks::CallbackChain`

Example:

```ruby
cb1 = Callback.new # ...
cb2 = Callback.new # ...
cbchain = CallbackChain.new
cbchain.append(cb1)
cbchain.append(cb2)
cbchain.invoke(target) {
  # run_callback's block
}
```

Pass: `rspec callbacks_spec.rb -e 'MyMongoid::MyCallbacks::CallbackChain'`

Note: the first before callback added should be called first, then the second. (It's the reverse ordering for after callbacks.)

```
MyMongoid::MyCallbacks
  MyMongoid::MyCallbacks::CallbackChain
    should initially be empty
    should initially set @chain to be the empty array
    should be able to append callbacks to the chain
    #invoke
      should call the callbacks in order, then call the block
```

# set_callback

With the work we've already done, this is easy. Just append a callback to the named callback chain.

**Implement** `.set_callback(name,kind,filter)` should append a callback to callback chain.

Example: `klass.set_callback(:save,:before,:before_save)`

# run_callbacks

Finally, `#run_callbacks` invokes a named callback chain.

**Implement** `#run_callbacks(name)`

Example:

```
klass.set_callback(:save,:before,:before_save)
klass.run_callbacks(:save) do
  main_method
end
```

Pass: `rspec callbacks_spec.rb -e '#run_callbacks'`

```
MyMongoid::MyCallbacks
  #run_callbacks
    should invoke the callback chain
```

# Bonus

**Implement** Extend `Callback#invoke(target)` to support String, Proc, and Object as filter.

