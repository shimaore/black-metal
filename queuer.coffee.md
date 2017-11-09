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
        constructor: (@queuer,@name) ->
          throw new Error 'CallPool requires queuer' unless @queuer?
          debug 'new CallPool', @name
          super 'CP', @name

        interface: redis_interface

        add: (call) ->
          debug 'CallPool.add', @name, call.key
          if super call.key
            heal call.transition 'pool'

        remove: (call) ->
          debug 'CallPool.remove', @name, call.key
          if super call.key
            heal call.transition 'unpool'

        has: (call) ->
          debug 'CallPool.has', @name, call.key
          super call.key

        calls: seem ->
          debug 'CallPool.all', @key
          result = []
          yield @forEach seem (key) =>
            call = new Call @queuer, {key}
            yield call.load()
            result.push call
          debug 'CallPool.all', @key, result.map (c) -> c.key
          result

Available agents
----------------

      class AgentPool extends RedisClient

        constructor: (@queuer,@name) ->
          throw new Error 'AgentPool requires queuer' unless @queuer?
          debug 'new AgentPool'
          super 'AP', @name

        interface: redis_interface

        add: seem (agent) ->
          debug 'AgentPool.add', agent.key
          score = yield agent.get_missed().catch -> 0.0
          score += Math.random()/100
          @sorted_add agent.key, score

        remove: (agent) ->
          debug 'AgentPool.remove', agent.key
          @sorted_remove agent.key

        reevaluate: seem (cb) ->
          debug 'AgentPool.reevaluate'
          yield @sorted_forEach seem (key) =>
            debug 'AgentPool.reevaluate', key
            agent = new Agent @queuer, key
            yield cb agent

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

        constructor: ->
          debug 'new Queuer'
          @ingress_pool = new CallPool this, 'ingress'
          @egress_pool = new CallPool this, 'egress'
          @available_agents = new AgentPool this, 'available'
          @__timers = {}

Monitor a call
--------------

        monitor_remote_call: (remote_call) ->
          debug 'Queuer.monitor_remote_call', remote_call.key
          @monitor_call remote_call, (disposition) ->
            switch disposition
              when 'recv_replace', 'replaced', 'bridge'
                'transferred'
              else
                'hungup'

        monitor_local_call: (agent_call) ->
          debug 'Queuer.monitor_local_call', agent_call.key
          @monitor_call agent_call, (disposition) ->
            'hangup'

        monitor_call: seem (call,hangup_event) ->
          debug 'Queuer.monitor_call', call.key

          return if yield call.is_monitored()

          monitor = yield call.monitor 'CHANNEL_HANGUP_COMPLETE', 'CHANNEL_BRIDGE', 'CHANNEL_UNBRIDGE'

          monitor.once 'CHANNEL_HANGUP_COMPLETE', hand ({body}) =>
            disposition = body?.variable_transfer_disposition
            debug 'Queuer.monitor_call: CHANNEL_HANGUP_COMPLETE', call.key, disposition, body.variable_endpoint_disposition

            heal monitor?.end()
            monitor = null

            yield call.load()
            if disposition is 'replaced'
              yield call.clear()

            agent_key = yield call.get_agent()
            if agent_key?
              agent = new Agent this, agent_key
              yield agent.del_call call.id, disposition

            yield call.transition hangup_event disposition

            return

          monitor.on 'CHANNEL_BRIDGE', hand ({body}) =>
            a_uuid = body['Bridge-A-Unique-ID']
            b_uuid = body['Bridge-B-Unique-ID']
            disposition = body?.variable_transfer_disposition
            debug 'Queuer.monitor_call: CHANNEL_BRIDGE', a_uuid, b_uuid, disposition, body.variable_endpoint_disposition

            yield call.load()

            agent_key = yield call.get_agent()

            their_call = new Call this, id:b_uuid
            yield their_call.load()

            yield call.transition 'bridge', call:their_call
            yield their_call.transition 'bridge', if agent_key? then agent_call:call else {call}
            return

          monitor.on 'CHANNEL_UNBRIDGE', hand ({body}) =>
            a_uuid = body['Bridge-A-Unique-ID']
            b_uuid = body['Bridge-B-Unique-ID']
            disposition = body?.variable_transfer_disposition
            debug 'Queuer.monitor_call: CHANNEL_UNBRIDGE', a_uuid, b_uuid, disposition, body.variable_endpoint_disposition

            yield call.load()
            if disposition is 'replaced'
              yield call.clear()

            agent_key = yield call.get_agent()
            if agent_key?
              agent = new Agent this, agent_key
              yield agent.del_call call.id, disposition

            their_call = new Call this, id:b_uuid
            yield their_call.load()

            yield call.transition 'unbridge', call:their_call
            yield their_call.transition 'unbridge', {call}
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

          build_call = seem (pool) =>

            debug 'Queuer.__evaluate_agent build_call', agent.key

The first step is for the agent to find a suitable call in the pool.

            calls = yield pool.calls()
            call = yield agent.policy calls

If no call was found the agent's state is unmodified.

            if not call?
              debug 'Queuer.__evaluate_agent build_call: no call found', agent.key
              return null

            unless yield call.lock()
              debug 'Queuer.__evaluate_agent build_call: call did not lock', agent.key, call.key
              return null

            event = if call.broadcasting then 'broadcast' else 'handle'
            unless yield call.transition event
              debug 'Queuer.__evaluate_agent build_call: call did not transition to handled', agent.key, call.key
              return null

