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

Main Queuer
===========

    module.exports = ({redis,Agent,Call}) ->

      class Pool extends RedisClient
        constructor: (@name) ->
          debug 'new Pool', @name
          @redis = redis if redis?
          super 'pool', @name

        add: (call) ->
          debug 'Pool.add', @name, call.key
          super call.key

        remove: (call) ->
          debug 'Pool.remove', @name, call.key
          super call.key

        has: (call) ->
          debug 'Pool.has', @name, call.key
          super call.key

        calls: seem ->
          debug 'Pool.all', @key
          result = []
          yield @forEach seem (key) ->
            call = new Call {key}
            yield call.load()
            result.push call
          debug 'Pool.all', @key, result
          result

      class EgressAgents extends RedisClient

        constructor: (@queuer) ->
          debug 'new EgressAgents'
          @redis = redis if redis?
          super 'pool', 'egress-agents'
          throw new Error 'EgressAgents requires queuer' unless @queuer

        add: seem (agent) ->
          debug 'EgressAgents.add', agent.key
          score = yield agent.get_missed().catch -> 0
          score += Math.random()/100
          @sorted_add agent.key, score

        remove: (agent) ->
          debug 'EgressAgents.remove', agent.key
          @sorted_remove agent.key

        reevaluate: seem (cb) ->
          debug 'EgressAgents.reevaluate'

reevaluate the list

          yield @sorted_forEach seem (key) =>
            debug 'EgressAgents.reevaluate', key
            agent = new Agent @queuer, key

if an agent is not idle anymore, remove it

            state = yield agent.get_state()
            debug 'EgressAgents.reevaluate state', key, state
            if state isnt 'idle'

              yield @remove agent

otherwise call the cb (in order) for any remaining agent

            else

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
          @ingress_pool = new Pool 'ingress_pool'
          @egress_pool = new Pool 'egress_pool'
          @available_agents = new EgressAgents this

When an agent moves to the `idle` state, the queuer picks one call out of the pool:
- either an ingress calls (they are always prioritized over egress calls)
- if no ingress calls require attention, an egress call is created and assigned to the agent.

The method `on_agent_idle` will return true if the agent is still idle after trying to present them with some calls.
Otherwise it returns false.

        on_agent_idle: seem (agent) ->
          debug 'Queuer.on_agent_idle', agent.key

Build Call
----------

Return:
- a Call (towards or from a selected third-party),
- or `null`.

          build_call = seem (pool,remove_before = false) =>

            debug 'Queuer.on_agent_idle build_call', agent.key

The first step is for the agent to find a suitable call in the pool.

            calls = yield pool.calls()
            call = yield agent.policy calls

If no call was found the agent's state is unmodified.

            if not call?
              debug 'Queuer.on_agent_idle build_call: no call found', agent.key
              return null

For egress calls, ensure the same call is not attempted twice to two different agents (at the same time) or to the same agent (one time after the other).

            if remove_before
              yield pool.remove call

            debug 'Queuer.on_agent_idle build_call: agent is no longer available', agent.key, call.key

Notify: this could be used for example to present a popup to the agent.

            notification_data = {call}

Transition the agent.

            debug 'Queuer.on_agent_idle build_call: transition the agent', agent.key, call.key

            if not yield agent.transition 'present', notification_data
              debug 'Queuer.on_agent_idle build_call: transition failed', agent.key, call.key
              yield agent.transition 'failed', notification_data
              return null

            debug 'Queuer.on_agent_idle build_call: transitioned', agent.key, call.key

            clear_call = seem ->
              debug 'Queuer.on_agent_idle build_call: clear_call', agent.key
              monitor?.end()
              monitor = null
              yield pool.remove call
                .catch debug.catch
              yield agent.remote_hungup call
                .catch debug.catch
              return

For a dial-out (egress) call we first need to attempt to contact the destination.
For a dial-in (ingress) call we already have the proper call UUID.

            exists = yield call.originate_external()

            debug 'Queuer.on_agent_idle build_call: originate external returned', agent.key, call.key, call.id, exists

            switch exists
              when 'missing'
                yield clear_call()
                return null
              when 'failed' # only for outbound calls
                yield clear_call()
                return null

            if not call.id?
              yield clear_call()
              return null

Notify the agent of the caller's hangup.

            monitor = yield call.monitor 'CHANNEL_HANGUP_COMPLETE'
            return null unless monitor?

            monitor.once 'CHANNEL_HANGUP_COMPLETE', hand =>
              debug 'Queuer.on_agent_idle build_call: caller hung up', agent.key
              yield call.load()
              yield clear_call()

Wait a little bit (this is meant to give a popup some time to settle).

            debug 'Queuer.on_agent_idle build_call: waiting for 1.5s before originate_external', agent.key, call.key
            yield sleep 1500
            yield call.load()

            return {call,monitor}

We need to send the call to the agent (using either onhook or offhook mode).

          send_to_agent = seem (pool,{call,monitor}) ->

            debug 'Queuer.on_agent_idle send_to_agent: originate', agent.key, call.key

            agent_call = yield agent.originate call

            notification_data = {call}

            unless agent_call?
              monitor?.end()
              yield agent.incr_missed()
              yield agent.transition 'missed', notification_data
              return false

            debug 'Queuer.on_agent_idle send_to_agent: bridge', agent.key, call.key

            unless yield call.bridge agent_call
              monitor?.end()
              yield call.remove(agent_call).catch -> yes
              yield agent_call.hangup()
              yield agent.transition 'failed', notification_data
              return false

            debug 'Queuer.on_agent_idle send_to_agent: Successfully bridged', agent.key, call.key, call.id, agent_call.key

            yield call.add_tag 'bridged'
            yield call.unbridge_except agent_call.key

            yield agent.reset_missed()

            yield pool.remove(call).catch -> yes

            call.report state:'connected-to-agent', agent:agent.key

            debug 'Queuer.on_agent_idle send_to_agent: transition agent', agent.key, call.key, call.id, agent_call.key
            yield agent.transition 'answer', notification_data

