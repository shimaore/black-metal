    @name = 'black-metal:queuer'
    seem = require 'seem'
    heal = (p) -> p.catch debug.catch
    hand = (f) ->
      F = seem f
      (args...) -> heal F args...
    RedisClient = require 'normal-key/client'

    debug = (require 'tangible') @name

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

    nextTick = ->
      new Promise (resolve) ->
        process.nextTick resolve

Main Queuer
===========

    module.exports = ({redis,Agent,Call}) ->

Call Pool
---------

      class CallPool extends RedisClient
        constructor: (@queuer,@name) ->
          throw new Error 'CallPool requires queuer' unless @queuer?
          debug 'new CallPool', @name
          @redis = redis if redis?
          super 'CP', @name

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
          debug 'CallPool.all', @key, result
          result

Available agents
----------------

      class AgentPool extends RedisClient

        constructor: (@queuer,@name) ->
          throw new Error 'AgentPool requires queuer' unless @queuer?
          debug 'new AgentPool'
          @redis = redis if redis?
          super 'AP', @name

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

        constructor: ->
          debug 'new Queuer'
          @ingress_pool = new CallPool this, 'ingress'
          @egress_pool = new CallPool this, 'egress'
          @available_agents = new AgentPool this, 'available'

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
            debug 'Queuer.__evaluate_agent build_call: precondition failed', agent.key, call.key

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

Notify: this could be used for example to present a popup to the agent.

            notification_data = {call}

Transition the agent.

            debug 'Queuer.__evaluate_agent build_call: transition the agent', agent.key, call.key

            if not yield agent.transition 'present', notification_data
              debug 'Queuer.__evaluate_agent build_call: transition failed', agent.key, call.key
              return null

            debug 'Queuer.__evaluate_agent build_call: transitioned', agent.key, call.key

Wait a little bit (this is meant to give a popup some time to settle).

            debug 'Queuer.__evaluate_agent build_call: waiting for 1.5s before originate_external', agent.key, call.key
            yield sleep 1500
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

Notify the agent of the caller's hangup.

            monitor = yield call.monitor 'CHANNEL_HANGUP_COMPLETE'
            unless monitor?
              yield agent.transition 'hangup', {call}
              return null

Set the remote call so that `remote_hungup` can do its job.

            yield agent.set_remote_call call

            monitor.once 'CHANNEL_HANGUP_COMPLETE', seem =>
              debug 'Queuer.__evaluate_agent build_call: caller hung up', agent.key
              monitor?.end()
              monitor = null
              yield call.load()
              yield call.transition 'hungup', {agent}
              return

            return {call,monitor}

### Send to Agent

We need to send the call to the agent (using either onhook or offhook mode).

          send_to_agent = seem (pool,{call,monitor}) ->

            debug 'Queuer.__evaluate_agent send_to_agent: originate', agent.key, call.key

            agent_call = yield agent.originate call

            notification_data = {call}

            unless agent_call?
              monitor?.end()
              monitor = null
              yield agent.set_remote_call null
              yield agent.incr_missed()
              yield agent.transition 'missed', notification_data
              heal call.transition 'pool'
              return false

            debug 'Queuer.__evaluate_agent send_to_agent: bridge', agent.key, call.key

            unless yield call.bridge agent_call
              monitor?.end()
              monitor = null
              yield heal call.remove agent_call.id # undo what was done in `call.originate_internal`
              yield agent.set_remote_call null
              yield agent_call.hangup()
              yield agent.transition 'failed', notification_data
              return false

            debug 'Queuer.__evaluate_agent send_to_agent: Successfully bridged', agent.key, call.key, call.id, agent_call.key
            yield agent.transition 'answer', notification_data

### Main body for `__evaluate_agent`

          some_call = yield agent.get_remote_call()
          if some_call?
            debug.dev 'Error: Agent was idle/waiting but still had a remote call', agent.key, some_call.key
          yield agent.set_remote_call null

