    chai = require 'chai'
    chai.should()
    seem = require 'seem'

    describe 'The Agent', ->
      redis =
        _: {}
        hincrbyAsync: (key,field,increment) ->
          console.log key, field, increment
          redis._[key] ?= {}
          redis._[key][field] ?= 0
          Promise.resolve redis._[key][field] += increment
        hsetAsync: (key,field,value) ->
          console.log key, field, value
          redis._[key] ?= {}
          Promise.resolve redis._[key][field] = value
        hgetAsync: (key,field) ->
          console.log key, field
          redis._[key] ?= {}
          Promise.resolve redis._[key][field]
        saddAsync: (key,member) ->
          redis._[key] ?= new Set
          redis._[key].add member
          Promise.resolve 1
        sremAsync: (key,member) ->
          redis._[key] ?= new Set
          redis._[key].delete member
          Promise.resolve 1
        sscan: (key,cursor) ->
          keys = []
          redis._[key].forEach (key) -> keys.push key
          ["0",keys]
      policy_for = (agent) ->
        ->
          agent.name is 'lululu'

      egress_call_for = (agent) ->
          switch agent.key
            when 'lululu'
              Promise.resolve direction: '33643482771'
            else
              Promise.resolve null

      profile = 'booh!'
      {Agent,Queuer} = (require '..') redis, policy_for, egress_call_for, profile
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

