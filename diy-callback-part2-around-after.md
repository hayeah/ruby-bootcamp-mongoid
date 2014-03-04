# DIY Callbacks Part 2 - Around & After Callbacks

In the last lesson we only imlemented the before callback. In this lesson, we'll implement the full callback chain with a recursive algorithm.

# Overview

Let's consider a class with a complex callback chain:

```
class Foo
  define_callbacks :save
  set_callback :save, :before, :before_1
  set_callback :save, :before, :before_2

  set_callback :save, :around, :around_1
  set_callback :save, :around, :around_2

  set_callback :save, :after, :after_1
  set_callback :save, :after, :after_2

  def save
    foo.run_callbacks(:save) do
    _save
  end

  protected

  def _save
    # actually saving to db
  end
end
```

When we save an instance of `Foo`,

```ruby
foo = Foo.new
foo.save
```

then the callback invokations would be in this order:

```
save
  before_1
  before_2
  around_1 {
    around_2 {
      _save
    }
  }
  after_2
  after_1
```

How might we implement this? To implement only the `before` and `after` callbacks, something like this pseudo-ruby code might work:

```ruby
# CallbackChain:invoke
def invoke(target,&main)
  # sort the callbacks in @chain into different kinds
  before_callbacks, around_callbacks, after_callbacks = sort_callbacks @chain

  before_callbacks.each do |cb|
    cb.invoke(target)
  end

  # how do we implement the around callbacks?
  main.call

  after_callbacks.each do |cb|
    cb.invoke(target)
  end
end
```

The `around` callbacks are trickier. We need to have them nested, like:

```ruby
around_callbacks[0] do
  around_callbacks[1] do
    # nested from 0..n
    around_callbacks[n] do
      main.call
    end
  end
end
```

Because the around callbacks requires nested calls, we can't use loops or iterators. Instead, we have to use a recursive algorithm (which turns out to be quite elegant).


# Writing A Simple Recursive "Function"

(Skip this if you are comfortable with recursion)

Let's review recursion by writing a recursive function to sum numbers from 1 to n.

Written as a loop:

```
def sum(n)
  i = n
  r = 0
  while i > 0
    r = r + i
    i -= 1
  end
  r
end
```

### By Recursion

There are 2 states, they are the loop variables `i` and `r`. The termination condition is `i > 0`.

In a recursive function, we use the function arguments as the "states". We define a helper function `_sum`, with two arguments as the "loop variables".

```ruby
def _sum(i,r)
  # ...
end
```

Then we change `sum` to call its helper `_sum` with the initial values (which are the same as the initial values of the loop variables):

```ruby
def sum(n)
  _sum(n,0)
end
```

The implementation for `_sum` is:

```ruby
def _sum(i,r)
  if i == 0
    r # end of "loop"
  else
    _sum(i-1,r+i) # this is the "loop body"
  end
end
```

Instead of changing states by changing the loop variables, `_sum` calls `_sum` (itself) again with different arguments.

This pattern is very common in recursive algorithms.

+ A helper function to call for the actual recursion
+ States are kept in arguments
+ To change the states, make recursive calls with different arguments

# Implement Before Callbacks Recursively

Let's create a recursive helper `CallbackChain#_invoke(i,target,block)`

+ `i` is the index number of the i-th callback in the CallbackChain.
+ `block` is the block of `run_callblack`.

The recursive call stack would look like:

```
invoke
  _invoke
    callback.invoke(target)
      _invoke
        callback.invoke(target)
          ...
            _invoke
              block.call
```

**Implement** Recursively run the `:before` callbacks.

Example:

```ruby
class Foo
  include MyMongoid::MyCallbacks

  define_callbacks :save

  set_callback :save, :before, :before_1
  set_callback :save, :before, :before_2

  def before_1
  end

  def before_2
  end

  def save
    run_callbacks(:save) {
      _save
    }
  end

  def _save
  end
end
foo.save
```

then the call stack looks like:

```
invoke
  _invoke
    before_1
    _invoke
      before_2
      _invoke
        block.call
```

Pass: `rspec callbacks-02_spec.rb -e 'run before callbacks recursively'`

```
MyMongoid::MyCallbacks
  run before callbacks recursively
    should recursively call _invoke
    should call the before methods in order
```

# Implement After Callbacks

**Implement** Recursively calling after callbacks.

Example:

```ruby
class Foo
  # ...
  set_callback :save, :after, :after_1
  set_callback :save, :after, :after_2
  # ...
end
foo.save
```

the call stack looks like:

```
invoke
  _invoke
      _invoke
          _invoke
            block.call
        after_2
    after_1
```

Pass: `rspec callbacks-02_spec.rb -e 'run after callbacks recursively'`

```
MyMongoid::MyCallbacks
  run after callbacks recursively
    should call the after methods in order
```

# Implement Around Callbacks

**Implement** Call around callbacks recursively.

Hint: modify `Callback#invoke` to accept a block, so within the around callback method `yield` would continue the callback chain.

Example:

```ruby
class Foo
  # ...
  set_callback :save, :around, :around_1
  set_callback :save, :around, :around_2

  def around_1
    around_1_top
    yield
    around_1_bottom
  end

  def around_2
    around_2_top
    yield
    around_2_bottom
  end
  # ...
end
foo.save
```

the call stack looks like:

```
invoke
  _invoke
    around_1
      around_1_top
      yield {
        _invoke
          around_2
            around_2_top
            yield {
              _invoke
                block.call
            }
            around_2_bottom
      }
      around_1_bottom
```

Pass: `rspec callbacks-02_spec.rb -e 'run around callbacks recursively'`

```
MyMongoid::MyCallbacks
  run around callbacks recursively
    should call the around methods in order
```

# Bonus

+ If a before callback returns `false` (and only `false`, not nil, [], or anything else), then it should terminate the recursive callback chain.
