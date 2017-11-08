Agent
=====

    @name = 'black-metal:agent'
    {debug,hand,heal} = (require 'tangible') @name
    seem = require 'seem'
    RedisClient = require 'normal-key/client'

    seconds = 1000
    minutes = 60*seconds

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

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

Monitor calls for this agent, keeping state so that we can decide whether the agent is busy or not.
(The `id` always refers to the agent's channel, not the remote end.)

      add_call: seem (id) ->
        debug 'Agent.add_call', @key, id
        return unless id?

        onhook_call = yield @get_onhook_call()
        if onhook_call? and id is onhook_call.id
          debug 'Agent.add_call: onhook agent connected (ignored)', @key, id
          return

        offhook_call = yield @get_offhook_call()
        if offhook_call? and id is offhook_call.id
          debug 'Agent.add_call: offhook agent connected (ignored)', @key, id
          return

        added = yield @add id

        if added
          debug 'Agent.add_call: added new call', @key, id
          yield @transition 'start_of_call'
        else
          debug 'Agent.add_call: call was already present', @key, id
        null

      del_call: seem (id,disposition) ->
        debug 'Agent.del_call', @key, id, disposition
        return unless id?

        removed = yield @remove id

        offhook_call = yield @get_offhook_call()
        if offhook_call? and id is offhook_call.id
          debug 'Agent.del_call: logout (for offhook agent call)', @key, id, disposition
          yield @set_offhook_call null
          yield @transition 'logout'
          return

        remote_call = yield @get_remote_call()
        if remote_call? and id is remote_call.id
          debug 'Agent.del_call: hangup (for remote call)', @key, id, disposition
          yield @clear_call remote_call
          if disposition isnt 'replaced'
            yield @transition 'hangup', call:remote_call
          return

        onhook_call = yield @get_onhook_call()
        if onhook_call? and id is onhook_call.id
          debug 'Agent.del_call: hangup (for onhook agent call)', @key, id, disposition
          yield @transition 'hangup'
          return

        count = yield @count()

        if removed and count is 0
          debug 'Agent.del_call: last call was removed', @key, count, id, disposition
          yield @transition 'end_of_calls'
        else
          debug 'Agent.del_call: calls left', @key, count, id, disposition
        null

Handle transitions
------------------

      lock: seem ->
        offset = Math.ceil 1000000 * Math.random()

        yield sleep 20 * Math.random()

        count = 7
        while count-- > 0
          before = yield @incr 'lock', offset
          return yes if before is offset
          yield sleep 20 * Math.random()
        return no

      transition: seem (event, notification_data = {}) ->
        debug 'Agent.transition', {agent: @key, event}

        old_state = yield @get_state()
        old_state ?= initial_state

        debug 'Agent.transition', {agent: @key, event, old_state}

        unless old_state of _transition
          yield @set_state initial_state
          throw new Error "Invalid state, transition from #{old_state} → event #{event}"

        unless event of _transition[old_state]
          debug "Ignoring event #{event} in state #{old_state}", {agent: @key}
          return false

        new_state = _transition[old_state][event]

        @queuer.clear_timer @key

        unless new_state of _transition
          yield @set_state initial_state
          throw new Error "Invalid state machine, transition from #{old_state} → event #{event} leads to unknown state #{new_state}"

        debug 'Agent.transition', {agent: @key, event, old_state, new_state}
        if new_state?
          yield @set_state new_state

          yield @reset 'lock'

          notification_data.old_state = old_state
          notification_data.state = new_state
          notification_data.event = event

          yield @notify? notification_data
          next_state = _transition[new_state]
          if 'timeout' of next_state and 'timeout_duration' of next_state
            on_timeout = =>
              @queuer.clear_timer @key
              heal @transition 'timeout'
            @queuer.set_timer @key, setTimeout on_timeout, next_state.timeout_duration-500+1000*Math.random()

          process.nextTick => heal @queuer.on_agent this, notification_data

          return true
        else
          return false

      __hangup_offhook: seem ->
        debug 'Agent.__hangup_offhook'
        offhook_call = yield @get_offhook_call()
        if offhook_call?
          debug 'Agent.__hangup_offhook', offhook_call.key
          yield offhook_call.hangup()
          yield @del_call offhook_call.id, 'hangup_offhook'
        offhook_call = null

Actively monitor the call between the queuer and an agent (could be an off-hook or an on-hook call).

      __monitor: seem (agent_call) ->
        debug 'Agent.__monitor', @key, agent_call?.key

        return unless agent_call?

        monitor = yield agent_call.monitor 'CHANNEL_HANGUP_COMPLETE', 'DTMF', 'CHANNEL_BRIDGE', 'CHANNEL_UNBRIDGE'

Hangup on agent call (agent can be called or callee).

        monitor?.once 'CHANNEL_HANGUP_COMPLETE', hand ({body}) =>
          disposition = body?.variable_transfer_disposition
          debug 'Agent.__monitor: CHANNEL_HANGUP_COMPLETE', @key, agent_call.key, disposition
          monitor?.end()
          monitor = null

          yield @set_onhook_call null
          call = yield @get_remote_call().catch -> null
          switch disposition
            when 'recv_replace', 'bridge'
              if yield @transition 'agent_transfer', {call}
                yield @clear_call call
            when 'replaced'
              true
            else
              if yield @transition 'agent_hangup', {call}
                yield @disconnect_remote()
          return

Bridge on agent call (calling or called).

        monitor?.on 'CHANNEL_BRIDGE', hand ({body}) =>
          a_uuid = body['Bridge-A-Unique-ID']
          b_uuid = body['Bridge-B-Unique-ID']
          debug 'Agent.__monitor: CHANNEL_BRIDGE', key, a_uuid, b_uuid

          remote_call = @new_call id:b_uuid
          yield remote_call.load()
          yield remote_call.set_agent @key
          yield @transition 'bridge', call:remote_call

          return

Unbridge on agent call (calling or called).

        monitor?.on 'CHANNEL_UNBRIDGE', hand ({body}) =>
          a_uuid = body['Bridge-A-Unique-ID']
          b_uuid = body['Bridge-B-Unique-ID']
          disposition = body.variable_transfer_disposition
          debug 'Agent.__monitor: CHANNEL_UNBRIDGE', key, a_uuid, b_uuid, disposition, body.variable_endpoint_disposition

          remote_call = @new_call id:b_uuid
          yield remote_call.load()

          if disposition is 'replaced'
            # expect body.variable_endpoint_disposition is 'ATTENDED_TRANSFER'
            yield remote_call.set_agent @key
          else
            yield @transition 'unbridge', call:remote_call

          return

        monitor?.on 'DTMF', hand ({body}) =>
          debug 'Agent.__monitor: DTMF', @key
          call = yield @get_remote_call().catch -> null
          switch body['DTMF-Digit']
            when '*', '7', '4', '1'
              if yield @transition 'force_hangup', {call}
                yield @disconnect_remote()
            when '#', '9', '6', '3'
              yield @transition 'complete', {call}
          return

        monitor

Start of an off-hook session for the agent (used by huge-play)
--------------------------------------------------------------

      accept_offhook: seem (call_uuid) ->
        debug 'Agent.accept_offhook', call_uuid
        yield @del_call call_uuid, 'accept_offhook'
        yield @__hangup_offhook()

Attempt to transition to login with the call-id.

        agent_call = @new_call id: call_uuid
        yield agent_call.load()
        unless yield @__monitor agent_call
          yield heal agent_call.hangup()
          return null

        yield agent_call.save()
        yield agent_call.set_agent @key
        yield @set_offhook_call agent_call
        unless yield @transition 'login'
          debug 'Agent.accept_offhook transition failed, hanging up'
          yield @__hangup_offhook()
          return null

        agent_call

Start of an on-hook session for the agent (used by huge-play)
-------------------------------------------------------------

      accept_onhook: seem ->
        debug 'Agent.accept_onhook'
        yield @__hangup_offhook()
        yield @transition 'login'

Originate a call towards an agent
---------------------------------

      originate: seem (caller) ->
        debug 'Agent.originate', @key

Note: assert(caller.key is (yield @get_remote_call()).key)

For off-hook the call already exists.

        offhook_call = yield @get_offhook_call()
        if offhook_call?
          return offhook_call

For on-hook we need to call the agent.

        agent_call = @new_call destination: @key
        yield agent_call.save()
        yield agent_call.set_agent @key
        agent_call = yield agent_call.originate_internal caller
        unless agent_call?
          return null

        unless yield @__monitor agent_call
          yield heal caller.remove agent_call.id
          heal agent_call.hangup()
          return null

        yield @set_onhook_call agent_call
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

      disconnect_remote: seem ->
        debug 'Agent.disconnect_remote'
        current_call = yield @get_remote_call()
        if current_call?
          yield current_call.hangup()
          yield @clear_call current_call

      clear_call: seem (call) ->
        return unless call?
        yield call.set_agent null
        yield @set_remote_call null

Tools
-----

      get_state: ->
        @get 'state'

      set_state: (state) ->
        @set 'state', state

      reset_missed: ->
        @reset 'missed'

      incr_missed: ->
        @incr 'missed'

      get_missed: seem ->
        v = yield @get 'missed'
        if v?
          parseInt v, 10
        else
          0

      get_call: seem (name) ->
        key = yield @get name
        if key?
          call = @new_call {key}
          yield call.load()
          call
        else
          null

      set_call: seem (name,call) ->
        if call?
          yield @set name, call.key
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

    _transition =

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

### Event: missed

A `missed` event might occur if the agent
- mode A: does not acknowlegde the call (TUI or GUI)
- mode B: does not answer the call

The agent is then marked `away` and this is reported to the manager.

### Event: answer

The `answer` event is triggered if the agent and the remote party are connected.

### Event: failed

The `failed` event is triggered if the call could not be presented.

### Event: hangup

The `hangup` event is triggered:
- if the agent hangs up the call via GUI (not implemented)
- if the remote party hangs up

### Event: agent_hangup

The `agent_hangup` event is triggered:
- if the agent hangs up the call with the queuer

### Event: complete

The complete event is triggered:
- mode A: if the agent acknowledges the wrap-up (TUI or GUI)
- mode B: if the agents hangs up or acknowledges the wrap-up (GUI)
Note: if the agent previously hung-up the wrap-up can only be ack'ed via the GUI.

### Event: timeout

The timeout event is triggered when an agent has been in the same state for a predefined delay.

### State: logged-out

The logged-out state is the initial state of the state machine.

In logged-out state an agent is not considered for calls.

      logged_out:

        login: 'idle'
        start_of_call: 'logged_out_busy'

### State: logged_out_busy

If an agent receives one or multiple calls in logged-out state, it is moved to the logged-out+busy state.

      logged_out_busy:

        login: 'busy'
        start_of_call: 'logged_out_busy'
        end_of_calls: 'logged_out'

Force log out on second logout.

        logout: 'logged_out'

### State: idle, waiting, pending.

An available agent is `idle`, `waiting`, or `pending`. The three states are equivalent, except from the queuer's point of view.

In `idle` state, the agent is automatically transitioned to `evaluating`.

This state should be short, so if it lasts too long resubmit it.

      idle:

        start_of_call: 'busy'
        end_of_calls: 'idle'
        logout: 'logged_out'
        evaluate: 'evaluating'
        timeout: 'idle'
        timeout_duration: 19*seconds

In `waiting` state, the agent is only transitioned on an external event `new_call`.

      waiting:

        start_of_call: 'busy'
        end_of_calls: 'idle'
        logout: 'logged_out'
        new_call: 'idle'

Regularly re-assess the situation (this allows to flush the pools).

        timeout: 'idle'
        timeout_duration: 599*seconds

All other states will re-transition via `idle` first (to force a re-evaluation of the state of the agent).

### State: evaluating

When an agent is free, we first re-evaluate whether there is any other call we may present to them.

First we look in the pool of existing ingress calls, then in the pool of existing (potential) egress calls.

      evaluating:

        start_of_call: 'busy'
        logout: 'logged_out'
        present: 'presenting'
        release: 'create_call'

This state should be relatively short, so if it lasts too long resubmit.

        timeout: 'idle'
        timeout_duration: 17*seconds

If no existing (or potential) call exists, we attempt to build a new call.

      create_call:

        start_of_call: 'busy'
        logout: 'logged_out'
        present: 'presenting'
        created: 'waiting'
        not_created: 'waiting'
        timeout: 'idle'
        timeout_duration: 71*seconds

### State: busy

The busy state is active when an agent's phone is active but on a call not related to the queuer.

      busy:

        start_of_call: 'busy'
        end_of_calls: 'idle'
        logout: 'logged_out_busy'

### State: away

In `away` state, the queuer will trigger a review of all available agents to dispatch a new call.

      away:

        start_of_call: 'busy'
        end_of_calls: 'idle'
        login: 'idle'
        logout: 'logged_out'
        timeout: 'idle'
        timeout_duration: 13*seconds

### State: presenting

The presenting state is active when an agent is presented a call but hasn't acknowledged it yet.

Upon transitioning to the presenting state:
- the agent is presented with the call's data and client information
- mode A: the agent is presented with a bip
- mode B: the agent's phone is dialed

      presenting:

        answer: 'in_call'
        missed: 'away'
        failed: 'idle'
        hangup: 'idle'

        logout: 'logged_out'

### State: in-call

      in_call:

        hangup: 'wrap_up'
        unbridge: 'wrap_up'
        force_hangup: 'wrap_up'
        agent_hangup: 'idle'
        agent_transfer: 'idle'

        logout: 'logged_out'

### State: wrap-up

      wrap_up:

        complete: 'idle'
        agent_hangup: 'idle'
        agent_transfer: 'idle' # in case the 'hangup' event arrives first

        logout: 'logged_out'

    module.exports = Agent
