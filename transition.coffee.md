Handle transitions
------------------

    transition = (name,initial_state,_transition) ->
      (event, notification_data = {}) ->
        debug name, {@key, event}

        old_state = await @state()

        unless old_state?
          await @transition_state old_state, initial_state
          old_state = initial_state

        debug name, {@key, event, old_state}

        unless old_state of _transition
          await @transition_state old_state, initial_state
          throw new Error "Invalid state, transition from #{old_state} → event #{event}"

        unless event of _transition[old_state]
          debug "#{name}: Ignoring event #{event} in state #{old_state}", @key
          return false

        new_state = _transition[old_state][event]

        @queuer.clear_timer @key

        unless new_state of _transition
          await @transition_state old_state, initial_state
          throw new Error "Invalid state machine, transition from #{old_state} → event #{event} leads to unknown state #{new_state}"

        debug name, {@key, event, old_state, new_state}
        if new_state?
          await @transition_state old_state, new_state

          notification_data.old_state = old_state
          notification_data.state = new_state
          notification_data.event = event

          await @notify? notification_data
          next_state = _transition[new_state]
          if 'timeout' of next_state and 'timeout_duration' of next_state
            on_timeout = =>
              @queuer.clear_timer @key
              heal @transition 'timeout'
            @queuer.set_timer @key, setTimeout on_timeout, next_state.timeout_duration-500+1000*Math.random()

          await nextTick()
          heal @post_process notification_data

          return true
        else
          return false

    module.exports = transition
    nextTick = -> new Promise (resolve) -> process.nextTick resolve
    {debug,heal} = (require 'tangible') 'black-metal:transition'
