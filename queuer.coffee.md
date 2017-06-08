    @name = 'black-metal:queuer'
    seem = require 'seem'
    RedisClient = require './redis'

    uuidV4 = require 'uuid/v4'
    debug = (require 'tangible') @name

    make_id = ->
      now = new Date() .toJSON()
      now[0...8] + uuidV4()

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

        not_presenting: seem ->
          debug 'Pool.not_presenting', @key
          result = []
          yield @forEach seem (key) =>
            call = new Call {key}
            yield call.load()
            if not yield call.presenting()
              result.push call
          debug 'Pool.not_presenting', @key, result
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
          @egress_agents = new EgressAgents this

When an agent moves to the `idle` state, the queuer picks one call out of the pool:
- either an ingress calls (they are always prioritized over egress calls)
- if no ingress calls require attention, an egress call is created and assigned to the agent.

        on_agent_idle: seem (agent) ->
          debug 'Queuer.on_agent_idle', agent.key

          for_pool = seem (pool) =>
            call = yield agent.topmost pool
            if call?
              debug 'Queuer.on_agent_idle present call', agent.key

This next line is redundant with what happens in `report_non_idle`, I guess.

              yield @egress_agents.remove agent

              yield agent.notify? 'present',
                call: call.key
                remote_number: yield call.get_remote_number()

              result = yield agent.present call
              switch result
                when 'answer'
                  yield pool.remove call
                  monitor = yield call.monitor()
                  monitor?.once 'CHANNEL_HANGUP_COMPLETE', seem =>
                    monitor.end()
                    yield agent.transition 'hangup'
                    monitor = null
                  return true
                when 'missed'
                  return false
            null

The `agent_pool` contains (at least the topmost) calls matching for this agent.

          debug 'Queuer.on_agent_idle: ingress pool', agent.key
          result = yield for_pool @ingress_pool
          return result if result?

          debug 'Queuer.on_agent_idle: egress pool', agent.key
          result = yield for_pool @egress_pool
          return result if result?

          debug 'Queuer.on_agent_idle no call', agent.key
          yield agent.park()

Note: it is OK for agent.filter_and_sort to throw away calls that will not make it to the top, since only the first element of the resulting pool will ever be used.

        queue_ingress_call: seem (call) ->
          debug 'Queuer.queue_ingress_call'
          yield call.save()
          yield @ingress_pool.add call
          monitor = yield call.monitor()
          monitor?.once 'CHANNEL_HANGUP_COMPLETE', seem =>
            yield @hungup_ingress_call call
            yield monitor.end()
          yield @reevaluate_idle_agents()

        hungup_ingress_call: seem (call) ->
          debug 'Queuer.hungup_ingress_call'
          yield call.load()
          yield call.unbridge()
          yield @ingress_pool.remove call

An egress pool is a set of dynamically constructed call instances (for example using an iterator). The call instance is created using data found e.g. in a database, the (egress) call is placed in the pool, and the idle agents are re-evaluated.

        queue_egress_call: seem (call) ->
          debug 'Queuer.queue_egress_call'
          yield call.save()
          yield @egress_pool.add call
          yield @reevaluate_idle_agents()

        report_idle: seem (agent) ->
          debug 'Queuer.report_idle', agent.key
          yield @egress_agents.add agent
          yield @create_egress_call_for agent

        report_non_idle: (agent) ->
          debug 'Queuer.report_non_idle', agent.key
          @egress_agents.remove agent

        reevaluate_idle_agents: ->
          debug 'Queuer.reevaluate_idle_agents'
          @egress_agents.reevaluate (agent) =>
            debug 'Queuer.reevaluate_idle_agents', agent.key
            @on_agent_idle agent

        create_egress_call_for: seem (agent) ->
          debug 'Queuer.create_egress_call_for', agent.key
          call = yield agent.create_egress_call()
          if call?
            yield @queue_egress_call call

        on_agent: seem (agent,state) ->
          debug 'Queuer.on_agent', agent.key, state

          if state is 'logged_out'
            yield agent.reset_missed()
            yield agent.clear()

          if state is 'wrap_up'
            yield agent.wrapup()

          if state is 'terminate_call'
            yield agent.disconnect_remote()

Only re-process if an agent is idle.

          if state is 'idle'
            if yield @on_agent_idle agent
              yield @report_idle agent
          else
            yield @report_non_idle agent

Agent behavior
--------------

Internal-lines, two ways to reach:
- mode A (off-hook agents): agents call into a number, are parked until a call is presented.
- mode B (on-hook agents): we ring the phone of the agent.

The state machine handles both modes.

Egress calls only really make sense in mode A.

      Queuer
