    {expect} = chai = require 'chai'
    chai.should()
    debug = (require 'tangible') 'black-metal:test:agent'

    Redis = require 'ioredis'
    RedisInterface = require 'normal-key/interface'

    EventEmitter = require 'events'

    sleep = (timeout) ->
      debug 'sleep', timeout
      new Promise (resolve) -> setTimeout resolve, timeout

    describe 'The Agent', ->
      redis = new Redis host:'redis'
      after -> setTimeout (-> redis.end()), 10000
      redis_interface = new RedisInterface redis

      created = 0
      create_egress_call = ->
        switch @key
          when 'lululu@test'
            debug "create_egress_call for #{@key}"
            return null if created++ > 0
            call = new TestCall 'testing123'
            await call.set_domain 'test'
            await call.set_reference 'hello-world'
            await call.add_tag 'skill:sales'
            setTimeout (-> call.transition 'dropped'), 2500
            call
          else
            debug "create_egress_call for #{@key} (ignored)"
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
        get_destination: -> Promise.resolve '33643482771'
        get_domain: -> Promise.resolve 'handy-bear'
        get_source: -> Promise.resolve 'famous-candy'
        set_endpoint: -> Promise.resolve yes

      {TaggedCall,TaggedAgent} = require '../tagged'
      class TestCall extends TaggedCall
        interface: redis_interface
        __api: api
        profile: profile
        Reference: Reference

      class TestAgent extends TaggedAgent
        interface: redis_interface
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
        debug 'Agent1 (lalala) logs in'
        agent1 = new TestAgent 'lalala@test'
        ok = await agent1.transition 'login'
        ok.should.be.true
        await sleep 50
        debug 'Agent2 (lululu) logs in'
        agent2 = new TestAgent 'lululu@test'
        agent2.add_tag 'skill:sales'
        ok = await agent2.transition 'login'
        ok.should.be.true
        await sleep 700
        debug 'Agent2 (lululu) should be receiving a call'
        (await redis.get 'agent-lululu@test-s').should.equal 'presenting'
        (await agent2.state()).should.equal 'presenting'
        (await redis.get 'agent-lalala@test-s').should.equal 'waiting'
        (await agent1.state()).should.equal 'waiting'
        await sleep 2000
        debug 'Agent2 (lululu) should be in-call'
        (await redis.get 'agent-lululu@test-s').should.equal 'in_call'
        (await agent2.state()).should.equal 'in_call'
        (await redis.get 'agent-lalala@test-s').should.equal 'waiting'
        (await agent1.state()).should.equal 'waiting'
        debug 'Agent1 (lalala) logs out'
        ok = await agent1.transition 'logout'
        debug 'Agent1 (lululu) logs out'
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
        state1 = await redis.get 'agent-lalilo@test-s'
        state2 = await redis.get 'agent-laloli@test-s'
        in_call = 0
        in_call += 1 if state1 is 'in_call'
        in_call += 1 if state2 is 'in_call'
        waiting = 0
        waiting += 1 if state1 is 'waiting'
        waiting += 1 if state2 is 'waiting'
        ok = await lalilo.transition 'logout'
        ok = await laloli.transition 'logout'
        queuer.end()
        chai.expect(in_call).to.equal 1
        chai.expect(waiting).to.equal 1
        return

      it 'should transition on events', ->
        @timeout 5000
        queuer = new Queuer()
        lalilo = new TestAgent 'lalilo@test'
        ok = await lalilo.transition 'login'
        ok.should.be.true
        await sleep 700
        (await lalilo.state()).should.equal 'waiting'

Create a new ingress call

        call = new TestCall 'test'
        await call.set_id '1234'
        await call.set_domain 'test'
        await call.set_reference 'hello-again'

Queue the call

        await queuer.queue_ingress_call call
        await sleep 2500

        agent_call = await lalilo.get_onhook_call()
        agent_call.should.have.property 'key'

Answer the call

        await call.on_bridge agent_call
        (await lalilo.state()).should.equal 'in_call'

Caller hangs up

        await call.on_unbridge agent_call, 'hangup'
        await sleep 250
        (await lalilo.state()).should.equal 'wrap_up'

        await call.on_hangup 'hangup'
        await sleep 250
        (await lalilo.state()).should.equal 'wrap_up'

