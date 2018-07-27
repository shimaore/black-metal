    assert = require 'assert'
    describe 'Priority then FIFO sort', ->
      prio_fifo_sort = require '../prio-fifo-sort'

      it 'should work', ->

        first = (a) -> a.sort(prio_fifo_sort)[0].should_be_first

        assert first [
          priority: null, started_at: 1,
          priority: null, started_at: 0, should_be_first: true
        ]
        assert first [
          priority: null, started_at: 1,
          priority: 1,    started_at: 10, should_be_first: true
        ]
        assert first [
          priority: 1, started_at: 1,
          priority: 2, started_at: 10, should_be_first: true
        ]
        assert first [
          priority: 1, started_at: 10,
          priority: 1, started_at: 1, should_be_first: true
        ]
        assert first [
          priority: null, started_at: 1
          priority: null, started_at: 5
          priority: 1, started_at: 10
          priority: 2, started_at: 10, should_be_first: true
          priority: 2, started_at: 100
        ]
