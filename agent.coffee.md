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

        offhook_call = yield @get_offhook_call()
        if offhook_call? and id is offhook_call.id
          debug 'Agent.del_call: logout for offhook agent call'
          yield @set_offhook_call null
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

      __hangup_offhook: seem ->
        debug 'Agent.__hangup_offhook'
        offhook_call = yield @get_offhook_call()
        if offhook_call?
          debug 'Agent.__hangup_offhook', offhook_call.key
          yield offhook_call.hangup()
          yield @del_call offhook_call.id
        offhook_call = null

Actively monitor the call between the queuer and an agent (could be an off-hook or an on-hook call).

      __monitor: seem (agent_call) ->
        debug 'Agent._monitor', @key

        return unless agent_call?

        monitor = yield agent_call.monitor()

        monitor?.once 'CHANNEL_HANGUP_COMPLETE', seem =>
          debug 'Agent._monitor: channel hangup complete', @key
          yield @set_onhook_call null
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

        agent_call = @new_call id: call_uuid
        yield agent_call.save()
        yield @set_offhook_call agent_call
        unless yield @transition 'login'
          debug 'Agent.accept_offhook transition failed, hanging up'
          yield @__hangup_offhook()
          return false

        @__monitor agent_call
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

        offhook_call = yield @get_offhook_call()
        if offhook_call?
          return offhook_call

For on-hook we need to call the agent.

        agent_call = @new_call destination: @key
        agent_call = yield agent_call.originate_internal caller
        yield @set_onhook_call agent_call
        @__monitor agent_call

        agent_call

Park an agent, indicating end-of-call + end-of-wrapup
-----------------------------------------------------

      park: seem ->
        debug 'Agent.park', @key

Actually park an off-hook agent.

        agent_call = yield @get_offhook_call()
        if agent_call?
          yield agent_call.park()

On-hook agents don't need to be parked, they should hangup.

        else
          true

Notify start of wrapup time to an agent
---------------------------------------

      wrapup: seem ->
        debug 'Agent.wrapup', @key

        agent_call = yield @get_offhook_call()
        agent_call ?= yield @get_onhook_call()
        if agent_call?
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
        notification_data =
          key: call.key
          destination: call.destination
        if yield @transition 'present', notification_data
          switch yield call.present this
            when true # success
              yield @set_remote_call call
              yield @transition 'answer', notification_data
            when false # failure, agent-side
              yield @transition 'logout'
            else # failure, other
              yield @transition 'timeout', notification_data

      disconnect_remote: seem ->
        debug 'Agent.disconnect_remote'
        current_call = yield @get_remote_call()
        if current_call?
          yield current_call.hangup()

Tools
-----

      get_state: ->
        @get 'state'

      set_state: (state) ->
        @set 'state', state


      get_call: seem (name) ->
        key = yield @get name
        if key?
          call = yield @new_call {key}
          yield call.load()
          call
        else
          null

      set_call: seem (name,call) ->
        if call?
          yield call.save()
          yield @set name, call?.key
        else
          yield @set name, null


      get_offhook_call: ->
        debug 'get_offhook_call'
        @get_call 'offhook-call'

      set_offhook_call: (offhook_call) ->
        debug 'set_offhook_call'
        @set_call 'offhook-call', offhook_call

      get_onhook_call: ->
        debug 'get_onhook_call'
        @get_call 'onhook-call'

      set_onhook_call: (onhook_call) ->
        debug 'set_onhook_call'
        @set_call 'onhook-call', onhook_call

      get_remote_call: ->
        debug 'get_remote_call'
        @get_call 'remote-call'

      set_remote_call: (remote_call)->
        debug 'set_remote_call'
        @set_call 'remote-call', remote_call

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
