    chai = require 'chai'
    chai.should()
    seem = require 'seem'
    debug = (require 'debug') 'black-metal:test:agent'

    describe 'The Agent', ->
      redis =
        _: {}
        hincrbyAsync: (key,field,increment) ->
          debug 'hincr', key, field, increment
          redis._[key] ?= {}
          redis._[key][field] ?= 0
          Promise.resolve redis._[key][field] += increment
        hsetAsync: (key,field,value) ->
          debug 'hset', key, field, value
          redis._[key] ?= {}
          Promise.resolve redis._[key][field] = value
        hgetAsync: (key,field) ->
          debug 'hget', key, field
          redis._[key] ?= {}
          Promise.resolve redis._[key][field]
        saddAsync: (key,member) ->
          debug 'sadd', key, member
          redis._[key] ?= new Set
          redis._[key].add member
          Promise.resolve 1
        sremAsync: (key,member) ->
          debug 'srem', key, member
          redis._[key] ?= new Set
          redis._[key].delete member
          Promise.resolve 1
        sscan: (key,cursor) ->
          debug 'sscan', key, cursor
          keys = []
          redis._[key].forEach (key) -> keys.push key
          Promise.resolve ["0",keys]
      policy_for = (agent) ->
        (calls) ->
          debug 'policy_for…'
          if agent.key is 'lululu'
            calls[0]
          else
            null

      egress_call_for = (agent) ->
        debug 'egress_call_for', agent.key
        switch agent.key
          when 'lululu'
            Promise.resolve destination: '33643482771'
          else
            Promise.resolve null

      api = (cmd) ->
        debug 'api', cmd
        true

      profile = 'booh!'
      {Agent,Queuer} = (require '../queuer') redis, policy_for, egress_call_for, profile, api
      key = 'lalala'

      it 'should increment external calls', seem ->
        queuer = {}
        agent = new Agent queuer, key
        yield agent.increment_external_calls()
        yield agent.decrement_external_calls()
        redis._.lalala.should.have.property 'external_calls', 0

      it 'should transition', seem ->
        queuer = new Queuer()
        agent = new Agent queuer, key
        ok = yield agent.transition 'login'
        ok.should.be.true
        redis._.lalala.should.have.property 'state', 'idle'
        chai.expect(redis._['possibly-idle'].has 'lalala').to.be.true

      it 'should trigger call', seem ->
        queuer = new Queuer()
        agent = new Agent queuer, 'lululu'
        ok = yield agent.transition 'login'
        ok.should.be.true
        redis._.lululu.should.have.property 'state', 'in_call'
