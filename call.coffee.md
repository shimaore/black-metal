class Call: a call from or towards a customer or agent
=====

    @name = 'black-metal:call'
    {debug,hand,heal} = (require 'tangible') @name
    seem = require 'seem'
    Solid = require 'solid-gun'
    RedisClient = require 'normal-key/client'

    seconds = 1000
    minutes = 60*seconds

Keep these under the shortest state-machine timer, currently 59s.

    progress_timeout = 18
    internal_timeout = 22
    external_timeout = 30

    make_id = ->
      Solid.time() + Solid.uniqueness()

    make_params = (data) ->
      ("#{k}=#{v}" for own k,v of data).join ','

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

    class Call extends RedisClient

The following field MUST be provided by an implementation class.
It should implement features similar to the ones found in the `api` object of `huge-play/middleware/setup`.
(Currently we rely on `api.truthy`, `api.monitor`, and `api.is_monitored`.)

      __api: null

      api: (args...) -> @__api.truthy args...

      constructor: (queuer,{destination,id,key}) ->

        throw new Error 'Call requires queuer' unless queuer?.is_a_queuer?()

Load an existing call profile back.

        if key?
          debug 'new Call', key
          super 'call', key

Create a new profile

        else
          unless destination? or id?
            throw new Error 'new Call: either destination or id is required'
          debug 'new Call', destination, id
          super 'call', id ? make_id()

        @queuer = queuer
        @destination = destination
        @id = id

        @__bridged_key = "#{@class_name}-#{@key}-Sb"

`new Call` MUST be followed by either `.save()` or `.load()`.

      save: seem ->
        debug 'Call.save', @key
        yield @set 'destination', @destination
        yield @set 'id', @id
        this

      load: seem ->
        debug 'Call.load', @key
        @destination = yield @get 'destination'
        @id = yield @get 'id'
        this

      exists: seem ->
        if @id?
          if yield @api "uuid_exists #{@id}"
            true
          else
            yield @transition 'miss'
            false
        else
          null

      report: ->

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
        debug 'Call.transition', {call: @key, event}
        yield @load()

        old_state = yield @state()
        old_state ?= initial_state

        debug 'Call.transition', {call: @key, event, old_state}

        unless old_state of _transition
          yield @set_state initial_state
          throw new Error "Invalid state, transition from #{old_state} → event #{event}"

        unless event of _transition[old_state]
          debug "Ignoring event #{event} in state #{old_state}", {call: @key}
          return false

        new_state = _transition[old_state][event]

        @queuer.clear_timer @key

        unless new_state of _transition
          yield @set_state initial_state
          throw new Error "Invalid state machine, transition from #{old_state} → event #{event} leads to unknown state #{new_state}"

        debug 'Call.transition', {call: @key, event, old_state, new_state}
        if new_state?
          yield @set_state new_state

          yield @reset 'lock'

          notification_data.old_state = old_state
          notification_data.state = new_state
          notification_data.event = event

          yield @report? notification_data
          next_state = _transition[new_state]
          if 'timeout' of next_state and 'timeout_duration' of next_state
            on_timeout = =>
              @queuer.clear_timer @key
              heal @transition 'timeout'
            @queuer.set_timer @key, setTimeout on_timeout, next_state.timeout_duration-500+1000*Math.random()

          process.nextTick => heal @queuer.on_call this, notification_data

          return true
        else
          return false

Originate a call towards an agent
---------------------------------

