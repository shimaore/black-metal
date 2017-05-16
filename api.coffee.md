    @name = 'black-metal:api'
    seem = require 'seem'

    FS = require 'esl'
    debug = (require 'tangible') @name

API
---

The command can be used either:
- to send an API command, typically a `uuid_`-type command (`uuid_bridge` etc.)
- to send a `myevents <uuid> json` command (lower-level) in order to monitor a call.

In the later case we reconfigure the client connection so that it behaves just like a server connection.

    api = (cfg) -> (cmd) ->
      debug 'api', cmd
      new Promise (resolve) ->

        try
          client = FS.client seem ->

            if cmd.match /^myevents/
              try
                debug 'api monitor'
                yield @send cmd
                yield @event_json 'ALL'

Use linger to capture channel-hangup-complete and other late channel events.

                yield @linger()
                yield @auto_cleanup()
                debug 'api monitor: ready'
                resolve this
              catch error
                debug 'api monitor: error', error
                resolve null
              return

            else

              debug 'api: sending', cmd

              on_success = (res) =>
                debug 'api: success', res.uuid
                @end()
                resolve res.uuid ? true

              on_failure = (error) =>
                debug 'api: error', error
                @end()
                resolve false

              @api cmd
              .then on_success, on_failure

          client.connect (cfg.socket_port ? 5722), '127.0.0.1'
          debug 'api: client connected'
        catch error
          debug 'api: error', error
          resolve false

    module.exports = api
