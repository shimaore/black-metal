    @name = 'black-metal'
    seem = require 'seem'

    uuidV4 = require 'uuid/v4'
    FS = require 'esl'
    debug = (require 'debug') @name

    api = (cmd) ->
      debug 'api', cmd
      new Promise (resolve,reject) ->
        try
          client = FS.client ->
            debug 'api: sending', cmd
            @api cmd
            .then resolve, reject
          client.connect (cfg.socket_port ? 5722), '127.0.0.1'
          debug 'api: client connected'
        catch error
          debug 'api: error', error
          reject error

    make_id = ->
      now = new Date() .toJSON()
      now[0...8] + uuidV4()

    module.exports = make_queuer = (redis,policy_for,egress_call_for,profile) ->

      class Pool
        constructor: (@name) ->
          debug 'new Pool', @name
          @_pool = new Set()

        add: (call) ->
          debug 'Pool.add', call
          @_pool.add call
          call.once_answered =>
            debug 'Pool.add: once_answered'
            @remove call

        remove: (call) ->
          debug 'Pool.remove', call
          @_pool.delete call

        not_presenting: ->
          debug 'Pool.not_presenting'
          result = []
          @_pool.forEach (call) ->
            if not call.presenting
              result.push call
          debug 'Pool.not_presenting', result
          result

class Call: a call from or towards a customer

       class Call
        constructor: ({@destination,@id,tags = []}) ->
          debug 'new Call', @destination, @id, tags
          unless @destination? or @id?
            throw new Error 'new Call: either destination or id is required'
          @tags = new Set tags
          @init_priority()
          @init_skills()
          @init_queues()
          @presenting = false
          @answered_cb = []

        init_priority: ->
          debug 'Call.init_priority'
          result = []
          @tags.forEach (tag) ->
            m = tag.match /^priority:(\d+)$/
            result.push m?[1]
          priorities = result.sort (a,b) ->
            switch
              when not a?
                1
              when not b?
                -1
              else
                b-a
          @priority = priorities[0] ? null

        init_skills: ->
          debug 'Call.init_skills'
          @skills_required = []
          @tags.forEach (tag) -> @skills_required.push tag if tag.match /^skill:/

        init_queues: ->
          debug 'Call.init_queues'
          @queues_required = []
          @tags.forEach (tag) -> @queues_required.push tag if tag.match /^queue:/

        in_domain: (domain) ->
          @tags.has "number_domain:#{domain}"

        originate: seem ->
          debug 'Call.originate', @destination
          id = make_id()
          yield api "originate {origination_uuid=#{id}}sofia/#{profile}/#{@destination} &park"
          # FIXME: CDRs etc.?
          id

        present: seem (agent) ->
          debug 'Call.present', agent
          @presenting = true

          try

For a dial-out (egress) call we first need to attempt to contact the destination.

            @id ?= yield @originate()

We need to send the call to the agent (using either mode A or mode B).

            agent_id = yield agent.is_off_hook_agent()
            agent_id ?= yield agent.originate()

            res = yield api "uuid_bridge #{@id} #{agent_id}"

            yield agent.transition 'answer'
            for cb in @answered_cb
              yield cb()
            @answered_cb = []
            true

          catch error
            debug "Call.present: #{error.stack ? error}"
            @presenting = false
            yield agent.transition 'timeout'
            null

        once_answered: (cb) ->
          @answered_cb.push cb

      class PossiblyIdleAgents

        constructor: (@queuer) ->
          debug 'new PossiblyIdleAgents'

        add: (agent) ->
          debug 'PossiblyIdleAgents.add', agent.key
          redis.saddAsync 'possibly-idle', agent.key

        remove: (agent) ->
          debug 'PossiblyIdleAgents.remove', agent.key
          redis.sremAsync 'possibly-idle', agent.key

        reevaluate: seem (cb) ->
          debug 'PossiblyIdleAgents.reevaluate'

reevaluate the list

          cursor = 0
          while cursor isnt '0'
            [cursor,keys] = yield redis.sscan 'possibly-idle', cursor
            debug 'PossiblyIdleAgents.reevaluate: sscan', cursor, keys

            for key in keys
              debug 'PossiblyIdleAgents.reevaluate', key
              agent = new Agent @queuer, key

