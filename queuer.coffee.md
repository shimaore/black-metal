    @name = 'black-metal:queuer'
    RedisClient = require 'normal-key/client'

    {debug,foot,heal} = (require 'tangible') @name

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

    nextTick = ->
      new Promise (resolve) ->
        process.nextTick resolve

Main Queuer
===========

    module.exports = (redis_interface,{Agent,Call}) ->

Call Pool
---------

      class CallPool extends RedisClient
        constructor: (queuer,domain,name) ->
          ### istanbul ignore next ###
          throw new Error 'CallPool requires queuer' unless queuer?.is_a_queuer?()
          ### istanbul ignore next ###
          throw new Error 'CallPool requires domain' unless domain?
          debug 'new CallPool', domain, name
          super 'CP', "#{domain}-#{name}"
          @queuer = queuer
          @domain = domain
          @name = name

        interface: redis_interface

        add: (call) ->
          debug 'CallPool.add', @key, call.key
          ### istanbul ignore else ###
          if await super call.key
            heal call.transition 'pool'
            @queuer.notify "pool:#{@domain}:#{@name}", "call:#{call.key}",
              event: 'add'
              call: call
          return

        remove: (call) ->
          debug 'CallPool.remove', @key, call.key
          ### istanbul ignore else ###
          if await super call.key
            heal call.transition 'unpool'
            @queuer.notify "pool:#{@domain}:#{@name}", "call:#{call.key}",
              event: 'remove'
              call: call
          return

        has: (call) ->
          debug 'CallPool.has', @key, call.key
          await super call.key

        calls: ->
          debug 'CallPool.calls', @key
          result = []
          queuer = @queuer
          domain = @domain
          await @forEach (key) ->
            call = new Call key
            await call.set_domain domain
            result.push call
          debug 'CallPool.calls', @key, result.map (c) -> c.key
          result

Available agents
----------------

      class AgentPool extends RedisClient

        constructor: (queuer,domain,name) ->
          ### istanbul ignore next ###
          throw new Error 'AgentPool requires queuer' unless queuer?.is_a_queuer?()
          ### istanbul ignore next ###
          throw new Error 'AgentPool requires domain' unless domain?
          debug 'new AgentPool', domain, name
          super 'AP', "#{domain}-#{name}"
          @queuer = queuer
          @domain = domain
          @name = name

        interface: redis_interface

        add: (agent) ->
          debug 'AgentPool.add', @key, agent.key
          score = await agent.get_missed().catch -> 0.0
          score += Math.random()/100
          @sorted_add agent.key, score

        remove: (agent) ->
          debug 'AgentPool.remove', @key, agent.key
          @sorted_remove agent.key

        reevaluate: (cb) ->
          debug 'AgentPool.reevaluate', @key
          queuer = @queuer
          await @sorted_forEach (key) ->
            debug 'AgentPool.reevaluate', key
            agent = new Agent key
            await cb agent
            return

          return

Call routing
------------

Calls might be ingress or egress.

The behavior is agent-driven; ingress calls wait in a pool, and the pool is depleted as agents become available.

Egress calls are created dynamically when no calls are available for a given agent.

For a given agent, how the ingress pool is filtered + sorted is based on domain, skills, call priorities, etc.

