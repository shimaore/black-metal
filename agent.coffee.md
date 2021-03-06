Agent
=====

    {debug,heal} = (require 'tangible') 'black-metal:agent'
    RedisClient = require 'normal-key/client'

    Solid = require 'solid-gun'
    make_id = ->
      Solid.time() + Solid.uniqueness()

    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout
    nextTick = -> new Promise (resolve) -> process.nextTick resolve

    transition = require './agent-state-machine'

Transfer-disposition values:
- `recv_replace`: we transfered the call out (blind transfer). (REFER To)
- `replaced`: we accepted an inbound, supervised-transfer call. (Attended Transfer on originating session.)
- `bridge`: we transfered the call out (supervised transfer).

    BLIND_TRANSFER = 'recv_replace'
    SUPERVISED_TRANSFER = 'bridge'
    ACCEPT_SUPERVISED_TRANSFER = 'replaced'

    {EventEmitter2} = require 'eventemitter2'
    events = new EventEmitter2()

    class Agent extends RedisClient

      @events: events

      # queuer: null # must be defined

      constructor: (key) ->
        debug 'new Agent', key
        ### istanbul ignore next ###
        throw new Error 'Agent requires key' unless key?
        [number,domain] = key.split '@'
        ### istanbul ignore next ###
        throw new Error "Agent requires number: #{key}" unless number?
        ### istanbul ignore next ###
        throw new Error "Agent requires domain: #{key}" unless domain?
        super 'agent', key
        @key = key
        @number = number
        @domain = domain

Virtual features
----------------

These are meants to be overriden in sub-classes.