Agent hangs up

        await agent_call.on_hangup 'hangup'
        await sleep 250
        (await lalilo.state()).should.equal 'waiting'

        queuer.end()
        return

      it 'should handle multiple calls', ->
        @timeout 15000

        ev = new EventEmitter
        private_api = (cmd) ->
          new Promise (resolve) ->
            ev.on 'agent_pickup', ->
              resolve '+'
        private_api.truthy = api.truthy
        class PrivateTestCall extends TestCall
          __api: private_api
        PrivateQueuer = (require '../queuer') redis_interface, Agent: TestAgent, Call: PrivateTestCall

        queuer = new PrivateQueuer()
        lalilo = new TestAgent 'lalilo@test'
        ok = await lalilo.transition 'login'
        ok.should.be.true
        await sleep 700
        (await lalilo.state()).should.equal 'waiting'

        debug 'Create two new ingress calls'

        call1 = new PrivateTestCall 'test1'
        await call1.set_id '1234'
        await call1.set_domain 'test'
        await call1.set_reference 'hello1'

        call2 = new PrivateTestCall 'test2'
        await call2.set_id '2345'
        await call2.set_domain 'test'
        await call2.set_reference 'hello2'

        debug 'Queue the calls'

        await queuer.queue_ingress_call call1
        await queuer.queue_ingress_call call2
        await sleep 2500

        (await lalilo.state()).should.equal 'presenting'

        debug 'Answer the first call'
        ev.emit 'agent_pickup'
        await sleep 500


        agent_call = await lalilo.get_onhook_call()
        agent_call.should.have.property 'key'

        call = await lalilo.get_remote_call()
        await call.on_bridge agent_call
        (await lalilo.state()).should.equal 'in_call'

        debug 'Caller hangs up'

        await call.on_unbridge agent_call, 'hangup'
        await sleep 500
        (await lalilo.state()).should.equal 'wrap_up'

        await call.on_hangup 'hangup'
        await sleep 500
        (await lalilo.state()).should.equal 'wrap_up'

        debug 'Agent hangs up, is presented new call'

        await agent_call.on_hangup 'hangup'
        await sleep 500
        (await lalilo.state()).should.equal 'presenting'
        await sleep 2000 # because the queuer will wait 1.5s before actually presenting

        debug 'Answer the second call'
        ev.emit 'agent_pickup'
        await sleep 500
        (await lalilo.state()).should.equal 'in_call'
        await sleep 2500
        (await lalilo.state()).should.equal 'in_call'

        debug 'Answer the second call'

        old_call = call
        call = await lalilo.get_remote_call()
        call.should.not.equal old_call
        await call.on_bridge agent_call
        (await lalilo.state()).should.equal 'in_call'

        agent_call = await lalilo.get_onhook_call()
        agent_call.should.have.property 'key'

        debug 'Caller hangs up: unbridge'

        await call.on_unbridge agent_call, 'hangup'
        await sleep 251
        (await lalilo.state()).should.equal 'wrap_up'

        debug 'Caller hangs up: hangup'
        await call.on_hangup 'hangup'
        await sleep 252
        (await lalilo.state()).should.equal 'wrap_up'

        debug 'Agent hangs up'

        await agent_call.on_hangup 'hangup'
        await sleep 253
        (await lalilo.state()).should.equal 'waiting'

        queuer.end()
        return


      it 'should handle broadcast calls', ->
        @timeout 40000

        ev = new EventEmitter
        private_api = (cmd) ->
          new Promise (resolve) ->
            if cmd.match /agent1@test/
              ev.on 'agent_pickup', -> resolve '+'
            if cmd.match /agent2@test/
              ev.on 'agent_canceled', -> resolve '-FAIL Canceled'

        private_api.truthy = (cmd) ->
          debug 'private_api.truthy', cmd
          if cmd.match /^uuid_exists/
            return 'true'
          if $ = cmd.match /^uuid_kill (\S+)$/
            await sleep 10
            the_call = new PrivateTestCall $[1]
            await the_call.on_hangup 'killed'
            await sleep 10
            ev.emit 'agent_canceled'

          true

        class PrivateTestCall extends TestCall
          __api: private_api
        PrivateQueuer = (require '../queuer') redis_interface, Agent: TestAgent, Call: PrivateTestCall

        queuer = new PrivateQueuer()

        debug 'Create two agents'
        agent1 = new TestAgent 'agent1@test'
        ok = await agent1.transition 'login'
        ok.should.be.true

        agent2 = new TestAgent 'agent2@test'
        ok = await agent2.transition 'login'
        ok.should.be.true

        await sleep 700
        (await agent1.state()).should.equal 'waiting'
        (await agent2.state()).should.equal 'waiting'

        debug 'Create a broadcast ingress calls'

        call = new PrivateTestCall 'broadcast-call'
        await call.set_id '9999'
        await call.set_domain 'test'
        await call.set_reference 'hello-broadcast'
        await call.add_tag 'broadcast'

        debug 'Queue the broadcast call'

        await queuer.queue_ingress_call call
        await sleep 2500

        debug 'agent1 should receive the call'
        (await agent1.state()).should.equal 'presenting'
        debug 'agent2 should receive the call'
        (await agent2.state()).should.equal 'presenting'

        debug 'Agent1 answers the call'
        ev.emit 'agent_pickup'
        await sleep 500

        debug 'Bridge the call with agent1'
        agent1_call = await agent1.get_onhook_call()
        await call.on_bridge agent1_call
        (await agent1.state()).should.equal 'in_call'
        await sleep 1000

        for second in [2..35]
          await sleep 1000
          debug "Verify state after #{second}s"
          (await agent1.state()).should.equal 'in_call'
          (await agent2.state()).should.be.oneOf ['away','waiting']

        queuer.end()
        return
