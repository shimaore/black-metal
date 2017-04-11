    @name = 'black-metal'

    Call = require './call'
    Agent = require './agent'
    Queuer = require './queuer'
    api = require './api'
    {policy,TaggedCall,TaggedAgent} = require './tagged'

    module.exports = {Call,Agent,Queuer,api,policy,TaggedCall,TaggedAgent}
