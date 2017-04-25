class Call: a call from or towards a customer

    @name = 'black-metal:call'
    debug = (require 'debug') @name
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

and save it (async).

          @save()

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
          'true' is yield @api "uuid_exists #{@id}"
        else
          null

Make sure to define the `api` method` and the `profile` member.

      originate_internal: seem (caller) ->
        debug 'Call.originate', @key, @destination

Agent in off-hook mode

        if @id?
          if yield @exists()
            return this
          else
            return null

Agent in on-hook mode

        id = @key
        source = yield caller.get_remote_number()
        source ?= 'caller'
        params = make_params
          origination_uuid: id
          origination_caller_id_number: source
          hangup_after_bridge: false
          park_after_bridge: true
          'sip_h_X-CCNQ3-Endpoint': @destination # Centrex-only

        if yield @api "originate {#{params}}sofia/#{@profile}/#{@destination} &park"
          @destination = null
          @id = id
          yield @save()
          this
        else
          null

      originate_external: seem ->
        debug 'Call.originate_external', @key, @destination

Ingress (or otherwise existing) call

        if @id?
          if yield @exists()
            return this
          else
            return null

Egress call

        id = @key

This is similar to what we do with `place-call` but we're calling the other way around. The `destination` consists of the reference, and we're creating a brand new call which emulates a call from the endpoint.

        reference = @destination

        data = yield @get_reference_data reference

        unless data?
          return null

        data.params.origination_uuid = id
        data.params.hangup_after_bridge = false
        data.params.park_after_bridge = true

        params = make_params data.params

        if yield @api "originate {#{params}}sofia/#{@profile}/#{data.destination}@#{data.domain} &park"
          @destination = null
          @id = id
          yield @save()
          this
        else
          null

      bridge: seem (agent_call) ->
        debug 'Call.bridge', @id, agent_call.id
        yield @api "uuid_broadcast #{agent_call.id} gentones::%(100,20,400);%(100,0,600) aleg"
        yield sleep 400
        yield @api "uuid_bridge #{@id} #{agent_call.id}"

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
        yield @api "uuid_kill #{@id}"
        @id = null
        @destination = null
        yield @save()

      monitor: seem ->
        debug 'Call.monitor', @id
        yield @api "myevents #{@id} json"

      presenting: seem ->
        yield @has_tag 'presenting'

      set_remote_number: (number) ->
        @set 'remote-number', number

      get_remote_number: ->
        @get 'remote-number'

Present
-------

Present the current call to the given agent.
Returns:
- true if success
- false if failure due to the agent (no response)
- null if failure due to other element.

      present: seem (agent) ->
        debug 'Call.present', agent.key
        @add_tag 'presenting'

        try

For a dial-out (egress) call we first need to attempt to contact the destination.
For a dial-in (ingress) call we already have the proper call UUID.

          yield @originate_external agent

We need to send the call to the agent (using either mode A or mode B).

          agent_call = yield agent.originate this

          if @id? and agent_call? and yield @bridge agent_call
            debug 'Call.present: Successfully bridged', @id, agent_call.key
            return true

          debug 'Call.present: Failed to bridge', @id, agent_call.key
          yield @del_tag 'presenting'
          if not agent_call?
            return false

        catch error
          debug "Call.present: #{error.stack ? error}"
          yield @del_tag 'presenting'

        null

    module.exports = Call