Transition the agent.

            debug 'Queuer.__evaluate_agent build_call: transition the agent', agent.key, call.key

            unless yield agent.lock()
              debug 'Queuer.__evaluate_agent build_call: agent did not lock', agent.key, call.key
              return null

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

          send_to_agent = seem (pool,call) =>

            debug 'Queuer.__evaluate_agent send_to_agent: originate', agent.key, call.key

            agent_call = yield agent.originate call

            unless agent_call?
              yield agent.set_remote_call null
              yield agent.incr_missed()
              yield agent.transition 'missed', {call}
              heal call.transition 'pool'
              return false

            debug 'Queuer.__evaluate_agent send_to_agent: bridge', agent.key, call.key

            yield agent_call.set_agent agent.key
            yield agent_call.transition 'handle'
            unless yield call.bridge agent_call
              yield heal call.remove agent_call.id # undo what was done in `call.originate_internal`
              yield agent.set_remote_call null
              yield agent_call.hangup()
              yield agent.transition 'failed', {call}
              return false

            debug 'Queuer.__evaluate_agent send_to_agent: Successfully bridged', agent.key, call.key, call.id, agent_call.key
            yield call.set_agent agent.key
            yield agent.transition 'answer', {call}

### Main body for `__evaluate_agent`

          some_call = yield agent.get_remote_call()
          if some_call?
            debug.dev 'Error: Agent was idle/waiting but still had a remote call', agent.key, some_call.key
          yield agent.set_remote_call null

Ingress pool

          debug 'Queuer.__evaluate_agent: ingress pool', agent.key

          remote_call = yield build_call(@ingress_pool).catch (error) ->
            debug.ops 'Queuer.__evaluate_agent: ingress pool, error in build_call', error.stack ? error.toString()
            null

          if remote_call?

            debug 'Queuer.__evaluate_agent: ingress pool, got client call', agent.key

            answered = yield send_to_agent(@ingress_pool, remote_call).catch (error) ->
              debug.ops 'Queuer.__evaluate_agent: ingress pool, error in send_to_agent', error.stack ? error.toString()
              null

            debug "Queuer.__evaluate_agent: ingress pool, agent answered=#{answered}", agent.key
            return true if answered is null
            return false if answered

          else
            debug "Queuer.__evaluate_agent: ingress pool, no matching client call", agent.key

Egress pool

          debug 'Queuer.__evaluate_agent: egress pool', agent.key

          remote_call = yield build_call(@egress_pool).catch (error) ->
            debug.ops 'Queuer.__evaluate_agent: egress pool, error in build_call', error.stack ? error.toString()
            null

          if remote_call?

            debug 'Queuer.__evaluate_agent: egress pool, got client call', agent.key

We forcibly remove the call so that we do not end up ringing the same prospect/customer twice, esp. if in the first case there was no agent available.

            yield @egress_pool.remove remote_call

            answered = yield send_to_agent(@egress_pool, remote_call).catch (error) ->
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
          yield @monitor_remote_call call
          yield @ingress_pool.add call

        __transition_available_agents: seem (event) ->
          debug 'Queuer.__transition_available_agents: start', {event}
          yield nextTick()
          @available_agents.reevaluate hand (agent) ->
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
            yield agent.transition 'created'
            yield @egress_pool.add call
          else
            debug 'Queuer.create_egress_call_for: no call', agent.key
            yield agent.transition 'not_created'

        on_agent: seem (agent,data) ->
          {state} = data
          debug 'Queuer.on_agent', agent.key, state

Only states were the agent might transition via `evaluate` are considered as states were the agents is available.

          switch state
            when 'idle', 'waiting'
              yield @available_agents.add agent
            else
              yield @available_agents.remove agent

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

          switch state

            when 'new' # aka `forgotten`
              yield heal @ingress_pool.add call

            when 'pooled'
              heal @__transition_available_agents 'new_call'

            when 'bridged'
              yield @ingress_pool.remove call
              yield @egress_pool.remove call

              if data.agent_call?

                yield call.set_answered()

Hang up all other (ringing) agents.

                yield call.unbridge_except data.agent_call.key

Do not automatically close the agent's call (in `dropped`) when a remote party hangs up.

              yield call.clear()

            when 'dropped'
              yield @ingress_pool.remove call
              yield @egress_pool.remove call
              yield call.unbridge_except()

          return

        clear_timer: (key) ->
          if @__timers[key]?
            clearTimeout @__timers[key]
            delete @__timers[key]

        set_timer: (key,timer) ->
          @clear_timer key
          @__timers[key] = timer

Switch agent
------------

        set_agent: seem (call,new_key) ->
          debug 'Queuer.set_agent', call.id, new_key

          return unless call? and new_key?

          old_key = yield call.get_agent()
          return if old_key is new_key

          if old_key?
            old_agent = new Agent this, old_key
            yield old_agent.del_call call.id

          new_agent = new Agent this, new_key
          yield new_agent.add_call call.id

          yield call.set_agent new_key

          return

Agent behavior
--------------

Internal-lines, two ways to reach:
- mode A (off-hook agents): agents call into a number, are parked until a call is presented.
- mode B (on-hook agents): we ring the phone of the agent.

The state machine handles both modes.

Egress calls only really make sense in mode A.

      Queuer
