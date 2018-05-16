Agent
=====

    {debug,heal} = (require 'tangible') 'black-metal:agent'
    RedisClient = require 'normal-key/client'

    Solid = require 'solid-gun'
    make_id = ->
      Solid.time() + Solid.uniqueness()

    transition = require './agent-state-machine'

Transfer-disposition values:
- `recv_replace`: we transfered the call out (blind transfer). (REFER To)
- `replaced`: we accepted an inbound, supervised-transfer call. (Attended Transfer on originating session.)
- `bridge`: we transfered the call out (supervised transfer).

    BLIND_TRANSFER = 'recv_replace'
    SUPERVISED_TRANSFER = 'bridge'
    ACCEPT_SUPERVISED_TRANSFER = 'replaced'

    class Agent extends RedisClient

      # queuer: null # must be defined

      constructor: (key) ->
        debug 'new Agent', key
        # istanbul ignore next
        throw new Error 'Agent requires key' unless key?
        [number,domain] = key.split '@'
        # istanbul ignore next
        throw new Error "Agent requires number: #{key}" unless number?
        # istanbul ignore next
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

      on_bridge: (call,disposition) ->
        debug 'Agent.on_bridge', @key, call.key, disposition

        switch
          when await @is_remote_call call
            debug.dev 'Agent.on_bridge: queuer-managed remote-call connected (ignored)', @key, call.key

          when await @is_onhook_call call
            debug 'Agent.on_bridge: onhook agent connected (ignored)', @key, call.key

          when await @is_offhook_call call
            debug 'Agent.on_bridge: offhook agent connected (ignored)', @key, call.key
            return

          else
            added = await @add call.key

            if added
              debug 'Agent.on_bridge: added new call', @key, call.key
              await @transition 'bridge', {call}
            else
              debug.dev 'Agent.on_bridge: Error: call was already present', @key, call.key

        null

### on-unbridge

Unbridge might happens because of transfers or because of hang-up.

Transfer-disposition values:
- `recv_replace`: we transfered the call out (blind transfer). (REFER To)
- `replaced`: we accepted an inbound, supervised-transfer call. (Attended Transfer on originating session.)
- `bridge`: we transfered the call out (supervised transfer).

      on_unbridge: (call,disposition) ->
        debug 'Agent.on_unbridge', @key, call.key, disposition

Remove a call-leg from the list of connected call-legs.

        removed = await @remove call.key

        switch

          when await @is_remote_call call
            debug 'Agent.on_unbridge: queuer-manager remote call disconnected', @key, call.key, disposition
            await @clear_call call
            if disposition isnt ACCEPT_SUPERVISED_TRANSFER
              await @transition 'hangup', {call}

          when await @is_onhook_call call
            debug 'Agent.on_unbridge: on-hook agent disconnected', @key, call.key, disposition

            switch disposition
              when BLIND_TRANSFER, SUPERVISED_TRANSFER
                if await @transition 'agent_transfer', {call}
                  await @clear_call call
                await call.transition 'transferred'
              when ACCEPT_SUPERVISED_TRANSFER
                await call.transition 'transferred'
              else
                await call.transition 'hungup'

          when await @is_offhook_call call
            debug 'Agent.on_unbridge: off-hook agent disconnected', @key, call.key, disposition

            switch disposition
              when BLIND_TRANSFER, SUPERVISED_TRANSFER
                if await @transition 'agent_transfer', {call}
                  await @clear_call call
                # await call.transition 'transferred'
              when ACCEPT_SUPERVISED_TRANSFER
                # await call.transition 'transferred'
                yes
              else
                # await call.transition 'hungup'
                no

All calls are assumed to be "other calls".

          else
            debug 'Agent.on_unbridge: other call disconnected', @key, call.key, disposition

            count = await @count()

            if removed and count is 0
              debug 'Agent.on_unbridge: last call was removed', @key, count, call.key, disposition
              await @transition 'end_of_calls'
            else
              debug 'Agent.on_unbridge: calls left', @key, count, call.key, disposition
              await @transition 'unbridge', {call}

        return

### on-hangup

Transfer-disposition values:
- `recv_replace`: we transfered the call out (blind transfer). (REFER To)
- `replaced`: we accepted an inbound, supervised-transfer call. (Attended Transfer on originating session.)
- `bridge`: we transfered the call out (supervised transfer).

Note that `hangup` may happen in two cases:
- the call is terminated before it ever gets connected (there are no bridge/unbridge events);
- the call is terminated after it gets connected (there might have been multiple bridge/unbridge events).

      on_hangup: (call,disposition) ->
        debug 'Agent.on_hangup', @key, call.key, disposition

        removed = await @remove call.key

        switch

Remote call was hung up

          when await @is_remote_call call
            debug 'Agent.on_hangup: queuer-managed remote call hang up', @key, call.key, disposition
            await @clear_call call
            if disposition isnt ACCEPT_SUPERVISED_TRANSFER
              await @transition 'hangup', {call}

Onhook agent call was hung up

          when await @is_onhook_call call
            debug 'Agent.on_hangup: on-hook agent call hung up', @key, call.key, disposition
            await @set_onhook_call null

            switch disposition
              when BLIND_TRANSFER, SUPERVISED_TRANSFER
                no
              when ACCEPT_SUPERVISED_TRANSFER
                no
              else
                if await @transition 'agent_hangup', {call}
                  await @disconnect_remote()
            await @transition 'hangup'

Offhook agent call was hung up

          when await @is_offhook_call call
            debug 'Agent.on_hangup: off-hook agent call hung up', @key, call.key, disposition
            await @set_offhook_call null
            switch disposition
              when BLIND_TRANSFER, SUPERVISED_TRANSFER
                no
              when ACCEPT_SUPERVISED_TRANSFER
                no
              else
                if await @transition 'agent_hangup', {call}
                  await @disconnect_remote()
            await @transition 'hangup'
            await @transition 'logout'

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

      on_dmtf: (digit) ->
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
        await @on_unbridge call, 'accept_offhook'
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
          return offhook_call

For on-hook we need to call the agent.

        onhook_call = await @get_onhook_call()
        if onhook_call?
          throw new Error 'In originate_to_agent, agent already has an on-hook call'
          return

        agent_call = new @Call make_id()
        await agent_call.set_domain @domain
        await agent_call.set_destination @key
        await agent_call.set_local_agent @key
        remote = await @get_remote_call()
        reason = await agent_call.originate_internal remote
        if reason?
          return {reason}

        await @set_onhook_call agent_call
        agent_call

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
          await @clear_call current_call

      clear_call: (remote_call) ->
        return unless remote_call?
        await remote_call.set_remote_agent null
        await @set_remote_call null

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