Main body for `on_agent_idle`
-----------------------------

Ingress pool

          debug 'Queuer.on_agent_idle: ingress pool', agent.key

          response = yield build_call(@ingress_pool).catch (error) ->
            debug.ops 'Queuer.on_agent_idle: ingress pool, error in build_call', error.stack ? error.toString()
            null

          if response?

            debug 'Queuer.on_agent_idle: ingress pool, got client call', agent.key

            answered = yield send_to_agent(@ingress_pool, response).catch (error) ->
              debug.ops 'Queuer.on_agent_idle: ingress pool, error in send_to_agent', error.stack ? error.toString()
              response?.monitor?.end()
              null

            debug "Queuer.on_agent_idle: ingress pool, agent answered=#{answered}", agent.key
            return true if answered is null
            return false if answered

          else
            debug "Queuer.on_agent_idle: ingress pool, no matching client call", agent.key

Egress pool

          debug 'Queuer.on_agent_idle: egress pool', agent.key

          response = yield build_call(@egress_pool, true).catch (error) ->
            debug.ops 'Queuer.on_agent_idle: egress pool, error in build_call', error.stack ? error.toString()
            null

          if response?

            debug 'Queuer.on_agent_idle: egress pool, got client call', agent.key

            answered = yield send_to_agent(@egress_pool, response).catch(error) ->
              debug.ops 'Queuer.on_agent_idle: egress pool, error in send_to_agent', error.stack ? error.toString()
              response?.monitor?.end()
              null

            debug "Queuer.on_agent_idle: egress pool, agent answered=#{answered}", agent.key
            return true if answered is null
            return false if answered

          else
            debug "Queuer.on_agent_idle: egress pool, no matching client call", agent.key

No call

          debug 'Queuer.on_agent_idle: no call', agent.key
          yield agent.park()


        queue_ingress_call: seem (call) ->
          debug 'Queuer.queue_ingress_call'
          yield @ingress_pool.add call
          monitor = yield call.monitor 'CHANNEL_HANGUP_COMPLETE'
          monitor?.once 'CHANNEL_HANGUP_COMPLETE', hand ({body}) =>
            debug 'Queuer.queue_ingress_call: channel hangup complete', call.key
            monitor?.end()
            monitor = null
            yield call.load()
            switch body?.variable_hangup_cause
              when 'ATTENDED_TRANSFER'
                debug 'Queuer.queue_ingress_call: attended_transfer'
                call.report state:'transferred-queuer', event:'attended-transfer'
              when 'BLIND_TRANSFER'
                debug 'Queuer.queue_ingress_call: blind_transfer'
                call.report state:'transferred-queuer', event:'blind-transfer'
              else
                call.report state:'hungup-queuer'
                yield @hungup_ingress_call call
            return
          yield @reevaluate_idle_agents()

        hungup_ingress_call: seem (call) ->
          debug 'Queuer.hungup_ingress_call'
          yield call.load()
          yield call.unbridge_except()
          yield @ingress_pool.remove call

An egress pool is a set of dynamically constructed call instances (for example using an iterator). The call instance is created using data found e.g. in a database, the (egress) call is placed in the pool, and the idle agents are re-evaluated.

        queue_egress_call: seem (call) ->
          debug 'Queuer.queue_egress_call'
          yield @egress_pool.add call
          yield @reevaluate_idle_agents()

        report_idle: seem (agent) ->
          debug 'Queuer.report_idle', agent.key
          yield @available_agents.add agent
          yield @create_egress_call_for agent

        report_non_idle: (agent) ->
          debug 'Queuer.report_non_idle', agent.key
          @available_agents.remove agent

        reevaluate_idle_agents: ->
          debug 'Queuer.reevaluate_idle_agents'
          @available_agents.reevaluate (agent) =>
            debug 'Queuer.reevaluate_idle_agents', agent.key
            @on_agent_idle agent

        create_egress_call_for: seem (agent) ->
          debug 'Queuer.create_egress_call_for', agent.key
          call = yield agent.create_egress_call()
          if call?
            yield @queue_egress_call call

        new_idle_agent: seem (agent) ->

          yield sleep 100
          some_call = yield agent.get_remote_call()
          if some_call?
            debug.dev 'Error: Agent is idle but still has remote call', agent.key, some_call.key
          yield agent.set_remote_call null

          yield sleep 300*Math.random()
          if yield @on_agent_idle agent
            yield @report_idle agent

        on_agent: seem (agent,state) ->
          debug 'Queuer.on_agent', agent.key, state

          if state is 'logged_out'
            yield agent.reset_missed()
            yield agent.clear()

          if state is 'wrap_up'
            yield agent.wrapup()

If an agent becomes away it is because they missed a call.
That call still needs to be presented to available agents.

          if state is 'away'
            yield @reevaluate_idle_agents()

If the agent is not idle no further processing is required.

          if state isnt 'idle'
            yield @report_non_idle agent
            return

If the agent is idle, move forward in the background.

          heal @new_idle_agent agent
          return

Agent behavior
--------------

Internal-lines, two ways to reach:
- mode A (off-hook agents): agents call into a number, are parked until a call is presented.
- mode B (on-hook agents): we ring the phone of the agent.

The state machine handles both modes.

Egress calls only really make sense in mode A.

      Queuer
