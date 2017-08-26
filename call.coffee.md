class Call: a call from or towards a customer
=====

    @name = 'black-metal:call'
    debug = (require 'tangible') @name
    seem = require 'seem'
    heal = (p) -> p.catch debug.catch
    hand = (f) ->
      F = seem f
      (args...) -> heal F args...
    Solid = require 'solid-gun'
    RedisClient = require 'normal-key/client'

    seconds = 1000
    minutes = 60*seconds
    timeout_duration = 1*minutes

    make_id = ->
      Solid.time() + Solid.uniqueness()

    make_params = (data) ->
      ("#{k}=#{v}" for own k,v of data).join ','

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

    class Call extends RedisClient

The following two methods MUST be provided by an implementation class:

      api: (cmd) -> throw new Error 'Call.api is not implemented.'
      monitor_api: (id,events) -> throw new Error 'Call.monitor_api is not implemented'

      constructor: (@queuer,{@destination,@id,key}) ->
        throw new Error 'Call requires queuer' unless @queuer?

Load an existing call profile back.

        if key?
          debug 'new Call', key
          super 'call', key

Create a new profile

        else
          debug 'new Call', @destination, @id
          super 'call', @id ? @make_id()
          unless @destination? or @id?
            throw new Error 'new Call: either destination or id is required'

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

      make_id: ->
        make_id()

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

        count = 10
        while count-- > 0
          before = (yield @incr 'lock', offset)?[0]?[0]?[1]
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

        if @__timeout?
          clearTimeout @__timeout
          @__timeout = null

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
          if 'timeout' of _transition[new_state]
            @__timeout = setTimeout (=> @transition 'timeout'), timeout_duration

          process.nextTick => heal @queuer.on_call this, notification_data

          return true
        else
          return false

Originate a call towards an agent
---------------------------------

Make sure to define the `api` method` and the `profile` member.

The destination endpoint is stored as the `@destination` field.

The `caller` object is the remote Call object.

      originate_internal: seem (caller) ->
        debug 'Call.originate_internal', @key, @destination

The `reference` is set either:
- by huge-play/middleware/client/fifo for an ingress call;
- by originate_external (below) for a successful egress call.

        reference = yield caller.get_reference()

        unless reference?
          debug.dev 'Call.originate_internal: caller has no reference', caller.key, @key, @destination
          return null

        source = yield caller.get_remote_number()
        source ?= 'caller'

Agent in off-hook mode

        if @id?
          if yield @exists()
            return this
          else
            return null

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
          progress_timeout: 18
          originate_timeout: 22
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

        if yield @api "originate {#{params}}sofia/#{@profile}/#{@destination} &park"
          @destination = null
          @id = id
          yield @save()
          yield @set_reference reference
          this
        else
          yield caller.remove id
          null

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
          progress_timeout: 18
          originate_timeout: 30
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
          yield @call.transition 'fail'
          return

      bridge: seem (agent_call) ->
        debug 'Call.bridge', @id, agent_call.id
        yield @api "uuid_break #{@id}"
        yield @api "uuid_broadcast #{agent_call.id} gentones::%(100,20,400);%(100,0,600) aleg"
        yield sleep 400
        if yield @api "uuid_bridge #{@id} #{agent_call.id}"
          yield @transition 'bridge', {agent_call}
        else
          false

Remove all the matched calls, except maybe one.

      unbridge_except: seem (except = null) ->
        debug 'Call.unbridge_except', @id, except
        yield @forEach (id) =>
          return if id is except
          @api("uuid_kill #{id}").catch -> yes
        yield @clear()

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
        yield @api "uuid_broadcast #{@id} gentones::%(100,20,600);%(100,0,400) aleg"
        yield sleep 400

      hangup: seem ->
        debug 'Call.hangup', @id
        if @id?
          yield @api("uuid_kill #{@id}").catch -> yes
        @id = null
        @destination = null
        yield @save()
        yield @transition 'hangup'

      monitor: (events...) ->
        debug 'Call.monitor', @id
        return null unless @id?
        @monitor_api @id, events

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

      bridged: seem ->
        'bridged' is yield @state()

      broadcast: ->
        @has_tag 'broadcast'

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

Call Transitions
----------------

    initial_state = 'new'

    _transition =

If a call is transitioned back to `new` it means it got forgotten.

      new:
        hangup: 'dropped' # hungup locally
        hungup: 'dropped' # hungup by remote end
        miss: 'dropped' # disappeared from system
        pool: 'pooled'
        timeout: 'new' # forgotten

Only pooled calls actually get considered.

      pooled:
        broadcast: 'handled'
        handle: 'handled'
        hangup: 'dropped' # hungup locally
        hungup: 'dropped' # hungup by remote end
        miss: 'dropped' # disappeared from system
        timeout: 'new'
        unpool: 'new'

      handled:
        bridge: 'bridged'
        broadcast: 'handled'
        fail: 'dropped'
        hangup: 'dropped' # hungup locally
        hungup: 'dropped' # hungup by remote end
        miss: 'dropped' # disappeared from system
        pool: 'pooled' # force re-try
        timeout: 'pooled' # force re-try

      bridged:
        hangup: 'dropped' # hungup locally
        hungup: 'dropped' # hungup by remote end
        miss: 'dropped' # disappeared from system

      dropped: {}

    module.exports = Call
