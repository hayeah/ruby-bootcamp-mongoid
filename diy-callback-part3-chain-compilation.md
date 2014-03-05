# DIY Callbacks Part 3 - Callback Chain Compilation

We mentioned earlier that `CallbackChain#compile` optimizes the callback chain. The `compile` method basically transforms the recursive `_invoke` algorithm into a chain of nested lambdas. In this lesson we'll see how this optimization is done.

# Overview

With `invoke`, a callback chain stack might look like

```
CallbackChain#invoke
  CallbackChain#_invoke
    Callback#invoke
      before_1
    CallbackChain#_invoke
      Callback#invoke
        before_2
        CallbackChain#_invoke
          Callback#invoke
            around {
              main.call
              CallbackChain#_invoke
                Callback#invoke
                  after_2
                CallbackChain#_invoke
                  Callback#invoke
                    after_1
            }
```

Our gaol is to get rid of all those `_invoke` calls when we run the callback chain. We want the stack to look like

```
before_1
  before_2
    around {
      main.call
      after_2
        after_1
    }
```

The compiled callback chain is a lambda that looks like:

```
lambda { |target,&main|
  target.before_1
  lambda { |target,&main|
    target.before_2
    lambda { |target,&main|
      target.around do
        main.call
        lambda { |target,_|
          target.after_2
          lambda { |target,_|
            target.after_1
          }.call(target,nil)
        }.call(target,nil)
      end
    }.call(target,&main)
  }.call(target,&main)
}.call(target,&main)
```

All the lambdas in this chain take the same arguments:

+ `target` is the instance that's running the callbacks.
+ `&main` is the block of a `run_callbacks` call.

`target` is passed into each lambda, and it recevies the actual callback methods (e.g. `before_1`, `around`, etc.). The `&main` block is run by the inner around callback.

By doing the compilation ahead of time, we can avoid the overhead of deciding what to do for a type of callback.

# Breaking Apart The Nested Lambda

There is another way to represent the nested lambda above. Let's first name the anonymous lambdas (k1 to k5):

```ruby
k1 = lambda { |target,&main|
  target.before_1
  k2 = lambda { |target,&main|
    target.before_2
    k3 = lambda { |target,&main|
      target.around do
        main.call
        k4 = lambda { |target,_|
          target.after_2
          k5 = lambda { |target,_|
            target.after_1
          }
          k5.call(target,nil)
        }
        k4.call(target,nil)
      end
    }
    k3.call(target,&main)
  }
  k2.call(target,&main)
}
k1.call(target,&main)
```

Now we can move the lambdas outside:

```
k5 = lambda { |target,_|
  target.after_1
}

k4 = lambda { |target,_|
  target.after_2
  k5.call(target,nil)
}

k3 = lambda { |target,&main|
  target.around do
    main.call
    k4.call(target,nil)
  end
}

k2 = lambda { |target,&main|
  target.before_2
  k3.call(target,&main)
}

k1 = lambda { |target,&main|
  target.before_1
  k2.call(target,&main)
}

k1.call(target,&main)
```

When k1 is called, it calls k2, which calls k3, which calls k4, etc. We got rid of the nestedness.

