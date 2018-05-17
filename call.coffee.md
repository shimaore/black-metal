class Call: a call from or towards a customer or agent
=====

    {debug,foot,heal} = (require 'tangible') 'black-metal:call'

Keep these under the shortest state-machine timer, currently 59s.

    progress_timeout = 18
    internal_timeout = 22
    external_timeout = 30

    make_params = (data) ->
      ("#{k}=#{v}" for own k,v of data).join ','

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

QueuerCall
----------

    Call = require 'screeching-eggs/call'
    transition = require './call-state-machine'

    class QueuerCall extends Call

The following field MUST be provided by an implementation class.
It should implement features similar to the ones found in the `api` object of `huge-play/middleware/setup`.
(Currently we rely on `api()` and `api.truthy()`.)

      __api: null
      queuer: null
      Reference: null

      api: (args...) -> @__api.truthy args...

      set_destination: (destination) ->
        debug 'set_destination', @key, destination
        @set 'destination', destination
      get_destination: -> @get 'destination'
      set_id: (id) ->
        debug 'set_id', @key, id
        @set 'id', id
      get_id: -> @get 'id'

      exists: ->
        api_id = await @get_id()
        if api_id?
          if await @api "uuid_exists #{api_id}"
            true
          else
            await @transition 'miss'
            false
        else
          null

      notify: ->

Handle transitions
------------------

      transition: transition

      post_process: (notification_data) -> @queuer.on_call this, notification_data

Originate a call towards an agent
---------------------------------

