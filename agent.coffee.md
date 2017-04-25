Agent
=====

    @name = 'black-metal:agent'
    debug = (require 'debug') @name
    seem = require 'seem'
    RedisClient = require './redis'

    class Agent extends RedisClient
      constructor: (@queuer,key) ->
        throw new Error 'Agent requires queuer' unless @queuer?
        super 'agent', key
        debug 'new Agent', @key
        [@number,@domain] = @key.split '@'

Virtual features
----------------

These are meants to be overriden in sub-classes.

Default policy: accept inbound calls in random order across all domains. (Not want you'd want in production.)

      policy: (calls) -> calls[0]

Default create policy: no egress calls, queuer is ingress-only.

      create_egress_call: -> null

This is meant to be defined in a sub-calls.

      # new_call: (data) -> new Call data

Base features
-------------

Monitor other calls for this agent, keeping state.

      add_call: seem (id) ->
        debug 'Agent.add_call', id
        return unless id?

        added = yield @add id

        if added
          yield @transition 'start_of_call'
        null

      del_call: seem (id) ->
        debug 'Agent.del_call', id
        return unless id?

        removed = yield @remove id

        if id is yield @is_offhook_agent()
          debug 'Agent.del_call: logout for offhook agent call'
          yield @set_id null
          yield @transition 'logout'
          return

        count = yield @count()

        if removed and count is 0
          yield @transition 'end_of_calls'
        null

Handle transitions

      transition: seem (event, notification_data = null) ->
        debug 'Agent.transition', event
        old_state = yield @get_state()
        old_state ?= initial_state

        debug 'Agent.transition', event, old_state

        unless old_state of agent_transition
          yield @set_state initial_state
          throw new Error "Invalid state, transition from #{old_state} → event #{event}"

        unless event of agent_transition[old_state]
          debug "Ignoring event #{event} in state #{old_state}"
          return false

        new_state = agent_transition[old_state][event]

        unless new_state of agent_transition
          yield @set_state initial_state
          throw new Error "Invalid state machine, transition from #{old_state} → event #{event} leads to unknown state #{new_state}"

        debug 'Agent.transition', event, old_state, new_state
        if new_state?
          yield @set_state new_state
          yield @notify? new_state, notification_data
          yield @queuer.on_agent this, new_state
          return true
        else
          return false

      is_offhook_agent: seem ->
        debug 'Agent.is_offhook_agent'
        result = yield @get_id()
        debug 'Agent.is_offhook_agent', result
        result

      __hangup_offhook: seem ->
        debug 'Agent.hangup_offhook'
        id = yield @is_offhook_agent()
        if id?
          debug 'Agent.hangup_offhook', id
          call = @new_call {id}
          yield call.hangup()
          yield @del_call id
        id

Actively monitor the call between the queuer and an agent.

      _monitor: seem (agent_call) ->
        debug 'Agent._monitor', @key
        monitor = yield agent_call.monitor()

        monitor?.once 'CHANNEL_HANGUP_COMPLETE', seem =>
          debug 'Agent._monitor: channel hangup complete', @key
          yield @set_onhook_id null
          yield @transition 'agent_hangup'
          yield monitor.end()
          monitor = null

        monitor?.on 'DTMF', seem ({body}) =>
          debug 'Agent._monitor: DTMF', @key
          switch body['DTMF-Digit']
            when '*'
              yield @transition 'force_hangup'
            when '#'
              yield @transition 'complete'

        monitor

Start of an off-hook session for the agent
------------------------------------------

      accept_offhook: seem (call_uuid) ->
        debug 'Agent.accept_offhook', call_uuid
        yield @del_call call_uuid
        yield @__hangup_offhook()

Attempt to transition to login with the call-id.

        yield @set_id call_uuid
        unless yield @transition 'login'
          debug 'Agent.accept_offhook transition failed, hanging up'
          yield @__hangup_offhook()
          return false

        agent_call = @new_call id: call_uuid
        @_monitor agent_call
        true

Start of an on-hook session for the agent
-----------------------------------------

      accept_onhook: seem ->
        debug 'Agent.accept_onhook'
        yield @__hangup_offhook()
        yield @transition 'login'

Originate a call towards an agent
---------------------------------

      originate: seem (caller) ->
        debug 'Agent.originate', @key

For off-hook the call already exists.

        id = yield @is_offhook_agent()
        if id?
          return id

For on-hook we need to call the agent.

        agent_call = @new_call destination: @key
        id = yield agent_call.originate_internal caller
        yield @set_onhook_id id
        @_monitor agent_call

        id

Park an agent, indicating end-of-call + end-of-wrapup
-----------------------------------------------------

      park: seem ->
        debug 'Agent.park', @key

Actually park an off-hook agent.

        id = yield @is_offhook_agent()
        if id?
          agent_call = @new_call {id}
          yield agent_call.park()

On-hook agents don't need to be parked, they should hangup.

        else
          true

Notify start of wrapup time to an agent
---------------------------------------

      wrapup: seem ->
        debug 'Agent.wrapup', @key

        id = yield @is_offhook_agent()
        id ?= yield @get_onhook_id()
        if id?
          agent_call = @new_call {id}
          yield agent_call.wrapup()

Topmost call for this agent
---------------------------

      topmost: seem (pool) ->
        debug 'Agent.topmost', @key, pool.name

ignore ringing calls

        @policy yield pool.not_presenting()

Present a call to this agent
----------------------------

      present: seem (call) ->
        debug 'Agent.present', call
        if yield @transition 'present', call.destination
          switch yield call.present this
            when true # success
              yield @set_current_call call.key
              yield @transition 'answer', call.destination
            when false # failure, agent-side
              yield @transition 'logout'
            else # failure, other
              yield @transition 'timeout', call.destination

      disconnect_remote: seem ->
        debug 'Agent.disconnect_remote'
        key = yield @get_current_call()
        if key?
          call = @new_call {key}
          call.load()
          yield call.hangup()

Tools
-----

      get_state: ->
        @get 'state'

      set_state: (state) ->
        @set 'state', state

      get_id: ->
        @get 'id'

      set_id: (id) ->
        @set 'id', id

      get_onhook_id: ->
        @get 'agent_id'

      set_onhook_id: (id) ->
        @set 'agent_id', id

      get_current_call: ->
        @get 'current-call'

      set_current_call: (remote_key)->
        @set 'current-call', remote_key

Agent Transitions
-----------------

    initial_state = 'logged_out'

    agent_transition =

### Event: login

The `login` event is triggered by:
- mode A: agent calls into the queuer
- mode B: agent logs in (either via TUI or GUI) and they were in no group so far.

### Event: logout

The `logout` event is triggered by:
- mode A: agent hangs up the call to the queuer.
- mode B: agent logs out (TUI or GUI) and there are no remaining groups they belong to.

### Event: start-of-call

The `start_of_call` event is triggered when a call outside of the queuer is presented or sent by the agent, making it unavailable to take new calls, without logging them out of the queuer.

### Event: end-of-calls

The `end_of_calls` event is triggered when all calls related to an agent (outside of the queuer) are finished.

### Event: present

The `present` event is triggered when the queuer assigns a call to an agent in the `idle` state.

### Event: timeout

A `timeout` event might occur if the agent
- mode A: does not acknowlegde the call (TUI or GUI)
- mode B: does not answer the call

The agent is then marked unavailable and this is reported to the manager.
- mode A: call into the queuer is forcibly hung up

### Event: answer

The `answer` event is triggered if the agent and the remote party are connected.

### Event: hangup

The `hangup` event is triggered:
- if the agent hangs up the call via GUI
- if the remote party hangs up

### Event: agent_hangup

The `agent_hangup` event is triggered:
- if the agent
  - mode B: hangs up the call with the queuer

### Event: complete

The complete event is triggered:
- mode A: if the agent acknowledges the wrap-up (TUI or GUI)
- mode B: if the agents hangs up or acknowledges the wrap-up (GUI)
Note: if the agent previously hung-up the wrap-up can only be ack'ed via the GUI.

### State: logged-out

The logged-out state is the initial state of the state machine.

In logged-out state an agent is not considered for calls.

      logged_out:

        login: 'idle'

### State: idle

      idle:

        start_of_call: 'busy'
        end_of_calls: 'idle'
        logout: 'logged_out'
        present: 'presenting'

### State: busy

The busy state is active when an agent's phone is active but on a call not related to the queuer.

      busy:

        start_of_call: 'busy'
        end_of_calls: 'idle'
        logout: 'logged_out'

### State: presenting

The presenting state is active when an agent is presented a call but hasn't acknowledged it yet.

Upon transitioning to the presenting state:
- the agent is presented with the call's data and client information
- mode A: the agent is presented with a bip
- mode B: the agent's phone is dialed

      presenting:

        answer: 'in_call'
        timeout: 'idle'

        logout: 'logged_out'

### State: in-call

      in_call:

        hangup: 'wrap_up'
        force_hangup: 'terminate_call'
        agent_hangup: 'idle'

        logout: 'logged_out'

      terminate_call:

        hangup: 'wrap_up'
        agent_hangup: 'idle'

        logout: 'logged_out'

### State: wrap-up

      wrap_up:

        complete: 'idle'
        agent_hangup: 'idle'

        logout: 'logged_out'

    module.exports = Agent
