    @name = 'black-metal:queuer'
    seem = require 'seem'
    RedisClient = require 'normal-key/client'

    {debug,hand,heal} = (require 'tangible') @name

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

    nextTick = ->
      new Promise (resolve) ->
        process.nextTick resolve

Main Queuer
===========

    module.exports = (redis_interface,{Agent,Call}) ->

Call Pool
---------

      class CallPool extends RedisClient
        constructor: (queuer,domain,name) ->
          throw new Error 'CallPool requires queuer' unless queuer?.is_a_queuer?()
          throw new Error 'CallPool requires domain' unless domain?
          debug 'new CallPool', domain, name
          super 'CP', "#{domain}-#{name}"
          @queuer = queuer
          @domain = domain
          @name = name

        interface: redis_interface

        add: (call) ->
          debug 'CallPool.add', @key, call.key
          if super call.key
            heal call.transition 'pool'

        remove: (call) ->
          debug 'CallPool.remove', @key, call.key
          if super call.key
            heal call.transition 'unpool'

        has: (call) ->
          debug 'CallPool.has', @key, call.key
          super call.key

        calls: seem ->
          debug 'CallPool.calls', @key
          result = []
          queuer = @queuer
          domain = @domain
          yield @forEach seem (key) ->
            call = new Call queuer, domain, {key}
            yield call.load()
            result.push call
          debug 'CallPool.calls', @key, result.map (c) -> c.key
          result

Available agents
----------------

      class AgentPool extends RedisClient

        constructor: (queuer,domain,name) ->
          throw new Error 'AgentPool requires queuer' unless queuer?.is_a_queuer?()
          throw new Error 'AgentPool requires domain' unless domain?
          debug 'new AgentPool', domain, name
          super 'AP', "#{domain}-#{name}"
          @queuer = queuer
          @domain = domain
          @name = name

        interface: redis_interface

        add: seem (agent) ->
          debug 'AgentPool.add', @key, agent.key
          score = yield agent.get_missed().catch -> 0.0
          score += Math.random()/100
          @sorted_add agent.key, score

        remove: (agent) ->
          debug 'AgentPool.remove', @key, agent.key
          @sorted_remove agent.key

        reevaluate: seem (cb) ->
          debug 'AgentPool.reevaluate', @key
          queuer = @queuer
          yield @sorted_forEach seem (key) ->
            debug 'AgentPool.reevaluate', key
            agent = new Agent queuer, key
            yield cb agent
            return

          return

Call routing
------------

Calls might be ingress or egress.

The behavior is agent-driven; ingress calls wait in a pool, and the pool is depleted as agents become available.

Egress calls are created dynamically when no calls are available for a given agent.

For a given agent, how the ingress pool is filtered + sorted is based on domain, skills, call priorities, etc.

