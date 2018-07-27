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

          else
            return false

        call.started_at = await call.get_started_at()

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

      sorted = filtered.sort prio_fifo_sort
      policed_call = sorted[0]
      debug 'policy selected call', policed_call
      policed_call

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
        debug 'new TaggedCall', key
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
    prio_fifo_sort = require './prio-fifo-sort'
    priority = require './priority'
