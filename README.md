tsLua
=====

A Lua interpreter with reference-counting garbage collection and copy-on-write tables
##Current status
100% Vaporware (The code you see here is an unmodified fork from LuaJIT 2.0)

##Planned changes (compared with LuaJIT 2.0)
###New types:
* **tstate**
  * Represents an (immutable) snapshot of the state of a table
  * Includes all key/value pairs of the table and (if present) its metatable, at the time the tstate is created
    * Does not reflect any subsequent changes to the parent table
      * Except for entries of weak tables being deleted by the garbage collector
  * (non-weak) tstates are interned (just like strings)
    * Hence they are useful as keys in another table

###Changes to core semantics:
####Garbage collection
* Unreachable objects that are not part of cycles will be collected (and if necessary, finalized) immediately.
* Unreachable objects that are part of cycles will all be collected by a call to `collectgarbage('collect')`
  * Other `collectgarbage` subfunctions will initially be no-ops (except `count` which is unchanged.)
* Tables can have finalizers (like in Lua 5.2)

####Other
* The metatables of all tables will be tstates (instead of tables) <small>(this will break some programs; see below)</small>

###Standard library additions
* `table.clone(ts)`: Create a new table with content copied from an existing table
* `table.snapshot(t, true)`: Get a snapshot of the table's current content, as a tstate
* `table.swap(t1, t2)`: Exchange the two tables' states, so that each table takes on the prior state of the other

###Syntax additions
* `@{foo=42}` is a shorthand for `table.snapshot({foo=42}, true)`, so you can write tstate constructors concisely.
  * There must be no space between the `@` and the `{`.  Also prefixing a table *variable* with `@` is not allowed.

##tstate Semantics
If: `t` is a table and  
`ts` is a *tstate* that was once the state of `t` (i.e. it was obtained using `table.snapshot(t, true)`) and  
`ts1` and `ts2` are arbitrary *tstate*s, then:

* `type(ts) == type(ts1) == type(ts2) == 'tstate'`
* `table.clone(ts)` creates a new table whose initial state is `ts`
* `table.snapshot(t, true)` gets the current state of the table `t` as a *tstate*.
* `debug.getmetatable(ts)` returns a copy of `t`'s metatable at the time `ts` was obtained.
* `getmetatable(ts)` returns the same value that `getmetatable(t)` would have returned at the time `ts` was created (Note that this may be affected by a `__metatable` metamethod.)
* `rawequal(table.snapshot(table.clone(ts), true), ts)`
  * Possibly also `table.snapshot(table.clone(ts)) == ts`, but an `__eq` metamethod may override this.
* `table.snapshot(t, true) ~= t and not rawequal(table.snapshot(t, true), t)` (Even if an `__eq` metamethod is present, it will not be invoked as the types do not match.)
* `not rawequal(table.clone(table.snapshot(t, true), t)` (These are distinct tables that happen to have the same state now but may diverge in future.)
* `rawequal(ts1, ts2)` if:
  * `ts1` and `ts2` have the same key/value pairs (according to `rawequal` applied to the values) and
  * `rawequal(getmetatable(ts1), getmetatable(ts2))` and
  * Neither `ts1` or `ts2` has a `__mode` (weak tstates are not interned.)
* If you have a reference to a tstate, then it must once have been the state of a table
  in the same VM (though that table may have changed since then.)

###Implicit conversions
* `table.snapshot(ts, true)` returns `ts`.

###Extensions to existing standard library functions
* Most standard library functions that accept a table will also accept a tstate, converting using `table.clone` if necessary, e.g. `unpack(ts)` will return the same results as `unpack(table.clone(ts))`.

The exceptions are those functions that depend on modifying tables in-place. It is an error to pass a tstate to any of the following:

* Both arguments of `table.swap`
* The first argument of `table.insert` and `table.remove`
* The first argument of `table.sort`
* The first argument of `setmetatable` and `debug.setmetatable`
* The first argument of `rawset`

##Incompatibilities with Lua 5.1 and LuaJIT 2.0:<ul>
<li> Metatables are no longer tables; they are tstates instead.<ul>
<li>`setmetatable(t, mt)` is equivalent to `setmetatable(t, table.clone(mt))`.  That is, metatables (of tables, though not userdata) are always copied when setting.
<li>The following will no longer work:
<pre>
local mt = {}
local t = setmetatable({}, mt)
function mt:__tostring()
  return "Hello! I have a __tostring metamethod!"
end
print(t) -- Expecting to see the message defined above
</pre> because `t` still has the empty metatable it was originally given.  It does not see the subsequent change to `mt`.  Some programs will need their definitions shuffled around to avoid this, e.g.
<pre>
local mt = {}
local t = {}
function mt:__tostring()
  return "Hello! I have a __tostring metamethod!"
end
setmetatable(t, mt)
print(t) -- Now we will see the expected message
</pre>
</ul>

##Implementation
* tsLua uses an automatic reference counting in place of Lua's incremental mark/sweep collector.
  * It will eventually include a (non-incremental) cycle collector.  Early versions will be unable to collect cycles (this should be worked around using weak references if required.)
* `table.snapshot` has copy-on-write semantics.  Initially it just bumps the reference count. Physical copying is deferred until the table is modified through the parent reference (or another clone.)
  * Tables with a reference count of 1 can be cloned/snapshotted as many times as you like with no copying being necessary as long as the parent reference is overwritten (or becomes unreachable) each time.
* full userdata and ffi cdata objects are also reference-counted (though there is no lazy copying or interning and they do not need the cycle collector.)

##Codebase
Currently a fork of LuaJIT 2.0

###Alternate versions planned
* There may be a version based on Lua 5.2 in future (but not yet as I am not familiar enough with the 5.2 codebase)
* There will probably never be a Lua 5.1-based version (the upstream product needs to be actively maintained, and Lua 5.1 is not.)
  * The same applies to LuaJIT 1.x 
* A luaj or kahlua2 would also probably not be feasible, because of Java's underlying (tracing) garbage collection, which is a shame.
  * But maybe a Lua 5.2 version could be built with a C-to-Java translator (that keeps the whole heap in a huge `ByteBuffer`)
* I have no idea yet whether a LuaJIT 2.1 version will be feasible, but I hope it will.