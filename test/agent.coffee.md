    chai = require 'chai'
    chai.should()
    seem = require 'seem'
    debug = (require 'debug') 'black-metal:test:agent'

    describe 'The Agent', ->
      redis =
        _: {}
        hsetAsync: (key,field,value) ->
          debug 'hset', key, field, value, typeof value
          redis._[key] ?= {}
          Promise.resolve redis._[key][field] = value
        hdelAsync: (key,field) ->
          debug 'hdel', key, field
          redis._[key] ?= {}
          if field of redis._[key]
            delete redis._[key][field]
            Promise.resolve 1
          else
            Promise.resolve 0
        hgetAsync: (key,field) ->
          debug 'hget', key, field
          redis._[key] ?= {}
          Promise.resolve redis._[key][field]
        zaddAsync: (key,member,score) ->
          debug 'zadd', key, member,score
          redis._[key] ?= {}
          redis._[key][member] = score
          Promise.resolve 1
        zscanAsync: (key,cursor) ->
          debug 'sscan', key, cursor
          keys = []
          redis._[key] ?= {}
          keys = Object.getOwnPropertyNames redis._[key]
          keys = keys.sort (a,b) -> redis._[key][a] - redis._[key][b]
          Promise.resolve ["0",keys]
        zremAsync: (key,member) ->
          debug 'zrem', key, member
          redis._[key] ?= {}
          delete redis._[key][member]
          Promise.resolve 1
        zcardAsync: (key) ->
          debug 'scard', key
          redis._[key] ?= {}
          Promise.resolve Object.getOwnPropertyNames(redis._[key]).length
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
        sismemberAsync: (key,member) ->
          debug 'sismember', key, member
          Promise.resolve key of redis._
        sscanAsync: (key,cursor) ->
          debug 'sscan', key, cursor
          keys = []
          redis._[key] ?= new Set
          redis._[key].forEach (key) -> keys.push key
          Promise.resolve ["0",keys]
        smembersAsync: (key) ->
          debug 'smembers', key
          result = []
          for value of redis._[key]
            result.push value
          Promise.resolve result
        sismemberAsync: (key,member) ->
          debug 'sismember', key
          redis._[key] ?= new Set
          Promise.resolve redis._[key].has member
        scardAsync: (key) ->
          debug 'scard', key
          redis._[key] ?= new Set
          Promise.resolve redis._[key].size

      policy = (calls) ->
        debug 'policy_for…'
        return null if @key is 'lalala'
        calls[0]

      create_egress_call = ->
        debug 'egress_call_for', @key
        switch @key
          when 'lululu'
            Promise.resolve new TestCall destination: '33643482771'
          else
            Promise.resolve null

      api = (cmd) ->
        debug 'api', cmd
        if cmd.match /^myevents/
          return Promise.resolve
            once: ->
            on: ->
        true

      profile = 'booh!'

      class TestCall extends require '../call'
        redis: redis
        api: api
        profile: profile
        get_reference_data: (reference) -> params: {}

      class TestAgent extends require '../agent'
        redis: redis
        policy: policy
        create_egress_call: create_egress_call
        new_call: (data) -> new TestCall data

      Queuer = (require '../queuer') {redis, Agent: TestAgent, Call: TestCall}

      it 'should increment external calls', seem ->
        queuer = {}
        agent = new TestAgent queuer, 'lalala'
        yield agent.add_call 1234
        yield agent.del_call 1234
        redis._['agent-set-lalala'].should.have.property 'size', 0

      it 'should transition on login', seem ->
        queuer = new Queuer()
        agent = new TestAgent queuer, 'lalala'
        ok = yield agent.transition 'login'
        ok.should.be.true
        redis._['agent-property-lalala'].should.have.property 'state', 'idle'
        redis._['pool-zset-egress-agents'].should.have.property 'lalala', 0

      it 'should trigger call on idle', seem ->
        queuer = new Queuer()
        agent = new TestAgent queuer, 'lululu'
        ok = yield agent.transition 'login'
        ok.should.be.true
        redis._['agent-property-lululu'].should.have.property 'state', 'in_call'

      it 'should transition on ingress', seem ->
        queuer = new Queuer()
        lalilo = new TestAgent queuer, 'lalilo'
        laloli = new TestAgent queuer, 'laloli'
        ok = yield lalilo.transition 'login'
        ok.should.be.true
        ok = yield laloli.transition 'login'
        ok.should.be.true
        redis._['agent-property-lalilo'].should.have.property 'state', 'idle'
        redis._['agent-property-laloli'].should.have.property 'state', 'idle'
        yield queuer.queue_ingress_call new TestCall id:'1234'
        redis._['agent-property-lalilo'].should.have.property 'state', 'in_call'
        redis._['agent-property-laloli'].should.have.property 'state', 'idle'
