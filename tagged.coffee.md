Tagging in the queuer
=====================

    debug = (require 'tangible') 'black-metal:tagged-call'
    Bluebird = require 'bluebird'

Policy
------

The policy is:

    policy = (calls) ->
      agent_state = await @state()
      agent_broadcast = await @has_tag 'broadcast'
      agent_key = @key
      agent_has_tag = (tag) => @has_tag tag

      filtered = await Bluebird.filter calls, (call) ->

        debug 'policy:filtered: Checking call for agent', call.key, agent_key

        call_state = await call.state()

        switch call_state
          when 'pooled'
            call_broadcast = await call.broadcast()
            call.broadcasting = call_broadcast
            yes

          when 'handled'

Do not make parallel attempts for calls that have no id (outbound calls).

            call_id = await call.get_id()
            unless call_id?
              return false

            call_broadcast = await call.broadcast()
            unless call_broadcast or agent_broadcast
              debug 'policy:filtered: Call is already being handled', call.key, agent_key
              return false

            call.broadcasting = true
          else
            return false

        call.waiting = if call_state is 'handled' then 0 else 1

        call_tags = new Set await call.tags()

- domains must match (this is now ensured by the call pools)

- if at least one queue is listed, the agent must accept calls from (one of) these queue(s)

        call_queues = queues call_tags
        debug 'policy:filtered: Checking agent for queues', call.key, agent_key, call_queues
        if call_queues.length > 0
          ok = false
          for queue_tag in call_queues when not ok
            if await agent_has_tag queue_tag
              ok = true
          unless ok
            debug 'policy:filtered: No queues match', call.key, agent_key, call_queues
            return false

- the agent must have all the skills listed for the call

        call_required_skills = skills call_tags
        debug 'policy:filtered: Checking agent for skills', call.key, agent_key, call_required_skills
        for skill_tag in call_required_skills
          unless await agent_has_tag skill_tag
            debug 'policy:filtered: No skill match', call.key, agent_key, skill_tag
            return false

        call.priority = priority call_tags
        debug 'policy:filtered: Call priority assigned', call.key, call.priority

        return true

      sorted = filtered.sort (a,b) ->

        a_prio = a.priority
        b_prio = b.priority
        waiting = a.waiting - b.waiting
        fifo = a.started_at - b.started_at

- sort higher priority calls first
- calls which are not being presented yet get preference
- equal priority calls are presented first-in first-out

        if not a_prio? and not b_prio?
          return fifo
        if a_prio? and not b_prio?
          return 1
        if b_prio? and not a_prio?
          return -1
        (a_prio - b_prio) or waiting or fifo

      policed_call = sorted[0]
      debug 'policy selected call', policed_call
      policed_call

Keep the highest priority value

    priority = (tags) ->
      debug 'priority'
      result = []
      tags.forEach (tag) ->
        if m = tag.match /^priority:(\d+)$/
          result.push parseInt m[1], 10
      priorities = result.sort (a,b) ->
        switch
          when not a?
            1
          when not b?
            -1
          else
            b-a
      debug 'priority', priorities[0]
      priorities[0] ? null

    skills = (tags) ->
      debug 'skills'
      result = []
      tags.forEach (tag) -> result.push tag if tag.match /^skill:/
      debug 'skills', result
      result

    queues = (tags) ->
      debug 'queues'
      result = []
      tags.forEach (tag) -> result.push tag if tag.match /^queue:/
      debug 'queues', result
      result

Tagged Call
-----------

See queue-based tagging in huge-play/middleware/client/fifo for an example.

Other tags might be added by the application (for example to add caller-based tags).

    QueuerCall = require './call'

    class TaggedCall extends QueuerCall

      constructor: (key) ->
        super key
        debug 'new TaggedCall'
        return

      set_tags: (tags) ->
        await @clear_tags()
        await @add_tags tags

      set_started_at: ->
        @set 'started-at', new Date().getTime()

      get_started_at: ->
        @get 'started-at'

Tagged Agent
------------

    Agent = require './agent'

    class TaggedAgent extends Agent
      policy: policy

    module.exports = {policy,TaggedCall,TaggedAgent}
