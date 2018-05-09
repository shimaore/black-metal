Agent
=====

    {debug,heal} = (require 'tangible') 'black-metal:agent'
    RedisClient = require 'normal-key/client'

    Solid = require 'solid-gun'
    make_id = ->
      Solid.time() + Solid.uniqueness()

    transition = require './agent-state-machine'

    class Agent extends RedisClient

      queuer: null # must be defined

      constructor: (key) ->
        debug 'new Agent', key
        throw new Error 'Agent requires key' unless key?
        [number,domain] = key.split '@'
        throw new Error "Agent requires number: #{key}" unless number?
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
        return unless call?

        if await @is_onhook_call call
          debug 'Agent.on_bridge: onhook agent connected (ignored)', @key, call.key
          return

        if await @is_offhook_call call
          debug 'Agent.on_bridge: offhook agent connected (ignored)', @key, call.key
          return

        if await @is_remote_call call
          debug.dev 'Agent.on_bridge: Error: attempting to add the (queuer-managed) remote-call', @key, call.key
          return

        added = await @add call.key

        if added
          debug 'Agent.on_bridge: added new call', @key, call.key
          await @transition 'bridge', {call}
        else
          debug 'Agent.on_bridge: Error: call was already present', @key, call.key
        null

### on-hangup / on-unbridge

Remove a call-leg from the list of connected call-legs.

Transfer-disposition values:
- `recv_replace`: we transfered the call out (blind transfer). (REFER To)
- `replaced`: we accepted an inbound, supervised-transfer call. (Attended Transfer on originating session.)
- `bridge`: we transfered the call out (supervised transfer).

      on_hangup: (call,disposition) ->
        debug 'Agent.on_hangup', @key, call.key, disposition
        # FIXME: this should only be forwarded to `on_unbridge` for calls that are not bridged (i.e. hangup during progress etc.).
        await @on_unbridge call, disposition

      on_unbridge: (call,disposition) ->
        debug 'Agent.on_unbridge', @key, call.key, disposition

        removed = await @remove call.key

        if await @is_offhook_call call
          debug 'Agent.on_unbridge (for offhook agent call)', @key, call.key, disposition
          await @set_offhook_call null

          switch disposition
            when 'recv_replace', 'bridge'
              if await @transition 'agent_transfer', {call}
                await @clear_call call
              # await agent_call.transition 'transferred'
            when 'replaced'
              # await agent_call.transition 'transferred'
              yes
            else
              if await @transition 'agent_hangup', {call}
                await @disconnect_remote()
              # await agent_call.transition 'hungup'

          await @transition 'hangup'

          await @transition 'logout'
          return

        if await @is_remote_call call
          debug 'Agent.on_unbridge (for remote call)', @key, call.key, disposition
          await @clear_call remote_call
          if disposition isnt 'replaced'
            await @transition 'hangup', {call}
          return

        if await @is_onhook_call call
          debug 'Agent.on_unbridge (for onhook agent call)', @key, call.key, disposition
          await @set_onhook_call null

          switch disposition
            when 'recv_replace', 'bridge'
              if await @transition 'agent_transfer', {call}
                await @clear_call call
              await agent_call.transition 'transferred'
            when 'replaced'
              await agent_call.transition 'transferred'
            else
              if await @transition 'agent_hangup', {call}
                await @disconnect_remote()
              await agent_call.transition 'hungup'

          await @transition 'hangup'
          return

        count = await @count()

        if removed and count is 0
          debug 'Agent.on_unbridge: last call was removed', @key, count, call.key, disposition
          await @transition 'end_of_calls'
        else
          debug 'Agent.on_unbridge: calls left', @key, count, call.key, disposition
          await @transition 'unbridge', {call}
        null

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
        debug 'get_call', name
        key = await @get name
        if key?
          call = new @Call key
          await call.set_domain @domain
          call
        else
          null

      set_call: (name,call) ->
        debug 'set_call', name
        if call?
          await @set name, call.key
        else
          await @set name, null

      is_call: (name,call) ->
        debug 'is_call', name
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
        @is_call 'onhook-call'

The remote-call should be a call leg, actively managed by the queuer, interesting this agent.
It should _not_ be included in the list of connected calls outside the queuer (which is managed using `@add_call`, `@del_call`).

      get_remote_call: ->
        @get_call 'remote-call'

      set_remote_call: (remote_call)->
        @set_call 'remote-call', remote_call

      is_remote_call: (call) ->
        @is_call 'remote-call', call

    module.exports = Agent
