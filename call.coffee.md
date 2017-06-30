class Call: a call from or towards a customer

    @name = 'black-metal:call'
    debug = (require 'tangible') @name
    seem = require 'seem'
    uuidV4 = require 'uuid/v4'
    RedisClient = require './redis'

    make_id = ->
      now = new Date() .toJSON()
      now[0...8] + uuidV4()

    make_params = (data) ->
      ("#{k}=#{v}" for own k,v of data).join ','

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

    class Call extends RedisClient

      constructor: ({@destination,@id,key}) ->

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
          yield @api "uuid_exists #{@id}"
        else
          null

Originate a call towards an agent
---------------------------------

Make sure to define the `api` method` and the `profile` member.

The destination endpoint is stored as the `@destination` field.

      originate_internal: seem (caller) ->
        debug 'Call.originate', @key, @destination

The `reference` is set either:
- by huge-play/middleware/client/fifo for an ingress call;
- by originate_external (below) for a successful egress call.

        reference = yield caller.get_reference()

        unless reference?
          debug.dev 'Call.originate: caller has no reference', caller.key, @key, @destination
          return null

        data = yield caller.get_reference_data reference

        source = yield caller.get_remote_number()
        source ?= 'caller'

Agent in off-hook mode

        if @id?
          if yield @exists()

FIXME how do we (re)bind the agent's off-hook call with this incoming call?

            return this
          else
            return null

Agent in on-hook mode

        id = @key

        alert_info = yield caller.get_alert_info()

        endpoint = @destination
        data.endpoint = endpoint
        data._in ?= []
        data._in.push endpoint unless endpoint in data._in
        data.state = 'originate-internal'
        data = yield @update_reference_data data

        music = yield caller.get_music()

        xref = "xref:#{reference}"
        params =
          origination_uuid: id
          origination_caller_id_number: source
          hangup_after_bridge: false
          park_after_bridge: true
          progress_timeout: 18
          originate_timeout: 22
          'sip_h_X-CCNQ3-Endpoint': @destination # Centrex-only
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
          if yield @exists()
            return this
          else
            return 'missing'

Egress call

        id = @key

This is similar to what we do with `place-call` but we're calling the other way around. The `destination` consists of the reference, and we're creating a brand new call which emulates a call from the endpoint.

        reference = @destination

        data = yield @get_reference_data reference

        unless data?
          return 'missing'

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

        for own k,v of data.params
          params[k] ?= v

        params = make_params params

        if yield @api "originate {#{params}}sofia/#{@profile}/#{data.destination}@#{data.domain} &park"
          @destination = null
          @id = id
          yield @save()
          yield @set_reference reference
          this
        else
          'failed'

      bridge: seem (agent_call) ->
        debug 'Call.bridge', @id, agent_call.id
        yield @api "uuid_break #{@id}"
        yield @api "uuid_broadcast #{agent_call.id} gentones::%(100,20,400);%(100,0,600) aleg"
        yield sleep 400
        yield @api "uuid_bridge #{@id} #{agent_call.id}"

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

      monitor: seem (events...) ->
        debug 'Call.monitor', @id
        client = yield @api "myevents #{@id} json"
        yield client.event_json events...
        client

      announce: seem (file) ->
        debug 'Call.announce', @id, file
        yield @api "uuid_broadcast #{@id} #{file}"

      presenting: seem ->
        count = yield @count()
        count > 0

      bridged: seem ->
        yield @has_tag 'bridged'

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

    module.exports = Call