if an agent isn't idle anymore, remove it

              state = yield agent.get_state()
              debug 'PossiblyIdleAgents.reevaluate state', key, state
              if state isnt 'idle'

                @remove agent

otherwise call the cb (in order) for any remaining agent

              else

                cb agent

      class Queuer

        constructor: ->
          debug 'new Queuer'
          @ingress_pool = new Pool 'ingress_pool'
          @egress_pool = new Pool 'egress_pool'
          @possibly_idle_agents = new PossiblyIdleAgents this

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

When an agent moves to the `idle` state, the queuer picks one call out of the pool:
- either an ingress calls (they are always prioritized over egress calls)
- if no ingress calls require attention, an egress call is created and assigned to the agent.

        on_agent_idle: seem (agent) ->
          debug 'Queuer.on_agent_idle', agent.key

The `agent_pool` contains (at least the topmost) calls matching for this agent.

          debug 'Queuer.on_agent_idle: ingress pool', agent.key
          call = yield agent.topmost @ingress_pool
          if call?
            debug 'Queuer.on_agent_idle present ingress call', agent.key
            yield agent.present call
            return false

          debug 'Queuer.on_agent_idle: egress pool', agent.key
          call = yield agent.topmost @egress_pool
          if call?
            debug 'Queuer.on_agent_idle present egress call', agent.key
            yield agent.present call
            return false

          debug 'Queuer.on_agent_idle no call', agent.key
          return true

Note: it's OK for agent.filter_and_sort to throw away calls that will not make it to the top, since only the first element of the resulting pool will ever be used.

        queue_ingress_call: (call) ->
          debug 'Queuer.queue_ingress_call'
          @ingress_pool.add call
          @reevaluate_idle_agents()

An egress pool is a set of dynamically constructed call instances (for example using an iterator). The topmost call instance is created using data found e.g. in a database, the (egress) call is placed in the pool, and the idle agents are re-evaluated.

        queue_egress_call: (call) ->
          debug 'Queuer.queue_egress_call'
          @egress_pool.add call
          @reevaluate_idle_agents()

        report_idle: seem (agent) ->
          debug 'Queuer.report_idle', agent.key
          yield @possibly_idle_agents.add agent
          yield @create_egress_call_for agent

        report_non_idle: (agent) ->
          debug 'Queuer.report_non_idle', agent
          @possibly_idle_agents.remove agent

        reevaluate_idle_agents: ->
          debug 'Queuer.reevaluate_idle_agents'
          @possibly_idle_agents.reevaluate (agent) =>
            debug 'Queuer.reevaluate_idle_agents', agent.key
            @on_agent_idle agent

        create_egress_call_for: seem (agent) ->
          debug 'Queuer.create_egress_call_for', agent.key
          call = yield agent.create_egress_call()
          if call?
            yield @queue_egress_call call

        on_agent: seem (agent,state) ->
          debug 'Queuer.on_agent', agent, state
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

Agent States
------------

- logged-out
- logged-in, idle (awaiting call)
- busy (e.g. on a call not managed by the queuer)
- presenting call (bip or ring out)
- accepted call = in-call
- wrap-up

        accept_off_hook_agent: seem (agent,call_uuid) ->
          if yield agent.transition 'login'
            yield agent.is_off_hook_agent call_uuid

        accept_on_hook_agent: seem (agent) ->
          id = yield agent.is_off_hook_agent()
          if id?
            @disconnect_call id?

Agent
=====

      class Agent
        constructor: (@queuer,@key) ->
          debug 'new Agent', @key
          [@number,@domain] = @key.split '@'
          @tags = new Set() # FIXME, retrieve tags for agent

        increment_external_calls: seem ->
          debug 'Agent.increment_external_calls'
          external_calls = yield @incr_value 'external_calls'

          if external_calls is 1 # was 0, presumably
            yield @transition 'start_of_external_calls'
          else
            null

        decrement_external_calls: seem ->
          debug 'Agent.decrement_external_calls'
          external_calls = yield @decr_value 'external_calls'

          if external_calls is 0
            yield @transition 'end_of_external_calls'
          else
            null

        transition: seem (event) ->
          debug 'Agent.transition', event
          old_state = yield @get_state()
          old_state ?= initial_state

          debug 'Agent.transition', event, old_state
          return false unless event of agent_transition[old_state]
          new_state = agent_transition[old_state][event]

          debug 'Agent.transition', event, old_state, new_state
          if new_state?
            yield @set_state new_state
            yield @queuer.on_agent this, new_state
            return true
          else
            return false

        is_off_hook_agent: seem (call_uuid) ->
          debug 'Agent.is_off_hook_agent', call_uuid
          if call_uuid?
            yield @set_id call_uuid
          else
            yield @get_id call_uuid

        originate: seem ->
          debug 'Agent.originate'
          id = make_id()
          yield api "originate {origination_uuid=#{id}}sofia/#{profile}/#{@key} &park"
          id

        topmost: seem (pool) ->
          debug 'Agent.topmost', pool.name
          policy = yield policy_for this