Make sure to define the `api` method` and the `profile` member.

The destination endpoint is stored as the `@destination` field.

The `caller` object is the remote Call object.

Returns a string indicating the error in case of failure; null in case of success.

      originate_internal: seem (caller) ->
        debug 'Call.originate_internal', @key, @destination

The `reference` is set either:
- by huge-play/middleware/client/fifo for an ingress call;
- by originate_external (below) for a successful egress call.

        reference = yield caller.get_reference()

        unless reference?
          debug.dev 'Call.originate_internal: caller has no reference', caller.key, @key, @destination
          return 'NO REFERENCE'

        source = yield caller.get_remote_number()
        source ?= 'caller'

Agent in off-hook mode

        if @id?
          if yield @exists()
            return null
          else
            return 'DOES NOT EXIST'

Agent in on-hook mode

        id = @key

        alert_info = yield caller.get_alert_info()

        endpoint = @destination

        xref = "xref:#{reference}"

        my_reference = new @Reference reference
        yield my_reference.set_endpoint endpoint
        yield my_reference.add_in endpoint

        music = yield caller.get_music()

        params =
          origination_uuid: id
          origination_caller_id_number: source
          hangup_after_bridge: false
          park_after_bridge: true
          progress_timeout: progress_timeout
          originate_timeout: internal_timeout
          'sip_h_X-En': @destination # Centrex-only
          alert_info: alert_info
          sip_invite_params: xref
          sip_invite_to_params: xref
          sip_invite_contact_params: xref
          sip_invite_from_params: xref
          ringback: music
          hold_music: music
        params = make_params params

Originates towards (presumably) OpenSIPS.

Add our call to the matched set on the caller's side so that we can stop the call(s) if the caller hangs up
or (in case of multiple presentations) when someone picks the call up.

        yield caller.add id

        body = yield @__api "originate {#{params}}sofia/#{@profile}/#{@destination} &park"

Typically the body might contain:
- `+OK <uuid>\n`
- `-ERR <reason>\n`
where reason might be:
- `PROGRESS_TIMEOUT` - phone rang but noone answered
- `NO_USER_RESPONSE` - phone is on DND
- `RECOVERY_ON_TIMER_EXPIRE` - phone is unreachable
- etc.

        connected = body?[0] is '+'

        if connected
          @destination = null
          @id = id
          yield @save()
          yield @set_reference reference
          return null
        else
          reason = body?.substr(5).trimRight 1
          reason ?= 'FAILED'
          yield caller.remove id
          return reason

Originate a call towards a third-party
--------------------------------------

The `reference` ID is stored as the `@destination` field.

      originate_external: seem ->
        debug 'Call.originate_external', @key, @destination

Ingress (or otherwise existing) call

        if @id?
          yield @exists()
          return

Egress call

        id = @key

This is similar to what we do with `place-call` but we're calling the other way around. The `destination` consists of the reference, and we're creating a brand new call which emulates a call from the endpoint.

        reference = @destination

        my_reference = new @Reference reference
        destination = yield my_reference.get_destination()
        domain = yield my_reference.get_domain()
        source = yield my_reference.get_source()

        xref = "xref:#{reference}"
        params =
          origination_uuid: id
          hangup_after_bridge: false
          park_after_bridge: true
          progress_timeout: progress_timeout
          originate_timeout: external_timeout
          sip_invite_params: xref
          sip_invite_to_params: xref
          sip_invite_contact_params: xref
          sip_invite_from_params: xref
          origination_caller_id_number: source

        params = make_params params

        if yield @api "originate {#{params}}sofia/#{@profile}/#{destination}@#{domain} &park"
          @destination = null
          @id = id
          yield @save()
          yield @set_reference reference
          this
        else
          yield @transition 'fail'
          return

      bridge: seem (agent_call) ->
        debug 'Call.bridge', @id, agent_call.id
        yield @api "uuid_break #{@id}"
        yield @api "uuid_broadcast #{agent_call.id} gentones::%(100,20,400);%(100,0,600) aleg"
        yield sleep 400
        yield @api "uuid_bridge #{@id} #{agent_call.id}"

Remove all the matched calls, except maybe one.

      unbridge_except: seem (except = null) ->
        debug 'Call.unbridge_except', @id, except
        self = this
        yield @forEach hand (id) ->
          return if id is except
          yield self.api("uuid_kill #{id}").catch -> yes
          yield self.remove id

      park: seem ->
        debug 'Call.park', @id
        # result = yield @api "uuid_park #{@id}"

Actually there is no need to park the call for real, this only creates issues
with the gentones notifications.

        result = true
        yield sleep 100
        yield @api "uuid_broadcast #{@id} gentones::%(100,20,400);%(100,0,400) aleg"
        yield sleep 400
        result

      wrapup: seem ->
        debug 'Call.wrapup', @id
        yield sleep 400
        yield @api "uuid_broadcast #{@id} gentones::%(100,20,600);%(100,0,400) aleg"
        yield sleep 400

      on_bridge: seem (b_call,data) ->
        data ?= call: b_call
        yield @interface.add @__bridged_key, b_call.key
        @transition 'bridge', data

      on_unbridge: seem (b_call,disposition) ->
        data = call: b_call
        yield @interface.remove @__bridged_key, b_call.key
        if 0 is yield @interface.count @__bridged_key
          @transition 'unbridge', data
        else
          @transition 'unbridge_ignore', data

      hangup: seem ->
        debug 'Call.hangup', @id
        if @id?
          yield @api("uuid_kill #{@id}").catch -> yes
        @id = null
        @destination = null
        yield @save()
        yield @transition 'hangup'

      monitor: (events...) ->
        debug 'Call.monitor', @id, events
        return null unless @id?
        @__api.monitor @id, events

      is_monitored: ->
        debug 'Call.is_monitored', @id
        return null unless @id?
        @__api.is_monitored @id

      announce: seem (file) ->
        debug 'Call.announce', @id, file
        yield @api "uuid_broadcast #{@id} #{file}"

      presenting: seem ->
        count = yield @count()
        count > 0

      state: ->
        @get 'state'

      set_state: (state) ->
        @set 'state', state

      answered: ->
        @get 'answered'

      set_answered: ->
        @set 'answered', true

      broadcast: ->
        @has_tag 'broadcast'

      poolable: ->
        @get 'poolable'

      set_poolable: ->
        @set 'poolable', true

      set_remote_number: (number) ->
        @set 'remote-number', number

      get_remote_number: ->
        @get 'remote-number'

      set_alert_info: (alert_info) ->
        @set 'alert-info', alert_info

      get_alert_info: ->
        @get 'alert-info'

      set_reference: (reference) ->
        @set 'reference', reference

      get_reference: ->
        @get 'reference'

      set_session: (session) ->
        @set 'session', session

      get_session: ->
        @get 'session'

      set_music: (uri) ->
        @set 'music', uri

      get_music: ->
        @get 'music'

Local-Agent: the agent attached to this call leg. Can only be set once.

      set_local_agent: seem (agent) ->
        debug 'Call.set_local_agent', @key, agent

        current = yield @get_local_agent()
        return if agent is current

        if current?
          debug.dev 'Error: Can only set local-agent once', @key, current, agent
          return

        yield @set 'local-agent', agent
        return

      get_local_agent: ->
        @get 'local-agent'

Remote-Agent: the agent attached to another call leg, presumably bridged to this one.

      set_remote_agent: seem (agent) ->
        debug 'Call.set_remote_agent', @key, agent

        if agent?
          reference = yield @get_reference()
          my_reference = new @Reference reference
          yield my_reference.set_endpoint agent
          yield my_reference.add_in agent

        yield @set 'remote-agent', agent
        return

      get_remote_agent: ->
        @get 'remote-agent'

Call Transitions
----------------

    initial_state = 'new'

    _transition =

Events:
- fail
- pool
- unpool
- hangup: local call, hungup locally
- transferred: remote call, transferred locally
- hungup: remote call, hungup by remote end
- miss : disappeared from system
- bridge
- unbridge
- unbridge_ignore
- broadcast
- handle

If a call is transitioned back to `new` it means it got forgotten / is in overflow.

      new:
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped' # disappeared from system
        pool: 'pooled'
        handle: 'handled'
        timeout: 'new' # forgotten
        timeout_duration: 97*seconds

Only pooled calls actually get considered.

      pooled:
        broadcast: 'handled'
        handle: 'handled'
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped' # disappeared from system
        timeout: 'new'
        timeout_duration: 31*seconds # overflow/forgotten
        unpool: 'new'

      handled:
        bridge: 'bridged'
        broadcast: 'handled'
        fail: 'dropped'
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped' # disappeared from system
        pool: 'pooled' # force re-try

This might lead to multiple agents ringing even if the `broadcast` option is not active, so we delay it a little.

        timeout: 'pooled' # force re-try
        timeout_duration: 131*seconds

      bridged:
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped' # disappeared from system
        pool: 'pooled'
        bridge: 'bridged' # more than one call bridged (e.g. during transfer)
        unbridge_ignore: 'bridged' # unbridge but other remain (definitely during transfers)
        unbridge: 'unbridged'

      unbridged:
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped' # disappeared from system
        bridge: 'bridged'
        pool: 'pooled'
        timeout: 'pooled' # forgotten
        timeout_duration: 31*seconds

      dropped:
        pool: 'pooled' # on transfer

    module.exports = Call
