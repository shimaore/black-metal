Call Transitions
----------------

    seconds = 1000
    minutes = 60*seconds

    initial_state = 'new'

    _transition =

Events:
- hangup
- transferred: from screeching-eggs
- hungup: from screeching-eggs
- miss : disappeared from system
- fail: originate-external failed
- pool
- unpool
- bridge
- unbridge

If a call is transitioned back to `new` it means it got forgotten / is in overflow.

      new:
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped'
        fail: 'dropped'
        track: 'tracked'
        pool: 'pooled'
        # unpool:
        # bridge: ignored, because we only track state for pooled calls, not all calls
        # unbridge:
        timeout: 'new' # forgotten
        timeout_duration: 97*seconds

Only pooled calls actually get considered.

      pooled:
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped'
        # fail:
        track: 'tracked'
        # pool:
        unpool: 'new'
        bridge: 'bridged'
        # unbridge:
        timeout: 'new'
        timeout_duration: 31*seconds # overflow/forgotten

      tracked:
        hangup: 'dropped'
        transferred: 'dropped'
        hungup: 'dropped'
        miss: 'dropped'
        # fail:
        # track:
        pool: 'pooled'
        # unpool:
        bridge: 'bridged'
        # unbridge:
        timeout: 'new'
        timeout_duration: 31*seconds # overflow/forgotten

      bridged:
        hangup: 'dropped'
        miss: 'dropped'
        # fail:
        pool: 'pooled'
        # unpool:
        bridge: 'bridged' # more than one call bridged (e.g. during transfer)
        unbridge: 'unbridged'
        # timeout:

      unbridged:
        hangup: 'dropped'
        miss: 'dropped'
        # fail:
        track: 'tracked'
        pool: 'pooled'
        # unpool:
        bridge: 'bridged'
        # unbridge:
        timeout: 'new' # re-queue
        timeout_duration: 36*seconds

      dropped:
        # hangup:
        # miss:
        # fail:
        pool: 'pooled' # on transfer
        track: 'tracked'
        # unpool:
        # bridge:
        # unbridge:

    module.exports = (require './transition') 'call', initial_state, _transition
