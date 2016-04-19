pere
====

pere is a utility for record and playback of DOM edits. It uses the DOM
MutationObserver to watch for changes and turns those changes into a JSON
representation that can be stored and replayed. It preserves timing
information, which means you can use it to show the process of writing a document.

You can see an example of using it for that purpose here:
https://samgentle.com/posts/2016-04-10-red-green-refactor-writing

And a demo where you can record and playback your own edits here:
https://demos.samgentle.com/pere

So far it only supports bare elements (no attributes) and text. Attributes
wouldn't be too hard to add, but I didn't need them at the time I wrote it.
The main thing missing from this library is a rigorous set of tests. It does a
lot of tricksy algorithmic things and text editing/patching is notoriously
easy to get wrong.

If you're interested in using this but wish it was more reliable, get in
touch. Or, y'know, just fork it and make it work yourself. Either is fine.


String diffing
--------------

One of the trickiest parts of this is that MutationObserver only gives us the
raw content of text nodes. It's up to us to turn that into a minimal
representation of the actual changes. This is a problem known as the longest
common subsequence, and the canonical algorithm is called the Myers diff
algorithm, which is documented here: http://www.xmailserver.org/diff2.pdf

I am using something... similar. It's not the proper Myers diff, really just a
kind-of-breadth-first-kind-of-A* thing. I don't recommend this, but it was
helpful for me to learn about how the Myers diff algorithm works. It would be
faster and neater to replace it with the actual algorithm, but since it works
well enough I haven't done that.

We start with some supporting cast members for our search algorithm. We need a
sorted insert function to maintain our priority queue of nodes in edit-space
to visit.

    defaultComp = (a, b) -> a > b

    sortedInsert = (ary, item, greater=defaultComp) ->
      start = 0
      end = ary.length
      while start < end
        i = Math.floor (start + end) / 2
        if greater item, ary[i]
          start = i + 1
        else
          end = i

      ary.splice start, 0, item
      ary

This is just a simple print function to show our path through edit space.
Useful for debugging.

    disp = (str1, str2, path) ->
      ret = []
      while path
        ret.unshift "-#{str1[path.prev.x]}" if path.dir is 0
        ret.unshift "+#{str2[path.prev.y]}" if path.dir is 1
        ret.unshift "=#{str1[path.prev.x]}" if path.dir is 2
        path = path.prev
      ret.join ''

    diffcomp = (path1, path2) ->
      path1.d > path2.d

This is the main function the runs the search. We maintain a list of nodes in
edit space, adding new ones as they come up. Edit space is, roughly, a
coordinate system where the x axis consumes characters from one string, the y
axis consumes characters from another, and diagonal movement represents a
match (consuming both at the same time).

In this system, diagonal movement is free, whereas other movements have a cost
of 1. This means the shortest path will always be the one with the most
diagonals. We keep going until we reach the bottom right of edit-space (all
characters consumed from both strings).

Finally, we return the path we took to get here (stored in a linked list on
the nodes themselves).

    diff = (str1, str2) ->
      visited = {}
      paths = [{x: 0, y: 0, d: 0, dir: -1, prev: null}]
      n = 0
      until !paths[0] or (paths[0].x == str1.length) and (paths[0].y == str2.length)
        thisPath = paths.shift()
        {x, y, dir, d, prev} = thisPath
        continue if visited["#{x}:#{y}"] or (x > str1.length) or (y > str2.length)

        visited["#{x}:#{y}"] = true
        if str1[x] and (str1[x] == str2[y])
          sortedInsert paths, {x: x + 1, y: y + 1, d, dir: 2, prev: thisPath}, diffcomp
        else if dir is 0
          sortedInsert paths, {x: x + 1, y, d: d + 1, dir: 0, prev: thisPath}, diffcomp
          sortedInsert paths, {x, y: y + 1, d: d + 1, dir: 1, prev: thisPath}, diffcomp
        else #if dir is 1
          sortedInsert paths, {x, y: y + 1, d: d + 1, dir: 1, prev: thisPath}, diffcomp
          sortedInsert paths, {x: x + 1, y, d: d + 1, dir: 0, prev: thisPath}, diffcomp

      p = paths[0]
      ret = []
      lastt = null
      chunks = { '-': [], '+': [], '=': [] }
      while p
        t = '-+='[p.dir]

        # Batch sequences of =, +/- together
        if !p.prev or (t != lastt and (t is '=' or lastt is '='))
          i = 0
          for _t in '+-=' when chunks[_t].length
            i += 1
            ret.unshift {t: _t, c: chunks[_t]}
            chunks[_t] = []

        if chunks[t]
          c = if p.dir is 1 then str2[p.prev.y] else str1[p.prev.x]
          chunks[t].unshift c

        lastt = t
        p = p.prev
      ret

Our return value includes full details of all matched and removed characters.
We don't need that for record/playback, but it might be useful for something
else, so we provide an extra function that strips it all out.

    strip = (patch) ->
      for p in patch
        if p.t is '=' or p.t is '-'
          {l: p.c.length, t: p.t}
        else
          p

