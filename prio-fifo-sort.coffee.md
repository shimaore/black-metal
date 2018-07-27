Call sorting
------------

    module.exports = prio_fifo_sort = (a,b) ->

      a_prio = a.priority
      b_prio = b.priority
      fifo = a.started_at - b.started_at

- sort higher priority calls first
- equal priority calls are presented first-in first-out

      if not a_prio? and not b_prio?
        return fifo
      if a_prio? and not b_prio?
        return -1
      if b_prio? and not a_prio?
        return 1
      (b_prio - a_prio) or fifo