ignore ringing calls

          policy pool.not_presenting()

        create_egress_call: seem ->
          debug 'Agent.create_egress_call'
          data = yield egress_call_for this
          debug 'Agent.create_egress_call', data
          if data?
            new Call data
          else
            null

        present: seem (call) ->
          debug 'Agent.present', call
          if yield @transition 'new_queuer_call'
            yield call.present this

        get_state: ->
          redis.hgetAsync @key, 'state'

        set_state: (state) ->
          debug 'Agent.set_state', state
          redis.hsetAsync @key, 'state', state

        get_id: ->
          redis.hgetAsync @key, 'id'

        set_id: (id) ->
          debug 'Agent.set_id', state
          redis.hsetAsync @key, 'id', id

        incr_value: (field) ->
          redis.hincrbyAsync @key, field, 1

        decr_value: (field) ->
          redis.hincrbyAsync @key, field, -1

      {Agent,Queuer}

Agent Transitions
-----------------

    initial_state = 'logged_out'

    agent_transition =

### State: logged-out

The logged-out state is the initial state of the state machine.

In logged-out state an agent is not considered for calls.

      logged_out:

Event: login

The `login` event is triggered by:
- mode A: agent calls into the queuer
- mode B: agent logs in (either via TUI or GUI) and they were in no group so far.

Next state: idle

        login: 'idle'

### State: idle

      idle:

Event: logout

The `logout` event is triggered by:
- mode A: agent hangs up the call to the queuer.
- mode B: agent logs out (TUI or GUI) and there are no remaining groups they belong to.

Next state: logged-out

        logout: 'logged_out'

Event: new-external-call

The `external-call` event is triggered when a call outside of the queuer is presented or sent by the agent, making it unavailable to take new calls, without logging them out of the queuer.

Next state: busy

        start_of_external_calls: 'busy'

Event: new-queuer-call

The `new-queuer-call` event is triggered when the queuer assigns a call to an agent in the `idle` state.

        new_queuer_call: 'presenting'

Upon transitioning to the presenting state:
- the agent is presented with the call's data and client information
- mode A: the agent is presented with a bip
- mode B: the agent's phone is dialed

### State: busy

The busy state is active when an agent's phone is active but on a call not related to the queuer.

Event: end-of-external-call

The `end-of-external-call` event is triggered when all calls related to an agent (outside of the queuer) are finished.

Next state: idle

      busy:

        end_of_external_calls: ->
          'idle'

### State: presenting

The presenting state is active when an agent is presented a call but hasn't acknowledged it yet.

      presenting:

Event: timeout

A timeout might occur if the agent
- mode A: does not acknowlegde the call (TUI or GUI)
- mode B: does not answer the call

        timeout: 'logged_out'

The agent is then marked unavailable and this is reported to the manager.
- mode A: call into the queuer is forcibly hung up

Event: answer

The answer event is triggered if the agent
- mode A: acknowledges the call (TUI or GUI)
- mode B: answers the call

Next state: in-call

        answer: 'in_call'

### State: in-call

      in_call:

Event: hangup

The hangup event is triggered:
- if the agent
  - mode A: hangs up the call with the queuer (in error) or via GUI
  - mode B: hangs up the call with the queuer (in error) or via GUI
- if the remote party hangs up

Next state: wrap-up

        hangup: 'wrap_up'

### State: wrap-up

      wrap_up:

Event: complete

The complete event is triggered:
- mode A: if the agent acknowledges the wrap-up (TUI or GUI)
- mode B: if the agents hangs up or acknowledges the wrap-up (GUI)
Note: if the agent previously hung-up the wrap-up can only be ack'ed via the GUI.

Next state: idle

        complete: 'idle'
