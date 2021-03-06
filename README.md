# ConCache

ConCache (Concurrent Cache) is an ETS based key/value storage with following additional features:

* row level synchronized writes (inserts, read/modify/write updates, deletes)
* TTL support
* modification callbacks

There is currently no proper reference documentation. For quick info, you can take a look at `con_cache.ex` where all interface functions are listed, and typespecs are provided.

## Usage in OTP applications

Setup project and app dependency in your `mix.exs`:

```elixir
  ...

  defp deps do
    [{:con_cache, "~> 0.6.0"}, ...]
  end

  def application do
    [applications: [:con_cache, ...], ...]
  end

  ...
```

A cache can be started using `ConCache.start` or `ConCache.start_link` functions. Both functions take two arguments - the first one being a list of ConCache options, and the second one a list of GenServer options for the process being started.

Typically you want to start the cache from a supervisor:

```elixir
Supervisor.start_link(
  [
    ...
    worker(ConCache, [[], [name: :my_cache]])
    ...
  ],
  ...
)
```

Notice the `name: :my_cache` option. The resulting process will be registered under this alias. Now you can use the cache as follows:

```elixir
# Note: all of these requests run in the caller process, without going through
# the started process.

ConCache.put(:my_cache, key, value)         # inserts value or overwrites the old one
ConCache.insert_new(:my_cache, key, value)  # inserts value or returns {:error, :already_exists}
ConCache.get(:my_cache, key)
ConCache.delete(:my_cache, key)

ConCache.update(:my_cache, key, fn(old_value) ->
  # This function is isolated on a row level. Modifications such as update, put, delete,
  # on this key will wait for this function to finish.
  # Modifications on other items are not affected.
  # Reads are always dirty.

  new_value
end)

# Similar to update, but executes provided function only if item exists.
# Otherwise returns {:error, :not_existing}
ConCache.update_existing(:my_cache, key, fn(old_value) ->
  new_value
end)


# Returns existing value, or calls function and stores the result.
# If many processes simultaneously invoke this function for the same key, the function will be
# executed only once, with all others reading the value from cache.
ConCache.get_or_store(:my_cache, key, fn() ->
  initial_value
end)


# Executes function only if item exists. The result is either the function's return value, or nil.
ConCache.with_existing(:my_cache, key, fn() ->
  ...
end)
```

Dirty modifiers operate directly on ets record without trying to acquire the row lock:

```elixir
ConCache.dirty_put(:my_cache, key, value)
ConCache.dirty_insert_new(:my_cache, key, value)
ConCache.dirty_delete(:my_cache, key)
ConCache.dirty_update(:my_cache, key, fn(old_value) -> ... end)
ConCache.dirty_update_existing(:my_cache, key, fn(old_value) -> ... end)
ConCache.dirty_get_or_store(:my_cache, key, fn() -> ... end)
```

### Callback

You can register your own function which will be invoked after an element is stored or deleted:

```elixir
worker(ConCache, [[callback: fn(data) -> ... end], [name: :my_cache]])

ConCache.put(:my_cache, key, value)         # fun will be called with {:update, cache_pid, key, value}
ConCache.delete(:my_cache, key)             # fun will be called with {:delete, cache_pid, key}
```

The delete callback is invoked before the item is deleted, so you still have the chance to fetch the value from the cache and do something with it.

### TTL

```elixir
worker(ConCache, [
  [
    ttl_check: :timer.seconds(1),
    ttl: :timer.seconds(5)
  ],
  [name: :my_cache]
])
```

This example sets up item expiry check every second. The default expiry for all cache items is 5 seconds. Since ttl_check interval is 1 second, the item lifetime might be at most 6 seconds.

However, the item lifetime is renewed on every modification. Reads don't extend ttl, but this can be changed when starting cache:

```elixir
worker(ConCache, [
  [
    ttl_check: :timer.seconds(1),
    ttl: :timer.seconds(5),
    touch_on_read: true
  ],
  [name: :my_cache]
])
```

In addition, you can manually renew item's ttl:

```elixir
ConCache.touch(:my_cache, key)
```

And you can override ttl for each item:

```elixir
ConCache.put(:my_cache, key, %ConCache.Item{value: value, ttl: ttl})

ConCache.update(:my_cache, key, fn(old_value) ->
  %ConCache.Item{value: new_value, ttl: ttl}
end)
```

If you use ttl value of 0 the item never expires.
In addition, unless you set `ttl_check` interval, the ttl check process will not be started, and items will never expire.

TTL check __is not__ based on brute force table scan, and should work reasonably fast assuming the check interval is not too small. I generally recommend `ttl_check` to be at least 1 second, possibly more, depending on the cache size and desired ttl.

