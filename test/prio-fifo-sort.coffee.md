    (require 'chai').should()
    describe 'Priority then FIFO sort', ->
      prio_fifo_sort = require '../prio-fifo-sort'

      first = (a) ->
        a.should.have.length.above 1
        r = a.sort prio_fifo_sort
        r[0].should.have.property 'should_be_first', true

      it 'should fail on improper spec', (done) ->
        try
          first [
            {priority: null, started_at: 1}
            {priority: null, started_at: 2, should_be_first: true}
          ]
        catch error
          done()

      it 'should sort on arrival time', ->
        first [
          {priority: null, started_at: 1}
          {priority: null, started_at: 0, should_be_first: true}
        ]
        first [
          {priority: null, started_at: 0, should_be_first: true}
          {priority: null, started_at: 1}
        ]

      it 'should sort on priority (one null)', ->
        first [
          {priority: null, started_at: 1}
          {priority: 1,    started_at: 10, should_be_first: true}
        ]
        first [
          {priority: 1,    started_at: 10, should_be_first: true}
          {priority: null, started_at: 1}
        ]

      it 'should sort on priority', ->
        first [
          {priority: 1, started_at: 1}
          {priority: 2, started_at: 10, should_be_first: true}
        ]
        first [
          {priority: 2, started_at: 10, should_be_first: true}
          {priority: 1, started_at: 1}
        ]

      it 'should sort on priority then arrival time', ->
        first [
          {priority: 1, started_at: 10}
          {priority: 1, started_at: 1, should_be_first: true}
        ]
        first [
          {priority: 1, started_at: 1, should_be_first: true}
          {priority: 1, started_at: 10}
        ]

      it 'should sort in large series', ->
        first [
          {priority: null, started_at: 1}
          {priority: null, started_at: 5}
          {priority: 1, started_at: 10}
          {priority: 2, started_at: 10, should_be_first: true}
          {priority: 2, started_at: 100}
        ]
        first [
          {priority: 2, started_at: 100}
          {priority: null, started_at: 5}
          {priority: 1, started_at: 10}
          {priority: 2, started_at: 10, should_be_first: true}
          {priority: null, started_at: 1}
        ]
