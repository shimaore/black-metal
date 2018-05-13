Call Transitions
----------------

    seconds = 1000
    minutes = 60*seconds

    initial_state = 'new'

    _transition =

Events:
- fail
- pool
- unpool
- hangup: local call, hungup locally
- transferred: remote call, transferred locally
- hungup: remote call, hungup by remote end
- miss : disappeared from system
- retry: call presented to agent, but agent missed the call
- bridge
- unbridge
- unbridge_ignore
- broadcast
- handle

If a call is transitioned back to `new` it means it got forgotten / is in overflow.

      new:
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped' # disappeared from system
        pool: 'pooled'
        handle: 'handled'
        sent_to_agent: 'handled'
        set_agent: 'handled'
        timeout: 'new' # forgotten
        timeout_duration: 97*seconds

Only pooled calls actually get considered.

      pooled:
        broadcast: 'handled'
        handle: 'handled'
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped' # disappeared from system
        timeout: 'new'
        timeout_duration: 31*seconds # overflow/forgotten
        unpool: 'new'

      handled:
        bridge: 'bridged'
        broadcast: 'handled' # This allows multiple presentations of the same call.
        fail: 'dropped'
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped' # disappeared from system
        retry: 'pooled' # force re-try

This might lead to multiple agents ringing even if the `broadcast` option is not active, so we delay it a little.

        timeout: 'pooled' # force re-try
        timeout_duration: 131*seconds

      bridged:
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped' # disappeared from system
        pool: 'pooled' # ?
        bridge: 'bridged' # more than one call bridged (e.g. during transfer)
        unbridge_ignore: 'bridged' # unbridge but other remain (definitely during transfers)
        unbridge: 'unbridged'

      unbridged:
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped' # disappeared from system
        bridge: 'bridged'
        pool: 'pooled'
        retry: 'pooled' # ?
        timeout: 'pooled' # forgotten
        timeout_duration: 36*seconds

      dropped:
        pool: 'pooled' # on transfer
        retry: 'pooled' # ?

    module.exports = (require './transition') 'call', initial_state, _transition