## Supervision

A call to `ConCache.start_link` (or `start`) creates the so called _cache owner process_. This is the process that is the owner of the underlying ETS table and also the process where TTL checks are performed. No other operation (such as get or put) runs in this process.

As you've seen from the examples above, it's your responsibility to place the cache owner process into your own supervision tree. This gives you the control of cache cleanup when some subtree terminates (since a termination of the owner process will release the ETS table).

If for some reason `:con_cache` application is terminated, all cache owner processes will be terminated as well, regardless of the fact that they do not reside in the `:con_cache` supervision tree.

## Process alias

Functions `ConCache.start` and `ConCache.start_link` return standard `{:ok, pid}` result. You can interface with the cache using this pid. As mentioned, cache operations are not running through this process - the pid is just used to discover the corresponding ETS table.

Most of the time using pid to interface the cache is not appropriate. Just like in examples above, you usually want to give some alias to your cache, and then access it via this alias. In the examples above, we used `name: :some_alias` to provide local alias. Alternatively, you can use following formats for `name` option:

```elixir
{:global, some_alias}         # globally registered alias
{:via, module, some_alias}    # registered through some module (e.g. gproc)
```

In this case, you can just pass the same tuple to other `ConCache` functions. For example, to use the cache with [gproc](https://github.com/uwiger/gproc), you can do something like this:

```elixir
ConCache.start_link([], name: {:via, :gproc, :my_cache})
...
ConCache.put({:via, :gproc, :my_cache}, :some_key, :some_value)
```

## Inner workings

### ETS table
The ETS table is always public, and by default it is of _set_ type. Some ETS parameters can be changed:

```elixir
ConCache.start_link(ets_options: [
  :named_table,
  {:name, :test_name},
  :ordered_set,
  {:read_concurrency, true},
  {:write_concurrency, true},
  {:heir, heir_pid}
])
```

The allowed types are set and ordered_set.

Additionally, you can override ConCache, and access ets directly:

```elixir
:ets.insert(ConCache.ets(cache), {key, value})
```

Of course, this completely overrides additional ConCache behavior, such as ttl, row locking and callbacks.

### Locking

To provide isolation, custom implementation of mutex is developed. This enables that each update operation is executed in the caller process, without the need to send data to another sync process.

When a modification operation is called, the ConCache first acquires the lock and then performs the operation. The acquiring is done using the pool of lock processes that reside in the ConCache supervision tree. The pool contains as many processes as there are schedulers.

If the lock is not acquired in a predefined time (default = 5 seconds, alter with _acquire\_lock\_timeout_ ConCache parameter) an exception will be generated.

You can use explicit isolation to perform isolated reads if needed. In addition, you can use your own lock ids to implement bigger granularity:

```elixir
ConCache.isolated(cache, key, fn() ->
  ConCache.get(cache, key)    # isolated read
end)

# Operation isolated on an arbitrary id. The id doesn't have to correspond to a cache item.
ConCache.isolated(cache, my_lock_id, fn() ->
  ...
end)

# Same as above, but immediately returns {:error, :locked} if lock could not be acquired.
ConCache.try_isolated(cache, my_lock_id, fn() ->
  ...
end)
```

Keep in mind that these calls are isolated, but not transactional (atomic). Once something is modified, it is stored in ets regardless of whether the remaining calls succeed or fail.
The isolation operations can be arbitrarily nested, although I wouldn't recommend this approach.

### TTL

When ttl is configured, the owner process works in discrete steps using `:erlang.send_after` to trigger the next step.

When an item ttl is set, the owner process receives a message and stores it in its internal structure without doing anything else. Therefore, repeated touching of items is not very expensive.

In the next discrete step, the owner process first applies the pending ttl set requests to its internal state. Then it checks which items must expire at this step, purges them, and calls `:erlang.send_after` to trigger the next step.

This approach allows the owner process to do fairly small amount of work in each discrete step.

### Consqeuences

Due to the locking and ttl algorithms just described, some additional processing will occur in the owner processes. The work is fairly optimized, but I didn't invest too much time in it.
For example, lock processes currently use pure functional structures such as `HashDict` and `:gb_trees`. This could probably be replaced with internal ETS table to make it work faster, but I didn't try it.

Due to locking and ttl inner workings, multiple copies of each key exist in memory. Therefore, I recommend avoiding complex keys.

## Status

ConCache has been used in production to manage several thousands of entries served to up to 4000 concurrent clients, on the load of up to 2000 reqs/sec. I don't maintain that project anymore, so I'm not aware of its current status.