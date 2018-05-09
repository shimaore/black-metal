Precondition: `docker run -p 127.0.0.1:6379:6379 redis` (for example).

    chai = require 'chai'
    chai.should()
    debug = (require 'tangible') 'black-metal:test:agent'

    Redis = require 'ioredis'
    RedisInterface = require 'normal-key/interface'

    sleep = (timeout) ->
      new Promise (resolve) -> setTimeout resolve, timeout

    describe 'The Agent', ->
      redis = new Redis()
      after -> setTimeout (-> redis.end()), 10000
      redis_interface = new RedisInterface redis

      policy = (calls) ->
        debug 'policy_for…'
        return null if @key is 'lalala@test'
        calls[0]

      created = 0
      create_egress_call = ->
        debug 'create_egress_call for', @key
        switch @key
          when 'lululu@test'
            return null if created++ > 0
            call = new TestCall 'testing123'
            await call.set_domain 'test'
            await call.set_destination '33643482771'
            await call.set_reference 'hello-world'
            setTimeout (-> call.transition 'dropped'), 2500
            call
          else
            null

      api = ->
        Promise.resolve '+'

      api.truthy = (cmd) ->
        debug 'api.truthy', cmd
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

      Call = require '../call'
      class TestCall extends Call
        interface: redis_interface
        __api: api
        profile: profile
        Reference: Reference

      Agent = require '../agent'
      class TestAgent extends Agent
        interface: redis_interface
        policy: policy
        create_egress_call: create_egress_call

      Queuer = (require '../queuer') redis_interface, Agent: TestAgent, Call: TestCall

      cleanup = ->
        await redis.flushdb()

      beforeEach cleanup

      it 'should increment external calls', ->
        queuer = new Queuer()
        agent = new TestAgent 'lalala@test'
        await agent.on_bridge new TestCall 1234
        (await redis.scard 'agent-lalala@test-S').should.equal 1
        await agent.on_unbridge new TestCall 1234
        (await redis.scard 'agent-lalala@test-S').should.equal 0
        queuer.end()

      it 'should transition on login', ->
        queuer = new Queuer()
        agent = new TestAgent 'lalala@test'
        ok = await agent.transition 'login'
        ok.should.be.true
        await sleep 700
        (await agent.get_missed()).should.be.a 'number'
        (await redis.get 'agent-lalala@test-s').should.equal 'waiting'
        (await agent.state()).should.equal 'waiting'
        (await redis.zrank 'AP-test-available-Z', 'lalala@test').should.be.a 'number'
        (await redis.zrank 'AP-test-available-Z', 'lalala@test').should.be.within 0, 1
        ok = await agent.transition 'logout'
        ok.should.be.true
        queuer.end()
        return

      it 'should trigger call on idle', ->
        @timeout 4000
        queuer = new Queuer()
        agent1 = new TestAgent 'lalala@test'
        ok = await agent1.transition 'login'
        ok.should.be.true
        await sleep 50
        agent2 = new TestAgent 'lululu@test'
        ok = await agent2.transition 'login'
        ok.should.be.true
        await sleep 700
        (await redis.get 'agent-lululu@test-s').should.equal 'presenting'
        (await agent2.state()).should.equal 'presenting'
        (await redis.get 'agent-lalala@test-s').should.equal 'waiting'
        (await agent1.state()).should.equal 'waiting'
        await sleep 1800
        (await redis.get 'agent-lululu@test-s').should.equal 'in_call'
        (await agent2.state()).should.equal 'in_call'
        (await redis.get 'agent-lalala@test-s').should.equal 'waiting'
        (await agent1.state()).should.equal 'waiting'
        ok = await agent1.transition 'logout'
        ok = await agent2.transition 'logout'
        queuer.end()
        return

      it 'should transition on ingress', ->
        @timeout 5000
        queuer = new Queuer()
        lalilo = new TestAgent 'lalilo@test'
        laloli = new TestAgent 'laloli@test'
        ok = await lalilo.transition 'login'
        ok.should.be.true
        ok = await laloli.transition 'login'
        ok.should.be.true
        await sleep 700
        (await redis.get 'agent-lalilo@test-s').should.equal 'waiting'
        (await lalilo.state()).should.equal 'waiting'
        (await redis.get 'agent-laloli@test-s').should.equal 'waiting'
        (await laloli.state()).should.equal 'waiting'
        call = new TestCall 'test'
        await call.set_id '1234'
        await call.set_domain 'test'
        await call.set_reference 'hello-again'
        await queuer.queue_ingress_call call
        await sleep 2500
        in_call = 0
        in_call += 1 if (await redis.get 'agent-lalilo@test-s') is 'in_call'
        in_call += 1 if (await redis.get 'agent-laloli@test-s') is 'in_call'
        chai.expect(in_call).to.equal 1
        waiting = 0
        waiting += 1 if (await redis.get 'agent-lalilo@test-s') is 'waiting'
        waiting += 1 if (await redis.get 'agent-laloli@test-s') is 'waiting'
        chai.expect(waiting).to.equal 1
        ok = await lalilo.transition 'logout'
        ok = await laloli.transition 'logout'
        queuer.end()
        return