This means calls must be tagged in order to be assessed for a given user. Typical tags may include:
- domain (typically the agent's support domain set will contain a single domain, which is a good idea since currently domains must be colocated);
- queue/fifo/call-group (typically the agent's will log into at least one queue/fifo/call-group);
- service-level (e.g. some agents may handle VIP-level only calls);
- priority (for sorting);
- order of arrival (for sorting);
- etc.

For a given agent, their pool of ingress-calls-to-handle is therefor a subset of the overall pool of calls to handle.

      class Queuer

        is_a_queuer: -> true

        notify: (key,id,data) ->

        constructor: ->
          debug 'new Queuer'
          @__timers = {}

          Agent::queuer = Call::queuer = this

          Agent::Call = Call
          Call::Agent = Agent

          @Agent = Agent
          @Call = Call
          return

        ingress_pool: (domain) ->
          new CallPool this, domain, 'ingress'

        egress_pool: (domain) ->
          new CallPool this, domain, 'egress'

        available_agents: (domain) ->
          new AgentPool this, domain, 'available'

Evaluate Agent
--------------

When an agent moves to the `idle` state, the queuer picks one call out of the pool:
- either an ingress calls (they are always prioritized over egress calls)
- if no ingress calls require attention, an egress call is created and assigned to the agent.

        on_idle_agent: (agent) ->
          debug 'Queuer.on_idle_agent', agent.key

### Precondition

We force a transition:
- to ensure that the current state was one that is valid for us to evaluate from;
- to ensure that no other operation will proceed along the same path at the same time.

          if not await agent.transition 'evaluate'
            debug 'Queuer.on_idle_agent: precondition failed', agent.key
            return

Give `on_agent` a chance to transition the agent out of the available pool.

          await nextTick()

### Build Call

Return:
- a Call (towards or from a selected third-party),
- or `null`.

          build_call = (pool) ->

            debug 'Queuer.on_idle_agent build_call', agent.key

The first step is for the agent to find a suitable call in the pool.

            calls = await pool.calls()
            call = await agent.policy calls

            if not call?
              debug 'Queuer.on_idle_agent build_call: no call found', agent.key
              return null

            return call

          transition_agent = (call) ->

Transition the agent.

            debug 'Queuer.on_idle_agent build_call: transition the agent', agent.key, call.key

            if not await agent.transition 'present', {call}
              debug 'Queuer.on_idle_agent build_call: transition failed', agent.key, call.key
              return false

            debug 'Queuer.on_idle_agent build_call: transitioned', agent.key, call.key

Wait a little bit (this is meant to give a popup some time to settle).

            debug 'Queuer.on_idle_agent build_call: waiting for 1.5s before originate_external', agent.key, call.key
            await sleep 1500-100+200*Math.random()

For a dial-out (egress) call we first need to attempt to contact the destination.
For a dial-in (ingress) call we already have the proper call UUID.

            await call.originate_external()

            debug 'Queuer.on_idle_agent build_call: originate external completed', agent.key, call.key

Notify the agent of the caller's state.

            await agent.set_remote_call call

            return true

### Send to Agent

We need to send the call to the agent (using either onhook or offhook mode).

          send_to_agent = (call) ->

            debug 'Queuer.on_idle_agent send_to_agent: originate', agent.key, call.key

            {reason} = agent_call = await agent.originate_to_agent() # always from remote-call

            debug 'Queuer.on_idle_agent send_to_agent: originate returned', agent.key, call.key, agent_call?.key, reason

            if reason?
              unless call.broadcasting
                await agent.incr_missed()
                await agent.transition 'missed', {call,reason}
              return false

            debug 'Queuer.on_idle_agent send_to_agent: bridge', agent.key, call.key, agent_call.key

            unless await call.bridge agent_call
              heal call.remove agent_call.key # undo what was done in `call.originate_internal`
              await agent_call.hangup()
              await agent.transition 'failed', {call}
              return false

            debug 'Queuer.on_idle_agent send_to_agent: Successfully bridged', agent.key, call.key, agent_call.key
            await call.set_remote_agent agent.key
            await agent.transition 'answer', {call}

### Main body for `on_idle_agent`

Clean up

          some_call = await agent.get_remote_call()

          if some_call?
            debug.dev 'Error: Agent was idle/waiting but still had a remote call', agent.key, some_call.key
          await agent.set_remote_call null

Ingress pool

          debug 'Queuer.on_idle_agent: ingress pool', agent.key
          ingress_pool = @ingress_pool agent.domain

          remote_call = await build_call(ingress_pool).catch (error) ->
            debug.ops 'Queuer.on_idle_agent: ingress pool, error in build_call', error.stack ? error.toString()
            null

          if remote_call?

            debug 'Queuer.on_idle_agent: ingress pool, got remote-call', agent.key, remote_call.key

            handlers = await remote_call.incr 'handlers', 1
            debug 'Queuer.on_idle_agent: ingress pool, handlers', agent.key, remote_call.key, handlers

            try

              if handlers is 1 or remote_call.broadcasting
                if await transition_agent remote_call
                  if await send_to_agent remote_call
                    return

            catch error
              debug "Queuer.on_idle_agent: ingress pool, error", agent.key, remote_call.key, error

            handlers = await remote_call.incr 'handlers', -1

          else
            debug "Queuer.on_idle_agent: ingress pool, no matching client call", agent.key

Egress pool

          debug 'Queuer.on_idle_agent: egress pool', agent.key
          egress_pool = @egress_pool agent.domain

          remote_call = await build_call(egress_pool).catch (error) ->
            debug.ops 'Queuer.on_idle_agent: egress pool, error in build_call', error.stack ? error.toString()
            null

          if remote_call?

            debug 'Queuer.on_idle_agent: egress pool, got client call', agent.key, remote_call.key

We forcibly remove the call so that we do not end up ringing the same prospect/customer twice, esp. if in the first case there was no agent available.

            await egress_pool.remove remote_call

            handlers = await remote_call.incr 'handlers', 1
            debug 'Queuer.on_idle_agent: egress pool, handlers', agent.key, remote_call.key, handlers

            try

              if handlers is 1 or remote_call.broadcasting
                if await transition_agent remote_call
                  if await send_to_agent remote_call
                    return

            catch error
              debug "Queuer.on_idle_agent: ingress pool, error", agent.key, remote_call.key, error

            handlers = await remote_call.incr 'handlers', -1

          else
            debug "Queuer.on_idle_agent: egress pool, no matching client call", agent.key

No call

          debug 'Queuer.on_idle_agent: no call was available, releasing', agent.key
          if await agent.transition 'release'
            await agent.park()
          else
            true

        queue_ingress_call: (call) ->
          debug 'Queuer.queue_ingress_call', call.key
          await call.set_poolable()
          domain = await call.get_domain()
          await @ingress_pool(domain).add call

On a newly-pooled call, we re-assess the situation of the agents in the available pool to decide where to send the call.

        on_pooled_call: (event,domain) ->
          debug 'Queuer.on_pooled_call: start', domain, event
          await nextTick()
          @available_agents(domain).reevaluate foot (agent) ->
            await nextTick()
            debug 'Queuer.on_pooled_call for agent', agent.key, event
            agent.transition event

Data is optional but is used by huge-play's `create-queuer-call`.

        create_egress_call_for: (agent,data) ->
          debug 'Queuer.create_egress_call_for', agent.key, data

The call instance is created using data found e.g. in a database, the (egress) call is placed in the pool, and the idle agents are re-evaluated.

          call = await agent.create_egress_call data
          if call?
            debug 'Queuer.create_egress_call_for: queue egress call', agent.key, call.key
            await call.set_poolable()
            await agent.transition 'created'
            domain = await call.get_domain()
            await @egress_pool(domain).add call
          else
            debug 'Queuer.create_egress_call_for: no call', agent.key
            await agent.transition 'not_created'

        on_agent: (agent,data) ->
          {state} = data
          debug 'Queuer.on_agent', agent.key, state

Only states were the agent might transition via `evaluate` are considered as states were the agents is available.

          switch state
            when 'idle', 'waiting'
              await @available_agents(agent.domain).add agent
            else
              await @available_agents(agent.domain).remove agent

          switch state
            when 'logged_out'
              await agent.reset_missed()
              await agent.clear()
              await agent.set_offhook_call null
              await agent.set_onhook_call null
              await agent.set_remote_call null

            when 'in_call'
              await agent.reset_missed()

            when 'wrap_up'
              await agent.wrapup()

If the agent is idle, move forward in the background.

            when 'idle'
              await @on_idle_agent agent

            when 'create_call'
              await @create_egress_call_for agent

          return

        on_call: (call,data) ->
          {state} = data
          debug 'Queuer.on_call', call.key, state

          domain = await call.get_domain()
          unless domain?
            debug 'Queuer.on_call: no domain, ignoring', call.key, state
            return

          ingress_pool = @ingress_pool domain
          egress_pool = @egress_pool domain

          switch state

            when 'new' # aka `forgotten`
              await call.reset 'handlers'
              if await call.poolable()
                heal ingress_pool.add call
              else
                debug.dev 'Ignoring non-poolable call', call.key

            when 'pooled'
              heal @on_pooled_call 'new_call', domain

            when 'bridged'
              await ingress_pool.remove call
              await egress_pool.remove call

              await call.set_answered()

              if data.agent_call?

Hang up all other (ringing) agents.

                await call.unbridge_except data.agent_call.key

Do not automatically close the agent's call (in `dropped`) when a remote party hangs up.

              await call.clear()

            when 'dropped'
              await ingress_pool.remove call
              await egress_pool.remove call
              await call.unbridge_except()

          return

        clear_timer: (key) ->
          if @__timers[key]?
            debug 'Queuer.clear_timer', key
            clearTimeout @__timers[key]
            delete @__timers[key]

        set_timer: (key,timer) ->
          @clear_timer key
          @__timers[key] = timer

        end: ->
          for own key of @__timers
            @clear_timer key
          return

Switch agent
------------

This is used by `huge-play` in order to track calls connected to an agent (especially outside the queuer).
The `call` is the agent-side leg (never a remote leg).

        set_agent: (call,new_key) ->
          debug 'Queuer.set_agent', call?.key, new_key

          return unless call? and new_key?

Hmmm this obviously test for some condition on the remote agent (so probably while doing transfers), but it could use some description.

          old_key = await call.get_remote_agent()
          return if old_key is new_key

Let the call know which agent it is connected to.

          await call.set_local_agent new_key

          return

Agent behavior
--------------

Internal-lines, two ways to reach:
- mode A (off-hook agents): agents call into a number, are parked until a call is presented.
- mode B (on-hook agents): we ring the phone of the agent.

The state machine handles both modes.

Egress calls only really make sense in mode A.

      Queuer
