    chai = require 'chai'
    chai.should()
    seem = require 'seem'
    debug = (require 'tangible') 'black-metal:test:agent'

    RedisInterface = require 'normal-key/interface'

    describe 'The Agent', ->
      redis =
        _: {}
        expireAsync: ->
          Promise.resolve null
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
        hincrbyAsync: (key,field,inc) ->
          debug 'hincrby', key, field, inc
          redis._[key] ?= {}
          redis._[key][field] ?= 0
          Promise.resolve redis._[key][field] += inc
        zaddAsync: (key,score,member) ->
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
          result = []
          for member in keys
            result.push member
            result.push redis._[key][member]
          Promise.resolve ["0",result]
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
        sinterstoreAsync: (key) ->
          debug 'scard', key
          redis._[key] = new Set
          Promise.resolve 1

      redis_interface = new RedisInterface [redis]

      policy = (calls) ->
        debug 'policy_for…'
        return null if @key is 'lalala'
        calls[0]

      create_egress_call = seem ->
        debug 'egress_call_for', @key
        switch @key
          when 'lululu'
            call = new TestCall destination: '33643482771'
            yield call.save()
            yield call.set_reference 'hello-world'
            call
          else
            null

      api = (cmd) ->
        debug 'api', cmd
        if cmd.match /^myevents/
          return Promise.resolve
            once: ->
            on: ->
        if cmd.match /^uuid_exists/
          return Promise.resolve 'true'
        Promise.resolve true

      profile = 'booh!'

      class TestCall extends require '../call'
        redis: redis_interface
        api: api
        profile: profile
        get_reference_data: (reference) -> params: {}
        update_reference_data: (reference_data) ->
        update_call_data: (call_data) ->

      class TestAgent extends require '../agent'
        redis: redis_interface
        policy: policy
        create_egress_call: create_egress_call
        new_call: (data) -> new TestCall data

      Queuer = (require '../queuer') redis: redis_interface, Agent: TestAgent, Call: TestCall

      it 'should increment external calls', seem ->
        queuer = new Queuer()
        agent = new TestAgent queuer, 'lalala'
        yield agent.add_call 1234
        yield agent.del_call 1234
        redis._['agent-lalala-S'].should.have.property 'size', 0

      it 'should transition on login', seem ->
        queuer = new Queuer()
        agent = new TestAgent queuer, 'lalala'
        ok = yield agent.transition 'login'
        ok.should.be.true
        redis._['agent-lalala-P'].should.have.property 'state', 'idle'
        redis._['pool-egress-agents-Z'].should.have
          .property 'lalala'
          .within 0, 1

      it 'should trigger call on idle', seem ->
        @timeout 4000
        queuer = new Queuer()
        agent = new TestAgent queuer, 'lululu'
        ok = yield agent.transition 'login'
        ok.should.be.true
        redis._['agent-lululu-P'].should.have.property 'state', 'presenting'

      it 'should transition on ingress', seem ->
        @timeout 4000
        queuer = new Queuer()
        lalilo = new TestAgent queuer, 'lalilo'
        laloli = new TestAgent queuer, 'laloli'
        ok = yield lalilo.transition 'login'
        ok.should.be.true
        ok = yield laloli.transition 'login'
        ok.should.be.true
        redis._['agent-lalilo-P'].should.have.property 'state', 'idle'
        redis._['agent-laloli-P'].should.have.property 'state', 'idle'
        call = new TestCall id:'1234'
        yield call.save()
        yield call.set_reference 'hello-again'
        yield queuer.queue_ingress_call
        in_call = 0
        in_call += 1 if redis._['agent-lalilo-P'].state is 'in_call'
        in_call += 1 if redis._['agent-laloli-P'].state is 'idle'
        chai.expect(in_call).to.equal 1
