Fast serialization for Lua
==========================

    local marshal = require "marshal"

Installation
------------

Requirements: Lua 5.1 or LuaJIT, a C compiler and the Lua development
headers (e.g. the `liblua5.1-0-dev` package on Debian/Ubuntu).

Install from the source tree (recommended for this fork), which builds and
installs the module according to the rockspec:

    luarocks make lua-marshal-1.0-1.rockspec

If your Lua/luarocks is installed system-wide, you may need `sudo`:

    sudo luarocks make lua-marshal-1.0-1.rockspec

To install into your user tree instead (no `sudo` needed):

    luarocks --local make lua-marshal-1.0-1.rockspec

`--local` usually requires the luarocks bin directory in your `PATH` and
module path, e.g.:

    export PATH="$HOME/.luarocks/bin:$PATH"
    eval "$(luarocks --local path)"

The same commands work with `luarocks install <package>` if the package is
available from a rocks server:

    sudo luarocks install lua-marshal

After installation:

    local marshal = require "marshal"

Note for Minetest/Luanti (e.g. TechAge): the module is loaded from a trusted
mod via `minetest.request_insecure_environment()` and `require("marshal")`,
so the mod must be listed in `secure.trusted_mods`.

Provides:
---------

* s = marshal.encode(v[, constants])    - serializes a value to a byte stream
* t = marshal.decode(s[, constants])    - deserializes a byte stream to a value
* t = marshal.clone(orig[, constants])  - deep clone a value (deep for tables and functions)

Features:
---------

Serializes tables, which may contain cycles, Lua functions with upvalues and basic data types.

All functions take an optional constants table which, if encountered during serialization,
are simply referenced from the constants table passed during deserialization. For example:

```Lua
local orig = { answer = 42, print = print }
local pack = marshal.encode(orig, { print })
local copy = marshal.decode(pack, { print })
assert(copy.print == print)
```

Hooks
-----

A hook is provided for influencing serializing behaviour via the `__persist` metamethod.
The `__persist` metamethod is expected to return a closure which is called during
deserialization. The return value of the closure is taken as the final decoded result.

This is useful for serializing both userdata and for use with object-oriented Lua,
since metatables are not serialized.

For example:

```Lua
local Point = { }
function Point:new(x, y)
   self.__index = self
   return setmetatable({ x = x, y = y }, self)
end
function Point:move(x, y)
   self.x = x
   self.y = y
end
function Point:__persist()
   local x = self.x
   local y = self.y
   return function()
      -- do NOT refer to self in this scope
      return setmetatable({ x = x, y = y }, Point)
   end
end
```
The above shows a way to persist an "instance" of Point (if you're thinking
in OO terms). In this case `Point` itself will be included in the encoded chunk
because it's referenced as an upvalue of the returned closure.

The `__persist` hook may *NOT* refer to the receiver (i.e. `self`
in the example) because this will cause deep recursion when upvalues
are serialized.

Limitations:
------------

Coroutines are not serialized. Userdata doesn't serialize either
however support for userdata the `__persist` metatable hook can be used.

Metatables and function environments are not serialized.

C functions, threads and userdata without a `__persist` hook cannot be
serialized; they are stored as `nil` instead of raising an exception.

Serialized code is not portable.

