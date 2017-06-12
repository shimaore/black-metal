    Redis = require 'redis'
    Bluebird = require 'bluebird'
    Bluebird.promisifyAll Redis.RedisClient.prototype
    Bluebird.promisifyAll Redis.Multi.prototype

    seem = require 'seem'
    @name = 'black-metal:redis'
    debug = (require 'tangible') @name

    class RedisClient
      constructor: (@class_name,@key) ->
        debug 'new RedisClient', @class_name, @key

        throw new Error "RedisClient expects class name as first parameter" unless @class_name?
        throw new Error "RedisClient expects key as second parameter" unless @key?

Either the environment provides the proper variable(s) and we initialize the `@redis` field,
or it is up to a sub-class to define it.

        if process.env.BLACK_METAL_REDIS?
          @redis ?= Redis.createClient process.env.BLACK_METAL_REDIS

Properties
----------

      __property_key: ->
        "#{@class_name}-property-#{@key}"

      get: (property) ->
        @redis.hgetAsync @__property_key(), property

      set: seem (property,value) ->
        if value?
          yield @redis.hsetAsync @__property_key(), property, value
        else
          yield @redis.hdelAsync @__property_key(), property

      reset: (property) ->
        @set property, 0

      incr: (property,increment = 1) ->
        @redis.hincrbyAsync @__property_key(), property, increment

Set
---

      __set_key: ->
        "#{@class_name}-set-#{@key}"

      add: (value) ->
        if value?
          @redis.saddAsync @__set_key(), value

      remove: (value) ->
        if value?
          @redis.sremAsync @__set_key(), value

      has: (value) ->
        if value?
          @redis.sismemberAsync @__set_key(), value

      count: ->
        @redis.scardAsync @__set_key()

      clear: ->
        @redis.sinterstoreAsync @__set_key(), "#{@class_name}-the-emtpy-set-#{@key}"

      forEach: seem (cb) ->
        set_key = @__set_key()
        debug "forEach #{@class_name} #{set_key}"
        cursor = 0
        while cursor isnt '0'
          [cursor,keys] = yield @redis.sscanAsync set_key, cursor
          debug "forEach #{@class_name} #{set_key}: sscan", cursor, keys

          for key in keys
            debug "forEach #{@class_name} #{set_key}: cb", key
            try
              yield cb key
            catch error
              debug.dev "forEach cb on #{key}: #{error.stack ? error}"


        return

Ordered-Set
---

      __zset_key: ->
        "#{@class_name}-zset-#{@key}"

      sorted_add: (value,score = 0) ->
        if value?
          @redis.zaddAsync @__zset_key(), score, value

      sorted_incr: (value,delta = 1) ->
        if value?
          @redis.zincrbyAsync @__zset_key(), delta, value

      sorted_remove: (value) ->
        if value?
          @redis.zremAsync @__zset_key(), value

      sorted_has: (value) ->
        if value?
          @score(value)?

      score: (value) ->
        if value?
          @redis.zscoreAsync @__zset_key(), value

      sorted_count: ->
        @redis.zcardAsync @__zset_key()

      sorted_forEach: seem (cb) ->
        set_key = @__zset_key()
        debug "forEach #{@class_name} #{set_key}"
        cursor = 0
        while cursor isnt '0'
          [cursor,values] = yield @redis.zscanAsync set_key, cursor
          debug "forEach #{@class_name} #{set_key}: zscan", cursor, values

          while values.length > 1
            key = values.shift()
            score = values.shift()
            debug "forEach #{@class_name} #{set_key}: cb", key
            try
              yield cb key
            catch error
              debug.dev "sorted_forEach cb on #{key}: #{error.stack ? error}"

        return

Tags
----

      __tag_key: ->
        "#{@class_name}-tag-#{@key}"

      add_tag: seem (tag) ->
        if tag?
          yield @redis.saddAsync @__tag_key(), tag

      add_tags: seem (tags) ->
        if tags.length > 0
          yield @redis.saddAsync @__tag_key(), tags

      del_tag: seem (tag) ->
        if tag?
          yield @redis.sremAsync @__tag_key(), tag

      clear_tags: seem ->
        yield @redis.sinterstoreAsync @__tag_key(), "#{@class_name}-the-emtpy-tag-set-#{@key}"

      tags: seem ->
        yield @redis.smembersAsync @__tag_key()

      has_tag: seem (tag) ->
        yield @redis.sismemberAsync @__tag_key(), tag

    module.exports = RedisClient
