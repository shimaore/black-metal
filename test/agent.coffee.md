    chai = require 'chai'
    chai.should()
    seem = require 'seem'

    describe 'The Agent', ->
      redis =
        _: {}
        hincrbyAsync: (key,field,increment) ->
          console.log 'hincr', key, field, increment
          redis._[key] ?= {}
          redis._[key][field] ?= 0
          Promise.resolve redis._[key][field] += increment
        hsetAsync: (key,field,value) ->
          console.log 'hset', key, field, value
          redis._[key] ?= {}
          Promise.resolve redis._[key][field] = value
        hgetAsync: (key,field) ->
          console.log 'hget', key, field
          redis._[key] ?= {}
          Promise.resolve redis._[key][field]
        saddAsync: (key,member) ->
          console.log 'sadd', key, member
          redis._[key] ?= new Set
          redis._[key].add member
          Promise.resolve 1
        sremAsync: (key,member) ->
          console.log 'srem', key, member
          redis._[key] ?= new Set
          redis._[key].delete member
          Promise.resolve 1
        sscan: (key,cursor) ->
          console.log 'sscan', key, cursor
          keys = []
          redis._[key].forEach (key) -> keys.push key
          Promise.resolve ["0",keys]
      policy_for = (agent) ->
        (calls) ->
          console.log 'policy_for…'
          if agent.key is 'lululu'
            calls[0]
          else
            null

      egress_call_for = (agent) ->
        console.log 'egress_call_for', agent.key
        switch agent.key
          when 'lululu'
            Promise.resolve destination: '33643482771'
          else
            Promise.resolve null

      api = (cmd) ->
        console.log 'api', cmd
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
        yield agent.transition 'login'
        redis._.lalala.should.have.property 'state', 'idle'
        chai.expect(redis._['possibly-idle'].has 'lalala').to.be.true

      it 'should trigger call', seem ->
        queuer = new Queuer()
        agent = new Agent queuer, 'lululu'
        yield agent.transition 'login'

