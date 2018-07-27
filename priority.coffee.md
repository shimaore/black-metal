Keep the highest priority value

    module.exports = priority = (tags) ->
      result = []
      tags.forEach (tag) ->
        if m = tag.match /^priority:(\d+)$/
          result.push parseInt m[1], 10
      priorities = result.sort (a,b) ->
        switch
          when not a?
            1
          when not b?
            -1
          else
            b-a
      priorities[0] ? null