Make sure to define the `api` method` and the `profile` member.

The destination endpoint is stored as the `destination` field.

The `caller` object is the remote Call object.

Returns a string indicating the error in case of failure; null in case of success.

      originate_internal: (caller) ->
        destination = await @get_destination()
        debug 'QueuerCall.originate_internal', @key, destination

The `reference` is set either:
- by huge-play/middleware/client/fifo for an ingress call;
- by originate_external (below) for a successful egress call.

        reference = await caller.get_reference()

        unless reference?
          debug.dev 'QueuerCall.originate_internal: caller has no reference', caller.key, @key, destination
          return 'NO REFERENCE'

        source = await caller.get_remote_number()
        source ?= 'caller'

Agent in off-hook mode

        api_id = await @get_id()
        if api_id?
          if await @exists()
            return null
          else
            return 'DOES NOT EXIST'

Agent in on-hook mode

        id = @key

        alert_info = await caller.get_alert_info()

        endpoint = destination

        xref = "xref:#{reference}"

        my_reference = new @Reference reference
        await my_reference.set_endpoint endpoint
        await my_reference.add_in endpoint

        music = await caller.get_music()

        params =
          origination_uuid: id
          origination_caller_id_number: source
          hangup_after_bridge: false
          park_after_bridge: true
          progress_timeout: progress_timeout
          originate_timeout: internal_timeout
          'sip_h_X-En': endpoint # Centrex-only
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

        await caller.add id

        body = await @__api "originate {#{params}}sofia/#{@profile}/#{destination} &park"

Typically the body might contain:
- `+OK <uuid>\n`
- `-ERR <reason>\n`
where reason might be:
- `PROGRESS_TIMEOUT` - phone rang but noone answered
- `NO_USER_RESPONSE` - phone is on DND
- `RECOVERY_ON_TIMER_EXPIRE` - phone is unreachable
- etc.

        debug 'originate_internal: originate returned', @key, id, body
        connected = body?[0] is '+'

        if connected
          await @set_destination null
          await @set_id id
          await @set_reference reference
          return null
        else
          reason = body?.substr(5).trimRight 1
          reason ?= 'FAILED'
          await caller.remove id
          return reason

Originate a call towards a third-party
--------------------------------------

The `reference` ID is stored as the `destination` field.

      originate_external: ->
        destination = await @get_destination()
        debug 'QueuerCall.originate_external', @key, destination

Ingress (or otherwise existing) call

        api_id = @get_id()
        if api_id?
          await @exists()
          return

Egress call

        id = @key

This is similar to what we do with `place-call` but we're calling the other way around. The `estination` consists of the reference, and we're creating a brand new call which emulates a call from the endpoint.

        reference = destination

        my_reference = new @Reference reference
        destination = await my_reference.get_destination()
        domain = await my_reference.get_domain()
        source = await my_reference.get_source()

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

        if await @api "originate {#{params}}sofia/#{@profile}/#{destination}@#{domain} &park"
          await @set_destination null
          await @set_id id
          await @set_domain domain
          await @set_reference reference
          this
        else
          await @transition 'fail'
          return

      bridge: (agent_call) ->
        agent_call_id = await agent_call.get_id()
        debug 'QueuerCall.bridge', @key, agent_call_id

        ### istanbul ignore next ###
        throw new Error "QueuerCall.bridge #{@key} (#{agent_call.key}) misses agent_call id" unless agent_call_id?

        await @api "uuid_break #{@key}"
        await @api "uuid_broadcast #{agent_call_id} gentones::%(100,20,400);%(100,0,600) aleg"
        await sleep 400
        await @api "uuid_bridge #{@key} #{agent_call_id}"

Remove all the matched calls, except maybe one.

      unbridge_except: (except = null) ->
        debug 'Call.unbridge_except', @key, except
        self = this
        await @forEach foot (id) ->
          return if id is except
          await self.api("uuid_kill #{id}").catch -> yes
          await self.remove id

      park: ->
        debug 'Call.park', @key
        # result = await @api "uuid_park #{@key}"

Actually there is no need to park the call for real, this only creates issues
with the gentones notifications.

        result = true
        await sleep 100
        await @api "uuid_broadcast #{@key} gentones::%(100,20,400);%(100,0,400) aleg"
        await sleep 400
        result

      wrapup: ->
        debug 'Call.wrapup', @key
        await sleep 400
        await @api "uuid_broadcast #{@key} gentones::%(100,20,600);%(100,0,400) aleg"
        await sleep 400

      hangup: ->
        debug 'Call.hangup', @key
        api_id = await @get_id()
        if api_id?
          await @api("uuid_kill #{api_id}").catch -> yes
        await @set_id null
        await @set_destination null
        await @transition 'hangup'

      announce: (file) ->
        debug 'Call.announce', @key, file
        await @api "uuid_broadcast #{@key} #{file}"

      presenting: ->
        count = await @count()
        count > 0

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

      set_domain: (domain) ->
        @set 'domain', domain

      get_domain: ->
        @get 'domain'

      set_session: (session) ->
        @set 'session', session

      get_session: ->
        @get 'session'

      set_music: (uri) ->
        @set 'music', uri

      get_music: ->
        @get 'music'

Local-Agent: the agent attached to this call leg. Can only be set, once.

      set_local_agent: (agent_key) ->
        debug 'Call.set_local_agent', @key, agent_key
        unless agent_key?
          debug.dev 'Error: Can only set (not deleted) the local-agent', @key, agent_key
          return

        current = await @get_local_agent()
        return if agent_key is current

        if current?
          debug.dev 'Error: Can only set local-agent once', @key, current, agent_key
          return

        await @set 'local-agent', agent_key
        return

      get_local_agent: ->
        @get 'local-agent'

Remote-Agent: the agent attached to another call leg, presumably bridged to this one.

      set_remote_agent: (agent_key) ->
        debug 'Call.set_remote_agent', @key, agent_key

        if agent_key?
          reference = await @get_reference()
          my_reference = new @Reference reference
          await my_reference.set_endpoint agent_key
          await my_reference.add_in agent_key

        await @set 'remote-agent', agent_key
        return

      get_remote_agent: ->
        @get 'remote-agent'

    module.exports = QueuerCall
