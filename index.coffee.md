    @name = 'black-metal'

    Call = require './call'
    Agent = require './agent'
    Queuer = require './queuer'
    {policy,TaggedCall,TaggedAgent} = require './tagged'

    module.exports = {Call,Agent,Queuer,policy,TaggedCall,TaggedAgent}
