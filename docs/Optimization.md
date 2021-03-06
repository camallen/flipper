# Optimization

## Memoizing Middleware

One optimization that flipper provides is a memoizing middleware. The memoizing middleware ensures that you only make one adapter call per feature per request.

This means if you check the same feature over and over, it will only make one Mongo, Redis, or whatever call per feature for the length of the request.

You can use the middleware from a Rails initializer like so:

```ruby
# create flipper dsl instance, see above Usage for more details
flipper = Flipper.new(...)

require 'flipper/middleware/memoizer'
config.middleware.use Flipper::Middleware::Memoizer, flipper
```

If you set your flipper instance up in an initializer, you can pass a block to the middleware and it will lazily load the instance the first time the middleware is invoked.

```ruby
# config/initializers/flipper.rb
module MyRailsApp
  def self.flipper
    @flipper ||= Flipper.new(...)
  end
end

# config/application.rb
config.middleware.use Flipper::Middleware::Memoizer, lambda {
  MyRailsApp.flipper
}
```

**Note**: Be sure that the middleware is high enough up in your stack that all feature checks are wrapped.

### Options

The Memoizer middleware also supports a few options. Use either `preload` or `preload_all`, not both.

* **`:preload`** - An `Array` of feature names (`Symbol`) to preload for every request. Useful if you have features that are used on every endpoint. `preload` uses `Adapter#get_multi` to attempt to load the features in one network call instead of N+1 network calls.
    ```ruby
    config.middleware.use Flipper::Middleware::Memoizer, flipper,
      preload: [:stats, :search, :some_feature]
    ```
* **`:preload_all`** - A Boolean value (default: false) of whether or not all features should be preloaded. Using this results in a `preload` call with the result of `Adapter#features`. Any subsequent feature checks will be memoized and perform no network calls. I wouldn't recommend using this unless you have few features (< 30?) and nearly all of them are used on every request.  
    ```ruby
    config.middleware.use Flipper::Middleware::Memoizer, flipper,
      preload_all: true
    ```

## Cache Adapters

Cache adapters allow you to cache adapter calls for longer than a single request and should be used alongside the memoization middleware to add another caching layer.

### Dalli

> Dalli is a high performance pure Ruby client for accessing memcached servers.

https://github.com/petergoldstein/dalli

Example using the Dalli cache adapter with the Memory adapter and a TTL of 600 seconds:

```ruby
dalli_client = Dalli::Client.new('localhost:11211')
memory_adapter = Flipper::Adapters::Memory.new
adapter = Flipper::Adapters::Dalli.new(memory_adapter, dalli_client, 600)
flipper = Flipper.new(adapter)
```
