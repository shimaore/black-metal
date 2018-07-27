    assert = require 'assert'

    describe 'Priority selector', ->
      priority = require '../priority'

      it 'should work', ->

        assert null is priority []
        assert null is priority ['skill:good']
        assert null is priority ['queue:best']
        assert null is priority ['queue:best','priority:']
        assert 0 is priority ['queue:best','priority:0']
        assert 1 is priority ['queue:best','priority:1']
        assert 1 is priority ['queue:best','priority:1','priority:0']
        assert 4 is priority ['queue:best','priority:3','priority:4']
        assert 200 is priority ['queue:best','priority:200','skill:decent']
        assert 200 is priority ['queue:best','priority:100','priority:200','skill:decent']
        assert 200 is priority ['queue:best','priority:1','priority:100','priority:200','skill:decent']