Ingress pool

          debug 'Queuer.__evaluate_agent: ingress pool', agent.key

          response = yield build_call(@ingress_pool).catch (error) ->
            debug.ops 'Queuer.__evaluate_agent: ingress pool, error in build_call', error.stack ? error.toString()
            null

          if response?

            debug 'Queuer.__evaluate_agent: ingress pool, got client call', agent.key

            answered = yield send_to_agent(@ingress_pool, response).catch (error) ->
              debug.ops 'Queuer.__evaluate_agent: ingress pool, error in send_to_agent', error.stack ? error.toString()
              response?.monitor?.end()
              null

            debug "Queuer.__evaluate_agent: ingress pool, agent answered=#{answered}", agent.key
            return true if answered is null
            return false if answered

          else
            debug "Queuer.__evaluate_agent: ingress pool, no matching client call", agent.key

Egress pool

          debug 'Queuer.__evaluate_agent: egress pool', agent.key

          response = yield build_call(@egress_pool, true).catch (error) ->
            debug.ops 'Queuer.__evaluate_agent: egress pool, error in build_call', error.stack ? error.toString()
            null

          if response?

            debug 'Queuer.__evaluate_agent: egress pool, got client call', agent.key

            @egress_pool.remove response.call

            answered = yield send_to_agent(@egress_pool, response).catch(error) ->
              debug.ops 'Queuer.__evaluate_agent: egress pool, error in send_to_agent', error.stack ? error.toString()
              response?.monitor?.end()
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
          debug 'Queuer.queue_ingress_call'
          monitor = yield call.monitor 'CHANNEL_HANGUP_COMPLETE'
          monitor?.once 'CHANNEL_HANGUP_COMPLETE', hand ({body}) =>
            debug 'Queuer.queue_ingress_call: channel hangup complete', call.key
            monitor?.end()
            monitor = null
            switch body?.variable_hangup_cause
              when 'ATTENDED_TRANSFER'
                debug 'Queuer.queue_ingress_call: attended_transfer'
                call.transition 'attended-transfer'
              when 'BLIND_TRANSFER'
                debug 'Queuer.queue_ingress_call: blind_transfer'
                call.transition 'blind-transfer'
              else
                call.transition 'hungup'
            return
          yield @ingress_pool.add call

        __transition_available_agents: (event) ->
          debug 'Queuer.__transition_available_agents: postpone', {event}
          process.nextTick =>
            debug 'Queuer.__transition_available_agents: start', {event}
            @available_agents.reevaluate (agent) ->
              debug 'Queuer.__transition_available_agents for agent', {agent: agent.key, event}
              agent.transition event
          return

        create_egress_call_for: seem (agent) ->
          debug 'Queuer.create_egress_call_for', agent.key

The call instance is created using data found e.g. in a database, the (egress) call is placed in the pool, and the idle agents are re-evaluated.

          call = yield agent.create_egress_call()
          if call?
            debug 'Queuer.create_egress_call_for: queue egress call', agent.key, call.key
            yield agent.transition 'created'
            yield @egress_pool.add call
          else
            debug 'Queuer.create_egress_call_for: no call', agent.key
            yield agent.transition 'not_created'

        on_agent: seem (agent,data) ->
          {state,event} = data
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

          switch data.state

            when 'new' # aka `forgotten`
              yield heal @ingress_pool.add call

            when 'pooled'
              @__transition_available_agents 'new_call'

            when 'bridged'
              yield @ingress_pool.remove call
              yield @egress_pool.remove call
              yield call.unbridge_except data.agent_call.key

            when 'dropped'
              yield @ingress_pool.remove call
              yield @egress_pool.remove call
              if data.agent?
                yield heal agent.remote_hungup call
              else
                yield heal call.unbridge_except()

Agent behavior
--------------

Internal-lines, two ways to reach:
- mode A (off-hook agents): agents call into a number, are parked until a call is presented.
- mode B (on-hook agents): we ring the phone of the agent.

The state machine handles both modes.

Egress calls only really make sense in mode A.

      Queuer
