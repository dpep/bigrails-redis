# BigRails::Redis

A simple Redis connection manager for Rails applications with distributed and [ConnectionPool](https://github.com/mperham/connection_pool) support.

## Installation

Add to your Gemfile:

    $ bundle add bigrails-redis

Create a redis configuration file:

    $ touch config/redis.rb

## Usage

### Configuring Connections

The configuration file (`config/redis.rb`) is just a plain Ruby file that will be evaluated when a connection is requested. Use the `connection` DSL method to declare your connections. The method will yield a block and you're expected to return a configuration hash.

```ruby
# Simple hardcoded example.
connection(:default) do
  {
    url: "redis://localhost"
  }
end

# Do something more dynamic.
%w(
  cache
  foobar
).each do |name|
  connection(name) do
    {
      url: Rails.application.credentials.fetch("#{name}_redis")
    }.tap do |options|
      # Maybe in CI, you need to change the host.
      if ENV['CI']
        options[:host] = "redishost"
      end
    end
  end
end

# Connection pool support.
connection(:sidekiq) do
  {
    url: "redis://localhost/2"
    pool_timeout: 5,
    pool_size: 5
  }
end

# Distributed Redis support.
connection(:baz) do
  {
    url: [
      "redis://host1",
      "redis://host2",
      "redis://host3"
    ]
  }
end
```

The configuration hash, by default, is passed to `ActiveSupport::Cache::RedisCacheStore.build_redis(...)`. This is a Rails supplied helper which allows for more options than demostrated above. You'll want to [check out its source](https://github.com/rails/rails/blob/main/activesupport/lib/active_support/cache/redis_cache_store.rb#L77-L100) to get a better idea of what it supports.

### Accessing Connections

To access connections inside the application, you can do the following:

```ruby
Rails.application.redis #=> Redis Registry

Rails.application.redis.for(:default) #=> Redis
Rails.application.redis.for(:cache) #=> Redis
Rails.application.redis.for(:foobar) #=> Redis
Rails.application.redis.for(:sidekiq) #=> ConnectionPool
```

If needed, you can request a [wrapped connection pool](https://github.com/mperham/connection_pool#migrating-to-a-connection-pool):

```ruby
Rails.application.redis.for(:pooled_connection, wrapped: true)
```

If you request a wrapped connection for a non-pooled connection, it'll just return the original, plain `Redis` connection object. Rails already modifies `Redis` to add `ConnectionPool`-like behavior by adding a `with` method that yields the connection itself.

### Verifying Connections

This library also allows you to verify connections on demand. If you want, perform the verification in a startup health check to make sure all your connections are valid. It will perform a simple [`PING` command](https://redis.io/commands/PING). An error will be raised if the connection bad.

```ruby
# Verify all connections:
Rails.application.redis.verify!

# Verify a single connection:
Rails.application.redis.verify!(:foobar)
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and the created tag, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/ngan/bigrails-redis. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [code of conduct](https://github.com/ngan/bigrails-redis/blob/master/CODE_OF_CONDUCT.md).

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

## Code of Conduct

Everyone interacting in the BigRails::Redis project's codebases, issue trackers, chat rooms and mailing lists is expected to follow the [code of conduct](https://github.com/ngan/bigrails-redis/blob/master/CODE_OF_CONDUCT.md).
