Precondition: `docker run -p 127.0.0.1:6379:6379 redis` (for example).

    chai = require 'chai'
    chai.should()
    seem = require 'seem'
    debug = (require 'tangible') 'black-metal:test:agent'

    Redis = require 'ioredis'
    RedisInterface = require 'normal-key/interface'

    sleep = (timeout) ->
      new Promise (resolve) -> setTimeout resolve, timeout

    describe 'The Agent', ->
      redis = new Redis()
      redis_interface = new RedisInterface [redis]

      policy = (calls) ->
        debug 'policy_for…'
        return null if @key is 'lalala'
        calls[0]

      create_egress_call = seem ->
        debug 'create_egress_call for', @key
        switch @key
          when 'lululu'
            call = new TestCall @queuer, destination: '33643482771'
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
            event_json: ->
            end: ->
        if cmd.match /^uuid_exists/
          return Promise.resolve 'true'
        Promise.resolve true

      profile = 'booh!'
      class Reference
        get_destination: -> Promise.resolve 'hello'
        get_domain: -> Promise.resolve 'handy-bear'
        get_source: -> Promise.resolve 'famous-candy'
        set_endpoint: -> Promise.resolve yes
        add_in: -> Promise.resolve yes

      class TestCall extends require '../call'
        redis: redis_interface
        api: api
        profile: profile
        Reference: Reference

      class TestAgent extends require '../agent'
        redis: redis_interface
        policy: policy
        create_egress_call: create_egress_call
        new_call: (data) -> new TestCall @queuer, data

      Queuer = (require '../queuer') redis: redis_interface, Agent: TestAgent, Call: TestCall

      cleanup = seem ->
        yield redis.del 'agent-lalala-S'
        yield redis.del 'agent-lalala-P'
        yield redis.del 'agent-lalala-Z'
        yield redis.del 'agent-lululu-S'
        yield redis.del 'agent-lululu-P'
        yield redis.del 'agent-lululu-Z'
        yield redis.del 'agent-lalilo-S'
        yield redis.del 'agent-lalilo-P'
        yield redis.del 'agent-lalilo-Z'
        yield redis.del 'agent-laloli-S'
        yield redis.del 'agent-laloli-P'
        yield redis.del 'agent-laloli-Z'
        yield redis.del 'AP-available-Z'
        yield redis.del 'CP-ingress-S'
        yield redis.del 'CP-egress-S'
        yield redis.del 'call-1234-P'

      before cleanup
      after cleanup

      it 'should increment external calls', seem ->
        queuer = new Queuer()
        agent = new TestAgent queuer, 'lalala'
        yield agent.add_call 1234
        (yield redis.scard 'agent-lalala-S').should.equal 1
        yield agent.del_call 1234
        (yield redis.scard 'agent-lalala-S').should.equal 0

      it 'should transition on login', seem ->
        queuer = new Queuer()
        agent = new TestAgent queuer, 'lalala'
        ok = yield agent.transition 'login'
        ok.should.be.true
        yield sleep 700
        (yield agent.get_missed()).should.be.a 'number'
        (yield redis.hget 'agent-lalala-P', 'state').should.equal 'waiting'
        (yield redis.zrank 'AP-available-Z', 'lalala').should.be.a 'number'
        (yield redis.zrank 'AP-available-Z', 'lalala').should.be.within 0, 1
        ok = yield agent.transition 'logout'
        ok.should.be.true

      it 'should trigger call on idle', seem ->
        @timeout 4000
        queuer = new Queuer()
        agent = new TestAgent queuer, 'lalala'
        ok = yield agent.transition 'login'
        ok.should.be.true
        agent = new TestAgent queuer, 'lululu'
        ok = yield agent.transition 'login'
        ok.should.be.true
        yield sleep 700
        (yield redis.hget 'agent-lululu-P', 'state').should.equal 'presenting'
        (yield redis.hget 'agent-lalala-P', 'state').should.equal 'waiting'
        yield sleep 1800
        (yield redis.hget 'agent-lululu-P', 'state').should.equal 'in_call'
        (yield redis.hget 'agent-lalala-P', 'state').should.equal 'waiting'

      it 'should transition on ingress', seem ->
        @timeout 4000
        queuer = new Queuer()
        lalilo = new TestAgent queuer, 'lalilo'
        laloli = new TestAgent queuer, 'laloli'
        ok = yield lalilo.transition 'login'
        ok.should.be.true
        ok = yield laloli.transition 'login'
        ok.should.be.true
        yield sleep 700
        (yield redis.hget 'agent-lalilo-P', 'state').should.equal 'waiting'
        (yield redis.hget 'agent-laloli-P', 'state').should.equal 'waiting'
        call = new TestCall queuer, id:'1234'
        yield call.save()
        yield call.set_reference 'hello-again'
        yield queuer.queue_ingress_call call
        yield sleep 2000
        in_call = 0
        in_call += 1 if (yield redis.hget 'agent-lalilo-P', 'state') is 'in_call'
        in_call += 1 if (yield redis.hget 'agent-laloli-P', 'state') is 'in_call'
        chai.expect(in_call).to.equal 1
        (yield redis.hget 'agent-laloli-P', 'state').should.equal 'waiting'