Patch is the inverse of diff (ie, it takes `str1` and the output of
`diff(str1, str2)` and gives you `str2`). It is, mercifully, much simpler than
diff.

    patch = (str, patch) ->
      newstr = []
      i = 0
      si = 0
      for p in patch
        if p.t is '='
          l = p.l
          newstr.push str.slice(si, si + l)
          i += l
          si += l
        if p.t is '+'
          newstr.push p.c.join('')
          i += p.l
        if p.t is '-'
          si += p.l
      newstr.join('')

Event API
---------

Observe takes an element and a callback, and calls that callback with all
edits that happen to the element (or its children). This includes structural
DOM changes as well as text edits.

The format looks like `{type: 'type', id: 'foo', data: 'bar'}` with various
properties instead of data for the different types. The types are `add` (for a
new node), `remove` (for a node going away) and `edit` (for changes to a
node). The edit type contains a diff as created by our `diff` function.

    observe = (el, callback) ->
      el.innerHTML = ""
      nodeid = 0
      nodeids = new Map()
      oldVals = new Map()
      removedids = new Map()
      editTimers = new Map()
      timestamp = Date.now()

      addNode = (parent, node) ->
        return if nodeids.get(node)
        id = nodeid++
        nodeids.set node, id
        oldVals.set(node, node.data) if node.data
        console.warn "parent unknown", parent if !nodeids.get(parent)? and parent isnt el
        if node.previousSibling? and !nodeids.get(node.previousSibling)?
          console.warn "sibling unknown", node.previousSibling, removedids.get(node.previousSibling)
        callback {
          type: 'add', id, nodeType: node.nodeType, tag: node.tagName,
          parent: nodeids.get(parent), data: node.data,
          after: (nodeids.get(node.previousSibling))
          timestamp: Date.now() - timestamp
        }

        addNode node, child for child in node.childNodes if node.childNodes

      removeNode = (node) ->
        removeNode child for child in node.childNodes if node.childNodes

        id = nodeids.get node
        nodeids.delete node
        oldVals.delete node
        clearTimeout editTimers.get node
        editTimers.delete node
        removedids.set node, id
        callback {type: 'remove', id, timestamp: Date.now() - timestamp} if id?

      observer = new MutationObserver (mutations) ->
        for mut in mutations then do (mut) ->
          if mut.type is 'childList'
            addNode mut.target, addedNode for addedNode in mut.addedNodes

            removeNode removedNode for removedNode in mut.removedNodes

          else if mut.type is 'characterData'
            target = mut.target
            return if editTimers.get target
            timer = setTimeout ->
              editTimers.delete target

              oldVal = oldVals.get(target) or ''
              newVal = target.data
              oldVals.set(target, newVal)
              return if oldVal == newVal

              edits = strip diff oldVal, newVal

              id = nodeids.get(target)
              callback {type: 'edit', id, edits, timestamp: Date.now() - timestamp} if id?
            , Math.floor(Math.random()*300)+200

            editTimers.set target, timer

      observer.observe el, {childList: true, characterData: true, subtree: true}
      {unobserve: -> observer.disconnect()}

Reflect will take a DOM element and return an object that you can call
`.apply` on to apply edits as produced by `observe`. That is, you should be able
to `apply` all the edits emitted by `observe` to an element and get the same
document back.

    reflect = (el) ->
      el.innerHTML = ""
      nodeids = {}
      apply: (e) ->
        switch e.type
          when 'add'
            if e.nodeType is 1
              node = document.createElement e.tag
            else if e.nodeType is 3
              node = document.createTextNode ""
            node.data = e.data if e.data?
            node.innerHTML = e.innerHTML if e.innerHTML?
            nodeids[e.id] = node
            parent = nodeids[e.parent] or el

            if afterNode = nodeids[e.after]
              parent.insertBefore node, afterNode.nextSibling
            else
              parent.insertBefore node, parent.firstChild

          when 'remove'
            nodeids[e.id].remove()
            delete nodeids[e.id]

          when 'edit'
            node = nodeids[e.id]
            newdata = patch(node.data, e.edits)
            node.data = newdata


Playback API
------------

This is a higher-level wrapper around `observe` and `reflect`. Instead of
operating with callbacks, it just returns or accepts an array of edits.

Record returns a mutable array that mutates as modifications are made. That
is, the array it returns should always reflect the total edits made to the
element at the time it is accessed.

    record = (el, callback=->) ->
      edits = []
      observer = observe el, (edit) ->
        edits.push edit
        callback(edit)
      edits.unobserve = observer.unobserve
      edits

Playback accepts an array of edits and, unlike the `apply` method on
`observe`, it respects timing information. `playback` returns an object that
can be used to control playback, including by pausing, resuming, or changing
playback speed.

    playback = (el, edits, speed=1.0) ->
      reflector = reflect el
      i = 0
      lasttime = 0
      timeout = null
      pause = ->
        clearTimeout timeout

      play = ->
        return unless edits[i]
        timeout = setTimeout ->
          edit = edits[i]

          reflector.apply edit
          lasttime = edit.timestamp
          i++
          play()
        , ((edits[i].timestamp - lasttime) / speed) or 0

      setSpeed = (_speed) ->
        speed = _speed
        pause()
        play()
      play()

      {setSpeed, pause, play}

Exports
-------

Export our API to either NodeJS or the browser.

    ex = {observe, reflect, record, playback}

    if typeof window == 'undefined'
      module.exports = ex
    else
      window.pere = ex