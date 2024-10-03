---
title: "今更見つけたredis_storeからredis_cache_storeに移行するときの注意点"
date: 2024-10-03T23:40:47+09:00
draft: false
---


Railsのcache機構では過去ビルトインのredisアダプタがなく、`gem 'redis-activesupport`によって使えるようになる`redis_store`が使われていた。

```ruby
config.cache_store = :redis_store, { url: 'redis://localhost:6379/' }
```

しかし、Rails 5.2からビルトインの`redis_cache_store`が導入された。

```ruby
config.cache_store = :redis_cache_store, { url: 'redis://localhost:6379/' }
```

`redis_store`と`redis_cache_store`の間で今更気づいた違いがあったのでメモ。
この記事の実行環境は以下です。

```bash
> ruby -v
ruby 3.3.5 (2024-09-03 revision ef084cc8f4) [arm64-darwin23]
> bin/rails c
Loading development environment (Rails 7.2.1)
rails-redis-cache-sandbox(dev)> Redis::VERSION
=> "5.3.0"
> docker compose exec redis redis-server --version
Redis server v=7.4.0 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=86eb6afc7b1d3b21
> cat Gemfile.lock | grep 'redis-activesupport'
    redis-activesupport (5.3.0)
```

## nxオプションとpxオプション
`redis_store`では`nx`オプションが使えるが、`redis_cache_store`では使えない。

```ruby
rails-redis-cache-sandbox(dev)> Rails.cache.class
=> ActiveSupport::Cache::RedisStore
rails-redis-cache-sandbox(dev)> Rails.cache.write("foo", true, nx: true, raw: true)
=> true
```

```bash
✗ docker compose exec redis redis-cli monitor
OK
1727966899.922581 [0 192.168.65.1:44932] "set" "foo" "true" "NX"
```

`redis_cache_store`では`nx`オプションを渡しても無視される。

```ruby
rails-redis-cache-sandbox(dev)> Rails.cache.class
=> ActiveSupport::Cache::RedisCacheStore
rails-redis-cache-sandbox(dev)> Rails.cache.write("foo", true, nx: true, raw: true)
=> true
```

```bash
1727967153.561036 [0 192.168.65.1:65366] "set" "foo" "true"
```

`redis_cache_store`は`unless_exist`オプションを使うとnxオプションと同じ挙動をする。

```ruby
rails-redis-cache-sandbox(dev)> Rails.cache.write("foo", true, unless_exist: true, raw: true)
=> false
```

```bash
1727967324.692603 [0 192.168.65.1:65366] "set" "foo" "true" "NX"
```

`redis_store`はそのまま`redis-rb`の`set`メソッドにオプションを渡しているのに対して、`redis_cache_store`は`ActiveSupport::Cache::RedisCacheStore`はオプションをそのまま渡したりはしていない模様。
https://github.com/rails/rails/blob/a11f0a63673d274c59c69c2688c63ba303b86193/activesupport/lib/active_support/cache/redis_cache_store.rb#L358-L379

`redis_store`から`redis_cache_store`に移行する際には、`nx`オプションを使っている箇所を`unless_exist`オプションに変更する必要がある。そのまま放置していても特にエラーが発生するわけではないので気づきにくいかもしれない。

同様に`redis_store`では`ex`オプションを渡せば`px`オプションを使うことができるが、
`redis_cache_store`で`ex`を渡しても無視されるので、こちらも移行時に気をつける必要がある。

```ruby
rails-redis-cache-sandbox(dev)> Rails.cache.class
=> ActiveSupport::Cache::RedisStore
rails-redis-cache-sandbox(dev)> Rails.cache.write("foo", true, ex: 10, raw: true)
=> "OK"
```

```bash
1727968022.718050 [0 192.168.65.1:28501] "set" "foo" "true" "EX" "10"
```

```rubyu
rails-redis-cache-sandbox(dev)> Rails.cache.class
=> ActiveSupport::Cache::RedisCacheStore
rails-redis-cache-sandbox(dev)> Rails.cache.write("foo", true, ex: 10, raw: true)
```

```bash
1727968250.025982 [0 192.168.65.1:51335] "set" "foo" "true"
```

`redis_cache_store`では`expires_in`オプションを使ってTTLを設定する。

```ruby
rails-redis-cache-sandbox(dev)> Rails.cache.class
=> ActiveSupport::Cache::RedisCacheStore
rails-redis-cache-sandbox(dev)> Rails.cache.write("foo", true, expires_in: 10.seconds, raw: true)
=> true
```

```bash
1727967808.851687 [0 192.168.65.1:44604] "set" "foo" "true" "PX" "10000"
```


## set + nx, px　と　setnxとsetex
`unless_exist`と`expires_in`オプションは`redis_store`でも使える。

```ruby
rails-redis-cache-sandbox(dev)> Rails.cache.class
=> ActiveSupport::Cache::RedisStore
rails-redis-cache-sandbox(dev)> Rails.cache.write("foo", true, unless_exist: true, raw: true)
=> false
```

```bash
1727968427.193977 [0 192.168.65.1:51686] "setnx" "foo" "true"
```

```ruby
rails-redis-cache-sandbox(dev)> Rails.cache.write("foo", true, expires_in: 10.seconds, raw: true)
=> "OK"
```

```bash
1727968788.970097 [0 192.168.65.1:51686] "setex" "foo" "10" "true"
```

`redis_cache_store`では`set`コマンドに`nx`や`px`オプションを渡しているのに対して、`redis_store`では`setnx`や`setex`コマンドを使っている。

redis的にはset + pxオプションを使ってほしい模様。
> Note: Since the SET command options can replace SETNX, SETEX, PSETEX, GETSET, it is possible that in future versions of Redis these commands will be deprecated and finally removed.
https://redis.io/docs/latest/commands/set/

もし`redis_store`を使っている場合は、`setnx`や`setex`という非推奨なコマンドを使っているという意識を持っておくと良さそう。





