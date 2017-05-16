Tagging in the queuer
=====================

    @name = 'black-metal:tagged-call'
    debug = (require 'tangible') @name
    seem = require 'seem'
    Bluebird = require 'bluebird'

Policy
------

The policy is:

    policy = seem (calls) ->
      filtered = yield Bluebird.filter calls, seem (call) =>

        debug 'Checking call for agent', call.key, @key

        call_tags = new Set yield call.tags()

- domains must match

        unless in_domain call_tags, @domain
          debug 'No domain match', call.key, @key, @domain, call_tags
          return false

- if at least one queue is listed, the agent must accept calls from (one of) these queue(s)

        call_queues = queues call_tags
        debug 'Checking agent for queues', call.key, @key, call_queues
        if call_queues.length > 0
          ok = false
          for queue_tag in call_queues when not ok
            if yield @has_tag queue_tag
              ok = true
          unless ok
            debug 'No queues match', call.key, @key, call_queues
            return false

- the agent must have all the skills listed for the call

        call_required_skills = skills call_tags
        debug 'Checking agent for skills', call.key, @key, call_required_skills
        for skill_tag in call_required_skills
          unless yield @has_tag skill_tag
            debug 'No skill match', call.key, @key, skill_tag
            return false

        call.priority = priority call_tags
        debug 'Call priority assigned', call.key, call.priority

        return true

      sorted = filtered.sort (a,b) ->
        a_prio = a.priority
        b_prio = b.priority
        fifo = a.started_at - b.started_at

- sort higher priority calls first
- equal priority calls are presented first-in first-out

        if not a_prio? and not b_prio?
          return fifo
        if a_prio? and not b_prio?
          return 1
        if b_prio? and not a_prio?
          return -1
        (a_prio - b_prio) or fifo

      sorted[0]

Keep the highest priority value

    priority = (tags) ->
      debug 'priority'
      result = []
      tags.forEach (tag) ->
        if m = tag.match /^priority:(\d+)$/
          result.push parseInt m[1]
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

    in_domain = (tags,domain) ->
      tags.has "number_domain:#{domain}"

Tagged Call
-----------

See queue-based tagging in huge-play/middleware/client/fifo for an example.

Other tags might be added by the application (for example to add caller-based tags).

    Call = require './call'

    class TaggedCall extends Call

      constructor: (opts) ->
        super opts
        {tags = []} = opts
        debug 'new TaggedCall', tags
        @add_tags tags
        @started_at = new Date().getTime()

Tagged Agent
------------

    Agent = require './agent'

    class TaggedAgent extends Agent
      policy: policy

    module.exports = {policy,TaggedCall,TaggedAgent}
