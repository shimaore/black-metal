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

      forEach: seem (cb) ->
        set_key = @__set_key()
        debug "forEach #{@class_name} #{set_key}"
        cursor = 0
        while cursor isnt '0'
          [cursor,keys] = yield @redis.sscanAsync set_key, cursor
          debug "forEach #{@class_name} #{set_key}: sscan", cursor, keys

          for key in keys
            debug "forEach #{@class_name} #{set_key}: cb", key
            yield cb key

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
        yield @redis.sinterstore @__tag_key(), "#{@class_name}-the-emtpy-tag-set-#{@key}"

      tags: seem ->
        yield @redis.smembersAsync @__tag_key()

      has_tag: seem (tag) ->
        yield @redis.sismemberAsync @__tag_key(), tag

    module.exports = RedisClient
