# Implement MyMongoid Lifecycle Callbacks

We will use `ActiveModel::Callbacks` to implement MyMongoid lifecycle callbacks.

## Objectives

+ Implement lifecycle callbacks using `ActiveModel::Callbacks`

# Declare Callbacks

Add the following gemspec dependencies

```
spec.add_dependency("activesupport", ["~> 4.0.3"])
spec.add_dependency("activemodel", ["~> 4.0.3"])
```

(remove older version of `activesupport` in gemspec)

**Implement** All callback hooks (`before`, `around`, `after`) for

+ delete
+ save
+ create
+ update

Pass: `rspec lifecycle_spec.rb -e 'all hooks'`

```
Should define lifecycle callbacks
  before, around, after hooks:
    should declare before hook for delete
    should declare around hook for delete
    should declare after hook for delete
    should declare before hook for save
    should declare around hook for save
    should declare after hook for save
    should declare before hook for create
    should declare around hook for create
    should declare after hook for create
    should declare before hook for update
    should declare around hook for update
    should declare after hook for update
```

**Implement** Only after callback hooks for

+ find
+ initialize

Pass: `rspec lifecycle_spec.rb -e 'only before hooks'`

```
Should define lifecycle callbacks
  only before hooks:
    should not declare before hook for find
    should not declare around hook for find
    should declare after hook for find
    should not declare before hook for initialize
    should not declare around hook for initialize
    should declare after hook for initialize
```

# Run Callbacks

Now we need to modify MyMongoid's CRUD methods to run these callbacks.

**Implement** Run create callbacks when creating a record.

Pass: `rspec lifecycle_spec.rb  -e 'run create callbacks'`

```
Should define lifecycle callbacks
  create callbacks
    should run callbacks when saving a new record
    should run callbacks when creating a new recor
```

**Implement** Run save callbacks when saving a record.

Pass: `rspec lifecycle_spec.rb  -e 'run save callbacks'`

```
Should define lifecycle callbacks
  run save callbacks
    should run callbacks when saving a new record
    should run callbacks when saving a persisted record
```

Exercise: Modify other CRUD methods on your own.