We used `k` to name these lambdas because they represent a ["program continuation"](http://en.wikipedia.org/wiki/Continuation). A "continuation" is basically a way of representing "the next computation". For example, after calling `before_1`, the "next computation" is supposed to call `before_2`, and k2 represents that.

You can go to a computation by calling its continuation. For example, `k1` "goto" the next computation by calling `k2`.

Continuation is an extremely powerful idea. Here's a short list of what continuation can do:

+ [tail-call optimization](http://stackoverflow.com/questions/310974/what-is-tail-call-optimization)
+ non-deterministic programming ([SICP section 4.3](http://eli.thegreenplace.net/2007/12/28/sicp-section-431) uses continuation to implement backtracking)
+ saving the state of a program
+ co-oroutines and generators
+ threads

**Bonus**: if you are familiar with scheme, and want to be very confused, then read [Continuations by example: Exceptions, time-traveling search, generators, threads, and coroutines](http://matt.might.net/articles/programming-with-continuations--exceptions-backtracking-search-threads-generators-coroutines/)

# Bottom-up Callback Chain Compilation

Our compilation process is slightly different from the above. We build the nested lambda by taking the `run_callbacks` block as the initial continuation, For each callback in the chain we create a new continuation that calls the previous continuation. We'll do that in reverse, starting with the last added callback first.

Here's an example:

Supposed we declare these callbacks (the order for kinds of callbacks doesn't matter. A `:before` callback can be added after an `:around` callback):

```
set_callback :save, :after, :after_1
set_callback :save, :before, :before_1
set_callback :save, :around, :around_1
set_callback :save, :after, :after_2
set_callback :save, :before, :before_2
```

Then the build-up process is like:

```ruby
# &main is the run_callbacks body
k0 = lambda {|_,&main| main.call }

# before_2
k1 = lambda { |target,&main|
  before_2_cb.call(target)
  k0.call(target,&main)
}

# after_2
k2 = lambda { |target,&main|
  k1.call(target,&main)
  after_2_cb.call(target)
}

# around
k3 = lambda { |target,&main|
  k2.call(target) do
    around_1_cb.call(target,&main)
  end
}

# before_1
k4 = lambda { |target,&main|
  before_1_cb.call(target)
  k3.call(target,&main)
}

# after_1
k5 = lambda { |target,&main|
  k4.call(target,&main)
  after_1_cb.call(target)
}
```

We can derive the nested lambda by substitution:

```ruby
k5 = lambda { |target,&main|
  k4 = lambda { |target,&main|
    before_1_cb.call(target)
    k3 = lambda { |target,&main|
      k2 = lambda { |target,&main|
        k1 = lambda { |target,&main|
          before_2_cb.call(target)
          k0 = lambda { |_,&main|
            main.call
          }.call(target,&main)
        }.call(target,&main)
        after_2_cb.call(target)
      }.call(target) do
        around_1_cb.call(target,&main)
      end
    }.call(target,&main)
  }.call(target,&main)
  after_1_cb.call(target)
}
```

Pay attention to how the `k3` continuation wraps the `:around` callback around `&main` before passing it into `k2`. To illustrate this change more clearly, let's rename the block argument to `main2` to denote the wrapped `&main`:

```ruby
k5 = lambda { |target,&main|
  k4 = lambda { |target,&main|
    before_1_cb.call(target)
    k3 = lambda { |target,&main|
      k2 = lambda { |target,&main2|
        k1 = lambda { |target,&main2|
          before_2_cb.call(target)
          k0 = lambda { |_,&main2|
            main2.call
          }.call(target,&main)
        }.call(target,&main)
        after_2_cb.call(target)
      }.call(target) do # this is main2
        around_1_cb.call(target,&main)
      end
    }.call(target,&main)
  }.call(target,&main)
  after_1_cb.call(target)
}
```

Once the chain is compiled, `run_callbacks` can run the callbacks by calling `k5.call(self,&block)`

# Callback#compile

**Implement** Callback#compile returns a lambda that invokes the callback method of a target.

Hint: the return lambda should accept an argument and a block. `lambda { |target,&block| ... }`

Example:

```ruby
fn = callback.compile
fn.call(target)
# or for an around callback
fn.call(target) do
  puts "top"
  yield
  puts "bottom"
end
```

Pass: `rspec callbacks-03_spec.rb -e 'Callback#compile'`

```
MyMongoid::MyCallbacks
  Callback#compile
    should invoke the callback method
    should invoke the callback method with a block if given one
```



# Compile Before Callbacks

The `#compile` method should take the callbacks added in `@chain` and build a nested lambda. The compiled lambda should be memoized in the `@callbacks` variable. Like how the `#invoke` has a recursive `#_invoke` helper, we'll implement `#compile` with the `#_compile` helper.

**Implement** CallbackChain#compile

Example:

```ruby
fn = callbacks.compile
fn.call(target) do
  main_logic
end

# the compilation result is memoized
callbacks.compile == fn

# if new callback is added, reset the memoization
callbacks.append(cb)
callbacks.compile != fn # get a new result
```

For the class `Foo`

```ruby
class Foo
  # ...
  define_callbacks :save

  set_callback :save, :before, :before_1
  set_callback :save, :before, :before_2

  def save
    run_callbacks(:save) {
      _save
    }
  end
  # ...
end
```

the compiled callback chain looks like:

```ruby
lambda { |target,&main|
  compiled_before_1_callback_call(target)
  lambda { |target,&main|
    compiled_before_2_callback_call(target)
    lambda { |_,&main|
      main.call
    }.call(target,&main)
  }.call(target,&main)
}
```

Hint: Use a recursive helper `_compile(k,i)`

+ `k` is the continuation
+ `i` is the index of the current callback
+ A continuation should an argument and a block: `lambda {|target,&block| ... } `
+ You should call `_compile` from `compile` with an initial continuation k

Hint: You should start with the last callback in the chain.

Pass: `rspec callbacks-03_spec.rb -e "compile before callbacks recursively"`

```
MyMongoid::MyCallbacks
  compile before callbacks recursively
    should return a lambda
    should recursively call _compile
    should call the before methods in order when compiled lambda is run
```

# Memoize Compilation

**Implement** `CallbackChain#compile` memoize compiled result in the `@callbacks` variable

Pass: `rspec callbacks-03_spec.rb -e 'CallbackChain#compile memoize'`

```
MyMongoid::MyCallbacks
  CallbackChain#compile memoize
    should memoize the compiled callback chain
    should reset memoization if a new callback is added
```

# Compile Around Callbacks

**Implement** `CallbackChain#compile` should support around callbacks

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
```

The around callback is a bit trickier. Let's build the continuations step by step:

```
k0 = lambda { |target,&main| main.call }
k1 = lambda { |target,&main|
  k0.call(target) do
    compiled_around_2_callback.call(target,&main)
  end
}
k2 = lambda { |target,&main|
  k1.call(target) do
    compiled_around_1_callback.call(target,&main)
  end
}
```

And the compilation result is:

```ruby
lambda { |target,&main| # k2
  lambda { |target,&main| # k1
    lambda { |target,&main| # k0
      main.call
    }.call(target) do
      compiled_around_2_callback.call(target,&main)
    end
  }.call(target) do
    compiled_around_1_callback.call(target,&main)
  end
}
```

Pass: `rspec callbacks-03_spec.rb -e "compile around callbacks recursively"`

```
MyMongoid::MyCallbacks
  compile around callbacks recursively
    should call the around methods in order when compiled lambda is run
```

# Compile After Callbacks

**Implement** `CallbackChain#compile` should support after callbacks

Example:

```ruby
class Foo
  # ...
  set_callback :save, :after, :after_1
  set_callback :save, :after, :after_2
  # ...
end
```

and the compiled chain is like:

```ruby
lambda { |target,&main|
  lambda { |target,&main|
    lambda { |target,&main|
      main.call
    }.call(target,&main)
    compiled_after_1_callback_call.call(target)
  }.call(target,&main)
  compiled_after_2_callback_call.call(target)
}
```

Pass: `rspec callbacks-03_spec.rb -e "compile after callbacks recursively"`

```ruby
MyMongoid::MyCallbacks
  compile after callbacks recursively
    should call the after methods in order
```

# run_callbacks

**Implement** `run_callbacks` should use compiled lambda to run callbacks.

Pass: `rspec callbacks-03_spec.rb -e "run_callbacks should use the compiled lambda to run callbacks"`

```
MyMongoid::MyCallbacks
  run_callbacks should use the compiled lambda to run callbacks
    should invoke #compile
    should run callback methods
```