Default policy: accept inbound calls in random order across all domains. (Not want you'd want in production.)

      policy: (calls) -> calls[0]

Default create policy: no egress calls, queuer is ingress-only.

      create_egress_call: -> null

Base features
-------------

### on-bridge

      on_bridge: (call,disposition,our_call) ->
        debug 'Agent.on_bridge', @key, call.key, disposition, our_call?.key

        switch
          when await @is_remote_call call
            debug 'Agent.on_bridge: queuer-managed remote-call connected (ignored)', @key, call.key
            events.emit [@key,'remote','bridge'], call, disposition, our_call

          when await @is_onhook_call call
            debug 'Agent.on_bridge: onhook agent connected (ignored)', @key, call.key
            events.emit [@key,'onhook','bridge'], call, disposition, our_call

          when await @is_offhook_call call
            debug 'Agent.on_bridge: offhook agent connected (ignored)', @key, call.key
            events.emit [@key,'offhook','bridge'], call, disposition, our_call

          else
            events.emit [@key,'external','bridge'], call, disposition, our_call

            added = await @add call.key

            if added
              debug 'Agent.on_bridge: added new call', @key, call.key
              await @transition 'bridge', {call}
            else
              debug.dev 'Agent.on_bridge: Error: call was already present', @key, call.key


        null

### on-unbridge

Unbridge might happens because of transfers or because of hang-up.

      on_unbridge: (call,disposition,our_call) ->
        debug 'Agent.on_unbridge', @key, call.key, disposition, our_call?.key

Remove a call-leg from the list of connected call-legs.

        removed = await @remove call.key
        count = await @count()

        switch

          when await @is_remote_call call
            debug 'Agent.on_unbridge: queuer-manager remote call disconnected', @key, call.key, disposition
            events.emit [@key,'remote','unbridge'], call, disposition, our_call

            await @set_remote_call null
            await call.set_remote_agent null

            switch disposition
              when BLIND_TRANSFER
                debug.dev 'Agent.on_unbridge: unexpected BLIND_TRANSFER on remote_call'
                no
              when SUPERVISED_TRANSFER
                debug.dev 'Agent.on_unbridge: unexpected SUPERVISED_TRANSFER on remote_call'
                no
              when ACCEPT_SUPERVISED_TRANSFER
                debug.dev 'Agent.on_unbridge: unexpected ACCEPT_SUPERVISED_TRANSFER on remote_call'
                no
              else
                await @transition 'unbridge', {call}

          when await @is_onhook_call call
            debug 'Agent.on_unbridge: on-hook agent disconnected', @key, call.key, disposition
            events.emit [@key,'unhook','unbridge'], call, disposition, our_call

            switch disposition
              when BLIND_TRANSFER
                if await @transition 'agent_transfer', {call}
                  await call.set_remote_agent null
              when SUPERVISED_TRANSFER
                if await @transition 'agent_transfer', {call}
                  await call.set_remote_agent null
              when ACCEPT_SUPERVISED_TRANSFER
                yes

          when await @is_offhook_call call
            debug 'Agent.on_unbridge: off-hook agent disconnected', @key, call.key, disposition
            events.emit [@key,'offhook','unbridge'], call, disposition, our_call

            switch disposition
              when BLIND_TRANSFER
                if await @transition 'agent_transfer', {call}
                  await call.set_remote_agent null
              when SUPERVISED_TRANSFER
                if await @transition 'agent_transfer', {call}
                  await call.set_remote_agent null
              when ACCEPT_SUPERVISED_TRANSFER
                debug.dev 'Agent.on_unbridge: unexpected ACCEPT_SUPERVISED_TRANSFER on onhook_call'
                yes
              else
                no

All calls are assumed to be "other calls".

          else
            debug 'Agent.on_unbridge: other call disconnected', @key, call.key, disposition
            events.emit [@key,'external','unbridge'], call, disposition, our_call

            if removed and count is 0
              debug 'Agent.on_unbridge: last call was removed', @key, count, call.key, disposition
              await @transition 'end_of_calls'
            else
              debug 'Agent.on_unbridge: calls left', @key, count, call.key, disposition
              await @transition 'unbridge', {call}

        return

### on-hangup

Note that `hangup` may happen in two cases:
- the call is terminated before it ever gets connected (there are no bridge/unbridge events);
- the call is terminated after it gets connected (there might have been multiple bridge/unbridge events).

      on_hangup: (call,disposition) ->
        debug 'Agent.on_hangup', @key, call.key, disposition

Notice: we `await nextTick()` for the same reason that we do it in the `transition` module: when we receive `unbridge` and `hangup` back-to-back from FreeSwitch, the transitions might fail because they overlap.
We could also retry the transition (once or twice) to alleviate the issue.

        switch

Remote call was hung up

Note: if the call was ever bridged, `on_unbridge` will have already cleared the remote-call, so we'll fall-through and ignore.

          when await @is_remote_call call
            debug 'Agent.on_hangup: queuer-managed remote call hang up', @key, call.key, disposition

            await @set_remote_call null
            await call.set_remote_agent null

            await nextTick()

            switch disposition
              when BLIND_TRANSFER
                debug.dev 'Agent.on_hangup: unexpected BLIND_TRANSFER on remote_call'
                no
              when SUPERVISED_TRANSFER
                debug.dev 'Agent.on_hangup: unexpected SUPERVISED_TRANSFER on remote_call'
                no
              when ACCEPT_SUPERVISED_TRANSFER
                debug.dev 'Agent.on_hangup: unexpected ACCEPT_SUPERVISED_TRANSFER on remote_call'
                no
              else
                await @transition 'hangup', {call}

Onhook agent call was hung up

          when await @is_onhook_call call
            debug 'Agent.on_hangup: on-hook agent call hung up', @key, call.key, disposition

            await @set_onhook_call null
            await @set_remote_call null # maybe

Sometimes when the agent hangs up, the `unbridge` and the `hangup` messages arrive too close, and the transition for the `hangup` message will fail (and the call is stuck in `wrap_up` state, typically). So we delay the `hangup` message a little, and retry if needed.

            await sleep 50

            twice = (what) ->
              await what() or (
                await sleep 100
                await what()
              )

            switch disposition
              when BLIND_TRANSFER
                if await twice => @transition 'agent_transfer', {call}
                  await call.set_remote_agent null
              when SUPERVISED_TRANSFER
                if await twice => @transition 'agent_transfer', {call}
                  await call.set_remote_agent null
              when ACCEPT_SUPERVISED_TRANSFER
                debug.dev 'Agent.on_hangup: unexpected ACCEPT_SUPERVISED_TRANSFER on onhook_call'
                no
              else
                await twice => @transition 'agent_hangup', {call}

Offhook agent call was hung up

          when await @is_offhook_call call
            debug 'Agent.on_hangup: off-hook agent call hung up', @key, call.key, disposition

            await @set_offhook_call null
            await @set_remote_call null # maybe

            await nextTick()

            switch disposition
              when BLIND_TRANSFER
                debug.dev 'Agent.on_hangup: unexpected BLIND_TRANSFER on offhook_call'
                no
              when SUPERVISED_TRANSFER
                debug.dev 'Agent.on_hangup: unexpected SUPERVISED_TRANSFER on remote_call'
                no
              when ACCEPT_SUPERVISED_TRANSFER
                debug.dev 'Agent.on_hangup: unexpected ACCEPT_SUPERVISED_TRANSFER on offhook_call'
                no
              else
                await @transition 'logout', {call}

          else
            debug 'Agent.on_hangup: other call hung up (ignored)', @key, call.key, disposition

        return

Handle transitions
------------------

      transition: transition

      post_process: (notification_data) -> @queuer.on_agent this, notification_data

Commands to FreeSwitch
----------------------

      __hangup_offhook: ->
        debug 'Agent.__hangup_offhook'
        offhook_call = await @get_offhook_call()
        if offhook_call?
          debug 'Agent.__hangup_offhook', offhook_call.key
          await offhook_call.hangup()
          await @del_call offhook_call.key, 'hangup_offhook'
        offhook_call = null

Unbridge on agent call (calling or called).

      on_dtmf: (digit) ->
        debug 'Agent.on_dtmf', @key, digit
        call = await @get_remote_call().catch -> null
        switch digit
          when '*', '7', '4', '1'
            if await @transition 'force_hangup', {call}
              await @disconnect_remote()
          when '#', '9', '6', '3'
            await @transition 'complete', {call}
        return

Start of an off-hook session for the agent (used by huge-play)
--------------------------------------------------------------

      accept_offhook: (call_uuid) ->
        debug 'Agent.accept_offhook', call_uuid
        call = new @Call call_uuid
        await call.set_domain @domain

Remove the call from the list of active calls, so that it doesn't get disconnected.

        await @on_unbridge call, 'accept_offhook', null

Hangup any other (potential) offhook call.

        await @__hangup_offhook()

Attempt to transition to login with the call-id.

        agent_call = new @Call call_uuid
        await agent_call.set_domain @domain
        await agent_call.set_id call_uuid
        await agent_call.set_local_agent @key
        await @set_offhook_call agent_call
        unless await @transition 'login'
          debug 'Agent.accept_offhook transition failed, hanging up'
          await @__hangup_offhook()
          return null

        agent_call

Start of an on-hook session for the agent (used by huge-play)
-------------------------------------------------------------

      accept_onhook: ->
        debug 'Agent.accept_onhook'
        await @__hangup_offhook()
        await @transition 'login'

Originate a call towards an agent
---------------------------------

      originate_to_agent: ->
        debug 'Agent.originate_to_agent', @key

For off-hook the call already exists.

        offhook_call = await @get_offhook_call()
        if offhook_call?
          debug 'Agent.originate_to_agent: using offhook call', @key, offhook_call.key
          return offhook_call

For on-hook we need to call the agent.

        onhook_call = await @get_onhook_call()

        if onhook_call?
          debug.dev 'Agent.originate_to_agent: trying to create a new onhook call when one already exists', @key, onhook_call.key
          return reason: 'duplicate-onhook-call'

        agent_call = new @Call make_id()
        await agent_call.set_domain @domain
        await agent_call.set_local_agent @key

        remote = await @get_remote_call()
        unless remote?
          debug.dev 'Agent.originate_to_agent: missing remote-call', @key
          return reason: 'missing-remote-call'

        await @set_onhook_call agent_call
        reason = await agent_call.originate_internal remote, @key
        if reason?
          debug 'Agent.originate_to_agent failed', @key, agent_call.key, reason
          return {reason}

        return agent_call

Originate a call from an agent
------------------------------

      originate_remote_call: (call) ->
        debug 'Agent.originate_remote_call', @key, call.key

        unless await @transition 'present', {call}
          debug 'Agent.originate_remote_call: transition failed', @key, call.key
          return false

Wait a little bit (this is meant to give a popup some time to settle).

        debug 'Agent.originate_remote_call: waiting for 1.5s before originate_external', @key, call.key
        await sleep 1500-100+200*Math.random()

        unless 'presenting' is await @state()
          debug 'Agent.originate_remote_call: state changed while waiting', @key, call.key
          return false

For a dial-out (egress) call we attempt to contact the destination.
For a dial-in (ingress) call we already have the proper call-id.

        unless await call.originate_external()
          debug 'Agent.originate_remote_call: originate-external failed', @key, call.key
          await @transition 'failed', {call}
          return false

        await @set_remote_call call
        return true

Connect a remote call
---------------------

      connect_remote_call: (call) ->
        debug 'Agent.connect_remote_call', @key, call.key

        {reason} = agent_call = await @originate_to_agent()

        debug 'Agent.connect_remote_call', @key, call.key, agent_call?.key, reason

        if reason?
          unless call.broadcasting
            await @incr_missed()
            await @transition 'missed', {call,reason}
          return false

        debug 'Agent.connect_remote_call: bridge', @key, call.key, agent_call.key

        await agent_call.transition 'track'

        unless await call.bridge agent_call
          await agent_call.hangup()
          await @transition 'failed', {call}
          return false

        debug 'Agent.connect_remote_call: Successfully bridged', @key, call.key, agent_call.key
        await call.set_remote_agent @key
        await @transition 'answer', {call}


Park an agent, indicating end-of-call + end-of-wrapup
-----------------------------------------------------

      park: ->
        debug 'Agent.park', @key

Actually park an off-hook agent.

        agent_call = await @get_offhook_call()
        if agent_call?
          await agent_call.park()

On-hook agents don't need to be parked, they should hangup.

        else
          true

Notify start of wrapup time to an agent
---------------------------------------

      wrapup: ->
        debug 'Agent.wrapup', @key

        agent_call = await @get_offhook_call()
        agent_call ?= await @get_onhook_call()
        if agent_call?
          await agent_call.wrapup()

      disconnect_remote: ->
        debug 'Agent.disconnect_remote', @key
        current_call = await @get_remote_call()
        if current_call?
          await current_call.hangup()
          await current_call.set_remote_agent null

Tools
-----

      reset_missed: ->
        @reset 'missed'

      incr_missed: ->
        @incr 'missed'

      get_missed: ->
        v = await @get 'missed'
        if v?
          parseInt v, 10
        else
          0

      get_call: (name) ->
        debug 'get_call', @key, name
        key = await @get name
        if key?
          call = new @Call key
          await call.set_domain @domain
          call
        else
          null

      set_call: (name,call) ->
        debug 'set_call', @key, name, call?.key
        if call?
          await @set name, call.key

A `designated` call should never be in the list of active (connected) calls.

          await @remove call.key
        else
          await @set name, null

      is_call: (name,call) ->
        debug 'is_call', @key, name, call?.key
        if call?
          key = await @get name
          key? and call.key is key
        else
          null

      get_offhook_call: ->
        @get_call 'offhook-call'

      set_offhook_call: (offhook_call) ->
        @set_call 'offhook-call', offhook_call

      is_offhook_call: (call) ->
        @is_call 'offhook-call', call

      get_onhook_call: ->
        @get_call 'onhook-call'

      set_onhook_call: (onhook_call) ->
        @set_call 'onhook-call', onhook_call

      is_onhook_call: (call) ->
        @is_call 'onhook-call', call

The remote-call should be a call leg, actively managed by the queuer, interesting this agent.
It should _not_ be included in the list of connected calls outside the queuer (which is managed using `@on_bridge`, `@on_unbridge`, `@on_hangup`).

      get_remote_call: ->
        @get_call 'remote-call'

      set_remote_call: (remote_call)->
        @set_call 'remote-call', remote_call

      is_remote_call: (call) ->
        @is_call 'remote-call', call

    module.exports = Agent
