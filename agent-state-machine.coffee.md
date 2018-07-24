Agent Transitions
-----------------

    seconds = 1000
    minutes = 60*seconds

    initial_state = 'logged_out'

    _transition =

### Event: login

The `login` event is triggered by:
- mode A: agent calls into the queuer
- mode B: agent logs in (either via TUI or GUI) and they were in no group so far.

### Event: logout

The `logout` event is triggered by:
- mode A: agent hangs up the call to the queuer.
- mode B: agent logs out (TUI or GUI) and there are no remaining groups they belong to.

### Event: start-of-call

The `start_of_call` event is triggered when a call outside of the queuer is presented or sent by the agent, making it unavailable to take new calls, without logging them out of the queuer.

### Event: end-of-calls

The `end_of_calls` event is triggered when all calls related to an agent (outside of the queuer) are finished.

### Event: present

The `present` event is triggered when the queuer assigns a call to an agent in the `idle` state.

### Event: missed

A `missed` event might occur if the agent
- mode A: does not acknowlegde the call (TUI or GUI)
- mode B: does not answer the call

The agent is then marked `away` and this is reported to the manager.

### Event: answer

The `answer` event is triggered if the agent and the remote party are connected.

### Event: failed

The `failed` event is triggered if the call could not be presented.

### Event: hangup

The `hangup` event is triggered:
- if the agent hangs up the call via GUI (not implemented)
- if the remote party hangs up

### Event: agent_hangup

The `agent_hangup` event is triggered:
- if the agent hangs up the call with the queuer

### Event: complete

The complete event is triggered:
- mode A: if the agent acknowledges the wrap-up (TUI or GUI)
- mode B: if the agents hangs up or acknowledges the wrap-up (GUI)
Note: if the agent previously hung-up the wrap-up can only be ack'ed via the GUI.

### Event: timeout

The timeout event is triggered when an agent has been in the same state for a predefined delay.

### State: logged-out

The logged-out state is the initial state of the state machine.

In logged-out state an agent is not considered for calls.

      logged_out:

        login: 'idle'
        bridge: 'logged_out_busy'

### State: logged_out_busy

If an agent receives one or multiple calls in logged-out state, it is moved to the logged-out+busy state.

      logged_out_busy:

        login: 'busy'
        bridge: 'logged_out_busy'
        end_of_calls: 'logged_out'

Force log out on second logout.

        logout: 'logged_out'

### State: idle, waiting, pending.

An available agent is `idle`, `waiting`, or `pending`. The three states are equivalent, except from the queuer's point of view.

In `idle` state, the agent is automatically transitioned to `evaluating`.

This state should be short, so if it lasts too long resubmit it.

      idle:

        bridge: 'busy'
        end_of_calls: 'idle'
        logout: 'logged_out'
        evaluate: 'evaluating'
        timeout: 'idle'
        timeout_duration: 19*seconds

In `waiting` state, the agent is only transitioned on an external event `new_call`.

      waiting:

        bridge: 'busy'
        end_of_calls: 'idle'
        logout: 'logged_out'
        new_call: 'idle'

Regularly re-assess the situation (this allows to flush the pools).

        timeout: 'idle'
        timeout_duration: 599*seconds

All other states will re-transition via `idle` first (to force a re-evaluation of the state of the agent).

### State: evaluating

When an agent is free, we first re-evaluate whether there is any other call we may present to them.

First we look in the pool of existing ingress calls, then in the pool of existing (potential) egress calls.

      evaluating:

        bridge: 'busy'
        logout: 'logged_out'
        present: 'presenting'
        release: 'create_call'

This state should be relatively short, so if it lasts too long resubmit.

        timeout: 'idle'
        timeout_duration: 17*seconds

If no existing (or potential) call exists, we attempt to build a new call.

      create_call:

        bridge: 'busy'
        logout: 'logged_out'
        present: 'presenting'
        created: 'waiting'
        not_created: 'waiting'
        timeout: 'idle'
        timeout_duration: 71*seconds

### State: busy

The busy state is active when an agent's phone is active but on a call not related to the queuer.

      busy:

        bridge: 'busy'
        end_of_calls: 'idle'
        logout: 'logged_out_busy'

### State: away

In `away` state, the queuer will trigger a review of all available agents to dispatch a new call.

      away:

        bridge: 'busy'
        end_of_calls: 'idle'
        login: 'idle'
        logout: 'logged_out'
        timeout: 'idle'
        timeout_duration: 13*seconds

### State: presenting

The presenting state is active when an agent is presented a call but hasn't acknowledged it yet.

Upon transitioning to the presenting state:
- the agent is presented with the call's data and client information
- mode A: the agent is presented with a bip
- mode B: the agent's phone is dialed

      presenting:

        answer: 'in_call'
        missed: 'away'
        failed: 'idle'
        hangup: 'idle'
        agent_hangup: 'idle'

        logout: 'logged_out'

### State: in-call

      in_call:

        hangup: 'wrap_up'
        unbridge: 'wrap_up'
        force_hangup: 'wrap_up'
        agent_hangup: 'idle'
        agent_transfer: 'idle'

        logout: 'logged_out'

### State: wrap-up

      wrap_up:

        complete: 'idle'
        agent_hangup: 'idle'
        agent_transfer: 'idle' # in case the 'hangup' event arrives first

        logout: 'logged_out'

    module.exports = (require './transition') 'agent', initial_state, _transition