This means calls must be tagged in order to be assessed for a given user. Typical tags may include:
- domain (typically the agent's support domain set will contain a single domain, which is a good idea since currently domains must be colocated);
- queue/fifo/call-group (typically the agent's will log into at least one queue/fifo/call-group);
- service-level (e.g. some agents may handle VIP-level only calls);
- priority (for sorting);
- order of arrival (for sorting);
- etc.

For a given agent, their pool of ingress-calls-to-handle is therefor a subset of the overall pool of calls to handle.

      class Queuer

        Call: Call
        Agent: Agent

        is_a_queuer: -> true

        constructor: ->
          debug 'new Queuer'
          @__timers = {}

        ingress_pool: (domain) ->
          new CallPool this, domain, 'ingress'

        egress_pool: (domain) ->
          new CallPool this, domain, 'egress'

        available_agents: (domain) ->
          new AgentPool this, domain, 'available'

Monitor a call
--------------

The functions in this section are meant for call legs that are _not_ related to an agent actively managed by the queuer
(because those call-legs are managed by the monitoring code in the Agent module).

The following is meant to handle:
- external call legs (ingress or egress)

        monitor_remote_call: (remote_call) ->
          debug 'Queuer.monitor_remote_call', remote_call.key
          @monitor_call remote_call, (disposition) ->
            switch disposition
              when 'recv_replace', 'replaced', 'bridge'
                'transferred'
              else
                'hungup'

- internal (agent-bound) call legs outside the queuer.

        monitor_local_call: (agent_call) ->
          debug 'Queuer.monitor_local_call', agent_call.key
          @monitor_call agent_call, (disposition) ->
            'hangup'

In both cases the main goal is to keep track of the agent that might be connected to a call (especially during transfers) and update the state accordingly.

        monitor_call: seem (a_call,hangup_event) ->
          debug 'Queuer.monitor_call', a_call.key

          return if yield a_call.is_monitored()

          monitor = yield a_call.monitor 'CHANNEL_HANGUP_COMPLETE', 'CHANNEL_BRIDGE', 'CHANNEL_UNBRIDGE'

          self = this

### Monitored call is hung up.

          monitor.once 'CHANNEL_HANGUP_COMPLETE', hand ({body}) ->
            disposition = body?.variable_transfer_disposition

            heal monitor?.end()
            monitor = null

In case we get `UNBRIDGE` and `CHANNEL_COMPLETE` at the same time, let `UNBRIDGE` transition first.

            yield nextTick()

            yield a_call.load()

            a_agent_key = yield a_call.get_local_agent()
            b_agent_key = yield a_call.get_remote_agent()

            debug 'Queuer.monitor_call: CHANNEL_HANGUP_COMPLETE', a_call.key, a_agent_key, b_agent_key, disposition, body.variable_endpoint_disposition

If the call was transfered, clear the list of matching calls (used by `unbridge_except`).

            if disposition is 'replaced'
              yield a_call.clear()

The call leg might be bound to an agent.
(Although it should not be an `on-hook` or `off-hook` call-leg: these are monitored in the `Agent` module.)
Use `del_call` to notify the agent that its own call-leg got disconnected.
(Really there should be a separate Agent method for that.)

            if a_agent_key?
              a_agent = new Agent self, a_agent_key
              yield a_agent.del_call a_call.id, disposition

The call leg might be connected to an agent.

            if b_agent_key?
              b_agent = new Agent self, b_agent_key
              yield b_agent.del_call a_call.id, disposition

            yield a_call.set_remote_agent null

            yield a_call.transition hangup_event disposition

            return

          monitor.on 'CHANNEL_BRIDGE', hand ({body}) ->
            a_uuid = body['Bridge-A-Unique-ID']
            b_uuid = body['Bridge-B-Unique-ID']
            disposition = body?.variable_transfer_disposition

            yield a_call.load()

            b_call = new Call self, a_call.domain, id:b_uuid

            yield a_call.load()
            a_agent_key = yield a_call.get_local_agent()

            yield b_call.load()
            b_agent_key = yield b_call.get_local_agent()

            debug 'Queuer.monitor_call: CHANNEL_BRIDGE', a_uuid, a_agent_key, b_uuid, b_agent_key, disposition, body.variable_endpoint_disposition

Let each call-leg know which agent it is connected to. (This includes _not_ being connected to an agent.)

            yield a_call.set_remote_agent b_agent_key
            yield b_call.set_remote_agent a_agent_key

            yield a_call.on_bridge b_call, if b_agent_key? then agent_call:b_call else call:b_call
            yield b_call.on_bridge a_call, if a_agent_key? then agent_call:a_call else call:a_call
            return

          monitor.on 'CHANNEL_UNBRIDGE', hand ({body}) ->
            a_uuid = body['Bridge-A-Unique-ID']
            b_uuid = body['Bridge-B-Unique-ID']
            disposition = body?.variable_transfer_disposition

            yield a_call.load()

            b_call = new Call self, a_call.domain, id:b_uuid

            yield a_call.load()
            a_agent_key = yield a_call.get_local_agent()

            yield b_call.load()
            b_agent_key = yield b_call.get_local_agent()

            debug 'Queuer.monitor_call: CHANNEL_UNBRIDGE', a_uuid, a_agent_key, b_uuid, b_agent_key, disposition, body.variable_endpoint_disposition

If the call was transfered, clear the list of matching calls (used by `unbridge_except`).

            if disposition is 'replaced'
              yield a_call.clear()

Remove the other call leg from each agent's list.

            if a_agent_key?
              a_agent = new Agent self, a_agent_key
              yield a_agent.del_call b_call.id, disposition

            if b_agent_key?
              b_agent = new Agent self, b_agent_key
              yield b_agent.del_call a_call.id, disposition

And remove the link to the remote agent as well.

            ###
            # This might run into issues.
            # We really need an atomic "set to this new value if the value was" operation.
            ###
            yield a_call.set_remote_agent null if b_agent_key is yield a_call.get_remote_agent()
            yield b_call.set_remote_agent null if a_agent_key is yield b_call.get_remote_agent()

            yield a_call.on_unbridge b_call
            yield b_call.on_unbridge a_call
            return

          return

Evaluate Agent
--------------

When an agent moves to the `idle` state, the queuer picks one call out of the pool:
- either an ingress calls (they are always prioritized over egress calls)
- if no ingress calls require attention, an egress call is created and assigned to the agent.

The method `evaluate_agent` will return `true` if an `idle` agent is still available after trying to present them with some calls.
Otherwise it returns `false`, indicating the agent was not, is not, or is no longer available.

        __evaluate_agent: seem (agent) ->
          debug 'Queuer.__evaluate_agent', agent.key

### Precondition

We force a transition:
- to ensure that the current state was one that is valid for us to evaluate from;
- to ensure that no other operation will proceed along the same path at the same time.

          if not yield agent.transition 'evaluate'
            debug 'Queuer.__evaluate_agent: precondition failed', agent.key

Return falsey: `evaluate` can only be successful if the previous state was `idle` or `waiting`, so the state was not `idle` nor `waiting`, meaning (since the state was not transitioned) that the agent was not and is not available.

            return false

Give `on_agent` a chance to transition the agent out of the available pool.

          yield nextTick()

### Build Call

Return:
- a Call (towards or from a selected third-party),
- or `null`.

          build_call = seem (pool) ->

            debug 'Queuer.__evaluate_agent build_call', agent.key

The first step is for the agent to find a suitable call in the pool.

            calls = yield pool.calls()
            call = yield agent.policy calls

If no call was found the agent's state is unmodified.

            if not call?
              debug 'Queuer.__evaluate_agent build_call: no call found', agent.key
              return null

            event = if call.broadcasting then 'broadcast' else 'handle'
            unless yield call.transition event
              debug 'Queuer.__evaluate_agent build_call: call did not transition to handled', agent.key, call.key
              return null

Transition the agent.

            debug 'Queuer.__evaluate_agent build_call: transition the agent', agent.key, call.key

            if not yield agent.transition 'present', {call}
              debug 'Queuer.__evaluate_agent build_call: transition failed', agent.key, call.key
              return null

            debug 'Queuer.__evaluate_agent build_call: transitioned', agent.key, call.key

Wait a little bit (this is meant to give a popup some time to settle).

            debug 'Queuer.__evaluate_agent build_call: waiting for 1.5s before originate_external', agent.key, call.key
            yield sleep 1500-100+200*Math.random()
            yield call.load()

For a dial-out (egress) call we first need to attempt to contact the destination.
For a dial-in (ingress) call we already have the proper call UUID.

            yield call.originate_external()

            debug 'Queuer.__evaluate_agent build_call: originate external returned', agent.key, call.key, call.id

            call_state = yield call.state()
            unless call_state is 'handled'
              debug 'Queuer.__evaluate_agent build_call: call state is not `handled`', agent.key, call.key, call.id, call_state
              yield agent.transition 'hangup', {call}
              return null

Notify the agent of the caller's state.

            yield agent.set_remote_call call
            if (yield @monitor_remote_call(call).catch -> true)

Hangup if we couldn't start monitoring.

              yield agent.transition 'hangup', {call}
              return null

            return call

### Send to Agent

We need to send the call to the agent (using either onhook or offhook mode).

          send_to_agent = seem (pool,call) ->

            debug 'Queuer.__evaluate_agent send_to_agent: originate', agent.key, call.key

            {reason} = agent_call = yield agent.originate call

            if reason
              yield agent.set_remote_call null
              yield agent.incr_missed()
              yield agent.transition 'missed', {call,reason}
              heal call.transition 'retry'
              return false

            debug 'Queuer.__evaluate_agent send_to_agent: bridge', agent.key, call.key

            yield agent_call.set_local_agent agent.key
            yield agent_call.transition 'handle' # FIXME rename event, e.g. 'send-to-agent'
            unless yield call.bridge agent_call
              yield heal call.remove agent_call.id # undo what was done in `call.originate_internal`
              yield agent.set_remote_call null
              yield agent_call.hangup()
              yield agent.transition 'failed', {call}
              return false

            debug 'Queuer.__evaluate_agent send_to_agent: Successfully bridged', agent.key, call.key, call.id, agent_call.key
            yield call.set_remote_agent agent.key
            yield agent.transition 'answer', {call}

### Main body for `__evaluate_agent`

          some_call = yield agent.get_remote_call()
          if some_call?
            debug.dev 'Error: Agent was idle/waiting but still had a remote call', agent.key, some_call.key
          yield agent.set_remote_call null

Ingress pool

          debug 'Queuer.__evaluate_agent: ingress pool', agent.key
          ingress_pool = @ingress_pool agent.domain

          remote_call = yield build_call.call(this,ingress_pool).catch (error) ->
            debug.ops 'Queuer.__evaluate_agent: ingress pool, error in build_call', error.stack ? error.toString()
            null

          if remote_call?

            debug 'Queuer.__evaluate_agent: ingress pool, got client call', agent.key

            answered = yield send_to_agent.call(this, ingress_pool, remote_call).catch (error) ->
              debug.ops 'Queuer.__evaluate_agent: ingress pool, error in send_to_agent', error.stack ? error.toString()
              null

            debug "Queuer.__evaluate_agent: ingress pool, agent answered=#{answered}", agent.key
            return true if answered is null
            return false if answered

          else
            debug "Queuer.__evaluate_agent: ingress pool, no matching client call", agent.key

Egress pool

          debug 'Queuer.__evaluate_agent: egress pool', agent.key
          egress_pool = @egress_pool agent.domain

          remote_call = yield build_call.call(this,egress_pool).catch (error) ->
            debug.ops 'Queuer.__evaluate_agent: egress pool, error in build_call', error.stack ? error.toString()
            null

          if remote_call?

            debug 'Queuer.__evaluate_agent: egress pool, got client call', agent.key

We forcibly remove the call so that we do not end up ringing the same prospect/customer twice, esp. if in the first case there was no agent available.

            yield egress_pool.remove remote_call

            answered = yield send_to_agent.call(this, egress_pool, remote_call).catch (error) ->
              debug.ops 'Queuer.__evaluate_agent: egress pool, error in send_to_agent', error.stack ? error.toString()
              null

            debug "Queuer.__evaluate_agent: egress pool, agent answered=#{answered}", agent.key
            return true if answered is null
            return false if answered

          else
            debug "Queuer.__evaluate_agent: egress pool, no matching client call", agent.key

No call

          debug 'Queuer.__evaluate_agent: no call', agent.key
          if yield agent.transition 'release'
            yield agent.park()

        queue_ingress_call: seem (call) ->
          debug 'Queuer.queue_ingress_call', call.key
          yield call.set_poolable()
          yield @monitor_remote_call call
          yield @ingress_pool(call.domain).add call

        __transition_available_agents: seem (event,domain) ->
          debug 'Queuer.__transition_available_agents: start', {domain,event}
          yield nextTick()
          @available_agents(domain).reevaluate hand (agent) ->
            yield nextTick()
            debug 'Queuer.__transition_available_agents for agent', {agent: agent.key, event}
            agent.transition event

Data is optional but is used by huge-play's `create-queuer-call`.

        create_egress_call_for: seem (agent,data) ->
          debug 'Queuer.create_egress_call_for', agent.key, data

The call instance is created using data found e.g. in a database, the (egress) call is placed in the pool, and the idle agents are re-evaluated.

          call = yield agent.create_egress_call data
          if call?
            debug 'Queuer.create_egress_call_for: queue egress call', agent.key, call.key
            yield call.set_poolable()
            yield agent.transition 'created'
            yield @egress_pool(call.domain).add call
          else
            debug 'Queuer.create_egress_call_for: no call', agent.key
            yield agent.transition 'not_created'

        on_agent: seem (agent,data) ->
          {state} = data
          debug 'Queuer.on_agent', agent.key, state

Only states were the agent might transition via `evaluate` are considered as states were the agents is available.

          switch state
            when 'idle', 'waiting'
              yield @available_agents(agent.domain).add agent
            else
              yield @available_agents(agent.domain).remove agent

          switch state
            when 'logged_out'
              yield agent.reset_missed()
              yield agent.clear()

            when 'in_call'
              yield agent.reset_missed()

            when 'wrap_up'
              yield agent.wrapup()

If the agent is idle, move forward in the background.

            when 'idle'
              yield @__evaluate_agent agent

            when 'create_call'
              yield @create_egress_call_for agent

          return

        on_call: seem (call,data) ->
          {state} = data
          debug 'Queuer.on_call', call.key, state

          ingress_pool = @ingress_pool call.domain
          egress_pool = @egress_pool call.domain

          switch state

            when 'new' # aka `forgotten`
              if yield call.poolable()
                yield heal ingress_pool.add call
              else
                debug.dev 'Ignoring non-poolable call', call.key

            when 'pooled'
              heal @__transition_available_agents 'new_call', call.domain

            when 'bridged'
              yield ingress_pool.remove call
              yield egress_pool.remove call

              yield call.set_answered()

              if data.agent_call?

Hang up all other (ringing) agents.

                yield call.unbridge_except data.agent_call.key

Do not automatically close the agent's call (in `dropped`) when a remote party hangs up.

              yield call.clear()

            when 'dropped'
              yield ingress_pool.remove call
              yield egress_pool.remove call
              yield call.unbridge_except()

          return

        clear_timer: (key) ->
          if @__timers[key]?
            debug 'Queuer.clear_timer', key
            clearTimeout @__timers[key]
            delete @__timers[key]

        set_timer: (key,timer) ->
          @clear_timer key
          @__timers[key] = timer

        end: ->
          for own key of @__timers
            @clear_timer key
          return

Switch agent
------------

This is used by `huge-play` in order to track calls connected to an agent (especially outside the queuer).
The `call` is the agent-side call (not a remote-call).

        set_agent: seem (call,new_key) ->
          debug 'Queuer.set_agent', call?.key, new_key

          return unless call? and new_key?

          yield call.transition 'handle' # FIXME rename event, e.g. `set-agent`

          old_key = yield call.get_remote_agent()
          return if old_key is new_key
          yield call.set_local_agent new_key

          if old_key?
            old_agent = new Agent this, old_key
            yield old_agent.del_call call.id

          new_agent = new Agent this, new_key
          yield new_agent.add_call call.id

          return

Agent behavior
--------------

Internal-lines, two ways to reach:
- mode A (off-hook agents): agents call into a number, are parked until a call is presented.
- mode B (on-hook agents): we ring the phone of the agent.

The state machine handles both modes.

Egress calls only really make sense in mode A.

      Queuer
