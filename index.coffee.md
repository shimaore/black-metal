    @name = 'black-metal'
    seem = require 'seem'

    FS = require 'esl'
    debug = (require 'debug') @name

    api = (cmd) ->
      debug 'api', cmd
      new Promise (resolve) ->
        try
          client = FS.client seem ->
            debug 'api: sending', cmd
            res = yield @api cmd
            if res.body?[0] is '+'
              resolve true
            else
              resolve false
          client.connect (cfg.socket_port ? 5722), '127.0.0.1'
          debug 'api: client connected'
        catch error
          debug 'api: error', error
          resolve false

    make_queuer = require './queuer'

    module.exports = (redis,policy_for,egress_call_for,profile) ->
      make_queuer redis,policy_for,egress_call_for,profile, api
