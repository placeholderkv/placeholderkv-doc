---
title: "Valkey Lua API reference"
linkTitle: "Lua API"
weight: 3
description: >
   Executing Lua in Valkey
aliases:
    - /topics/lua-api
    - /docs/manual/programmability/lua-api/
---

Valkey includes an embedded [Lua 5.1](https://www.lua.org/) interpreter.
The interpreter runs user-defined [ephemeral scripts](eval-intro.md) and [functions](functions-intro.md). Scripts run in a sandboxed context and can only access specific Lua packages. This page describes the packages and APIs available inside the execution's context.

## Sandbox context

The sandboxed Lua context attempts to prevent accidental misuse and reduce potential threats from the server's environment.

Scripts should never try to access the Valkey server's underlying host systems.
That includes the file system, network, and any other attempt to perform a system call other than those supported by the API.

Scripts should operate solely on data stored in Valkey and data provided as arguments to their execution.

### Global variables and functions

The sandboxed Lua execution context blocks the declaration of global variables and functions.
The blocking of global variables is in place to ensure that scripts and functions don't attempt to maintain any runtime context other than the data stored in Valkey.
In the (somewhat uncommon) use case that a context needs to be maintain between executions,
you should store the context in Valkey' keyspace.

Valkey will return a "Script attempted to create global variable 'my_global_variable" error when trying to execute the following snippet:

```lua
my_global_variable = 'some value'
```

And similarly for the following global function declaration:

```lua
function my_global_function()
  -- Do something amazing
end
```

You'll also get a similar error when your script attempts to access any global variables that are undefined in the runtime's context:

```lua
-- The following will surely raise an error
return an_undefined_global_variable
```

Instead, all variable and function definitions are required to be declared as local.
To do so, you'll need to prepend the [_local_](https://www.lua.org/manual/5.1/manual.html#2.4.7) keyword to your declarations.
For example, the following snippet will be considered perfectly valid by Valkey:

```lua
local my_local_variable = 'some value'

local function my_local_function()
  -- Do something else, but equally amazing
end
```

**Note:**
the sandbox attempts to prevent the use of globals.
Using Lua's debugging functionality or other approaches such as altering the meta table used for implementing the globals' protection to circumvent the sandbox isn't hard.
However, it is difficult to circumvent the protection by accident.
If the user messes with the Lua global state, the consistency of AOF and replication can't be guaranteed.
In other words, just don't do it.

### Imported Lua modules

Using imported Lua modules is not supported inside the sandboxed execution context.
The sandboxed execution context prevents the loading modules by disabling Lua's [`require` function](https://www.lua.org/pil/8.1.html).

The only libraries that Valkey ships with and that you can use in scripts are listed under the [Runtime libraries](#runtime-libraries) section.

## Runtime globals

While the sandbox prevents users from declaring globals, the execution context is pre-populated with several of these.

### The _server_ singleton

The _server_ singleton is an object instance that's accessible from all scripts.
It provides the API to interact with Valkey from scripts.
Its description follows [below](#server_object).

**Note:**
For compatibility with Redis, Valkey also exposes a _redis_ top level object, that exposes the exact same set of APIs as the _server_ object.
Valkey does not intend to drop compatibility for this _redis_ API, but it is recommended to use the _server_ object for newly developed scripts.

### <a name="the-keys-global-variable"></a>The _KEYS_ global variable

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: no

**Important:**
to ensure the correct execution of scripts, both in standalone and clustered deployments, all names of keys that a function accesses must be explicitly provided as input key arguments.
The script **should only** access keys whose names are given as input arguments.
Scripts **should never** access keys with programmatically-generated names or based on the contents of data structures stored in the database.

The _KEYS_ global variable is available only for [ephemeral scripts](eval-intro.md).
It is pre-populated with all key name input arguments.

### <a name="the-argv-global-variable"></a>The _ARGV_ global variable

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: no

The _ARGV_ global variable is available only in [ephemeral scripts](eval-intro.md).
It is pre-populated with all regular input arguments.

## <a name="server_object"></a>_server_ object

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: yes

The Valkey Lua execution context always provides a singleton instance of an object named _server_.
The _server_ instance enables the script to interact with the Valkey server that's running it.
Following is the API provided by the _server_ object instance.

### <a name="server.call"></a> `server.call(command [,arg...])`

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: yes

The `server.call()` function calls a given Valkey command and returns its reply.
Its inputs are the command and arguments, and once called, it executes the command in Valkey and returns the reply.

For example, we can call the `ECHO` command from a script and return its reply like so:

```lua
return server.call('ECHO', 'Echo, echo... eco... o...')
```

If and when `server.call()` triggers a runtime exception, the raw exception is raised back to the user as an error, automatically.
Therefore, attempting to execute the following ephemeral script will fail and generate a runtime exception because `ECHO` accepts exactly one argument:

```lua
127.0.0.1:6379> EVAL "return server.call('ECHO', 'Echo,', 'echo... ', 'eco... ', 'o...')" 0
(error) ERR Wrong number of args calling Valkey command from script script: b0345693f4b77517a711221050e76d24ae60b7f7, on @user_script:1.
```

Note that the call can fail due to various reasons, see [Execution under low memory conditions](eval-intro.md#execution-under-low-memory-conditions) and [Script flags](#script_flags)

To handle Valkey runtime errors use `server.pcall()` instead.

### <a name="server.pcall"></a> `server.pcall(command [,arg...])`

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: yes

This function enables handling runtime errors raised by the Valkey server.
The `server.pcall()` function  behaves exactly like [`server.call()`](#server.call), except that it:

* Always returns a reply.
* Never throws a runtime exception, and returns in its stead a [`server.error_reply`](#server.error_reply) in case that a runtime exception is thrown by the server.

The following demonstrates how to use `server.pcall()` to intercept and handle runtime exceptions from within the context of an ephemeral script.

```lua
local reply = server.pcall('ECHO', unpack(ARGV))
if reply['err'] ~= nil then
  -- Handle the error sometime, but for now just log it
  server.log(server.LOG_WARNING, reply['err'])
  reply['err'] = 'ERR Something is wrong, but no worries, everything is under control'
end
return reply
```

Evaluating this script with more than one argument will return:

```
127.0.0.1:6379> EVAL "..." 0 hello world
(error) ERR Something is wrong, but no worries, everything is under control
```

### <a name="server.error_reply"></a> `server.error_reply(x)`

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: yes

This is a helper function that returns an [error reply](protocol.md#resp-errors).
The helper accepts a single string argument and returns a Lua table with the _err_ field set to that string.

The outcome of the following code is that _error1_ and _error2_ are identical for all intents and purposes:

```lua
local text = 'ERR My very special error'
local reply1 = { err = text }
local reply2 = server.error_reply(text)
```

Therefore, both forms are valid as means for returning an error reply from scripts:

```
127.0.0.1:6379> EVAL "return { err = 'ERR My very special table error' }" 0
(error) ERR My very special table error
127.0.0.1:6379> EVAL "return server.error_reply('ERR My very special reply error')" 0
(error) ERR My very special reply error
```

For returning Valkey status replies refer to [`server.status_reply()`](#server.status_reply).
Refer to the [Data type conversion](#data-type-conversion) for returning other response types.

**Note:**
By convention, Valkey uses the first word of an error string as a unique error code for specific errors or `ERR` for general-purpose errors.
Scripts are advised to follow this convention, as shown in the example above, but this is not mandatory.

### <a name="server.status_reply"></a> `server.status_reply(x)`

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: yes

This is a helper function that returns a [simple string reply](protocol.md#resp-simple-strings).
"OK" is an example of a standard Valkey status reply.
The Lua API represents status replies as tables with a single field, _ok_, set with a simple status string.

The outcome of the following code is that _status1_ and _status2_ are identical for all intents and purposes:

```lua
local text = 'Frosty'
local status1 = { ok = text }
local status2 = server.status_reply(text)
```

Therefore, both forms are valid as means for returning status replies from scripts:

```
127.0.0.1:6379> EVAL "return { ok = 'TICK' }" 0
TICK
127.0.0.1:6379> EVAL "return server.status_reply('TOCK')" 0
TOCK
```

For returning Valkey error replies refer to [`server.error_reply()`](#server.error_reply).
Refer to the [Data type conversion](#data-type-conversion) for returning other response types.

### <a name="server.sha1hex"></a> `server.sha1hex(x)`

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: yes

This function returns the SHA1 hexadecimal digest of its single string argument.

You can, for example, obtain the empty string's SHA1 digest:

```
127.0.0.1:6379> EVAL "return server.sha1hex('')" 0
"da39a3ee5e6b4b0d3255bfef95601890afd80709"
```

### <a name="server.log"></a> `server.log(level, message)`

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: yes

This function writes to the Valkey server log.

It expects two input arguments: the log level and a message.
The message is a string to write to the log file.
Log level can be on of these:

* `server.LOG_DEBUG`
* `server.LOG_VERBOSE`
* `server.LOG_NOTICE`
* `server.LOG_WARNING`

These levels map to the server's log levels.
The log only records messages equal or greater in level than the server's `loglevel` configuration directive.

The following snippet:

```lua
server.log(server.LOG_WARNING, 'Something is terribly wrong')
```

will produce a line similar to the following in your server's log:

```
[32343] 22 Mar 15:21:39 # Something is terribly wrong
```

### <a name="server.setresp"></a> `server.setresp(x)`

* Since version: 6.0.0
* Available in scripts: yes
* Available in functions: yes

This function allows the executing script to switch between [Redis Serialization Protocol (RESP)](protocol.md) versions for the replies returned by [`server.call()`](#server.call) and [`server.pcall()`](#server.pcall).
It expects a single numerical argument as the protocol's version.
The default protocol version is _2_, but it can be switched to version _3_.

Here's an example of switching to RESP3 replies:

```lua
server.setresp(3)
```

Please refer to the [Data type conversion](#data-type-conversion) for more information about type conversions.

### <a name="server.set_repl"></a> `server.set_repl(x)`

* Since version: 3.2.0
* Available in scripts: yes
* Available in functions: no

**Note:**
this feature is only available when script effects replication is employed.
Calling it when using verbatim script replication will result in an error.
As of Redis OSS version 2.6.0, scripts were replicated verbatim, meaning that the scripts' source code was sent for execution by replicas and stored in the AOF.
An alternative replication mode added in version 3.2.0 allows replicating only the scripts' effects.
As of Redis OSS version 7.0, script replication is no longer supported, and the only replication mode available is script effects replication.

**Warning:**
this is an advanced feature. Misuse can cause damage by violating the contract that binds the Valkey master, its replicas, and AOF contents to hold the same logical content.

This function allows a script to assert control over how its effects are propagated to replicas and the AOF afterward.
A script's effects are the Valkey write commands that it calls.

By default, all write commands that a script executes are replicated.
Sometimes, however, better control over this behavior can be helpful.
This can be the case, for example, when storing intermediate values in the master alone.

Consider a script that intersects two sets and stores the result in a temporary key with `SUNIONSTORE`.
It then picks five random elements (`SRANDMEMBER`) from the intersection and stores (`SADD`) them in another set.
Finally, before returning, it deletes the temporary key that stores the intersection of the two source sets.

In this case, only the new set with its five randomly-chosen elements needs to be replicated.
Replicating the `SUNIONSTORE` command and the `DEL`ition of the temporary key is unnecessary and wasteful.

The `server.set_repl()` function instructs the server how to treat subsequent write commands in terms of replication.
It accepts a single input argument that only be one of the following:

* `server.REPL_ALL`: replicates the effects to the AOF and replicas.
* `server.REPL_AOF`: replicates the effects to the AOF alone.
* `server.REPL_REPLICA`: replicates the effects to the replicas alone.
* `server.REPL_SLAVE`: same as `REPL_REPLICA`, maintained for backward compatibility.
* `server.REPL_NONE`: disables effect replication entirely.

By default, the scripting engine is initialized to the `server.REPL_ALL` setting when a script begins its execution.
You can call the `server.set_repl()` function at any time during the script's execution to switch between the different replication modes.

A simple example follows:

```lua
server.replicate_commands() -- Enable effects replication in versions lower than Redis OSS v7.0
server.call('SET', KEYS[1], ARGV[1])
server.set_repl(server.REPL_NONE)
server.call('SET', KEYS[2], ARGV[2])
server.set_repl(server.REPL_ALL)
server.call('SET', KEYS[3], ARGV[3])
```

If you run this script by calling `EVAL "..." 3 A B C 1 2 3`, the result will be that only the keys _A_ and _C_ are created on the replicas and AOF.

### <a name="server.replicate_commands"></a> `server.replicate_commands()`

* Since version: 3.2.0
* Until version: 7.0.0
* Available in scripts: yes
* Available in functions: no

This function switches the script's replication mode from verbatim replication to effects replication.
You can use it to override the default verbatim script replication mode used by Valkey until version 7.0.

**Note:**
as of Redis OSS v7.0, verbatim script replication is no longer supported.
The default, and only script replication mode supported, is script effects' replication.
For more information, please refer to [`Replicating commands instead of scripts`](eval-intro.md#replicating-commands-instead-of-scripts)

### <a name="server.breakpoint"></a>  `server.breakpoint()`

* Since version: 3.2.0
* Available in scripts: yes
* Available in functions: no

This function triggers a breakpoint when using the [Valkey Lua debugger](ldb.md).

### <a name="server.debug"></a> `server.debug(x)`

* Since version: 3.2.0
* Available in scripts: yes
* Available in functions: no

This function prints its argument in the [Valkey Lua debugger](ldb.md) console.

### <a name="server.acl_check_cmd"></a> `server.acl_check_cmd(command [,arg...])`

* Since version: 7.0.0
* Available in scripts: yes
* Available in functions: yes

This function is used for checking if the current user running the script has [ACL](acl.md) permissions to execute the given command with the given arguments.

The return value is a boolean `true` in case the current user has permissions to execute the command (via a call to [server.call](#server.call) or [server.pcall](#server.pcall)) or `false` in case they don't.

The function will raise an error if the passed command or its arguments are invalid.

### <a name="server.register_function"></a> `server.register_function`

* Since version: 7.0.0
* Available in scripts: no
* Available in functions: yes

This function is only available from the context of the `FUNCTION LOAD` command.
When called, it registers a function to the loaded library.
The function can be called either with positional or named arguments.

#### <a name="server.register_function_pos_args"></a> positional arguments: `server.register_function(name, callback)`

The first argument to `server.register_function` is a Lua string representing the function name.
The second argument to `server.register_function` is a Lua function.

Usage example:

```
127.0.0.1:6379> FUNCTION LOAD "#!lua name=mylib\n server.register_function('noop', function() end)"
```

#### <a name="server.register_function_named_args"></a> Named arguments:  `server.register_function{function_name=name, callback=callback, flags={flag1, flag2, ..}, description=description}`

The named arguments variant accepts the following arguments:

* _function\_name_: the function's name.
* _callback_: the function's callback.
* _flags_: an array of strings, each a function flag (optional).
* _description_: function's description (optional).

Both _function\_name_ and _callback_ are mandatory.

Usage example:

```
127.0.0.1:6379> FUNCTION LOAD "#!lua name=mylib\n server.register_function{function_name='noop', callback=function() end, flags={ 'no-writes' }, description='Does nothing'}"
```

#### <a name="script_flags"></a> Script flags

**Important:**
Use script flags with care, which may negatively impact if misused.
Note that the default for Eval scripts are different than the default for functions that are mentioned below, see [Eval Flags](eval-intro.md#eval-flags)

When you register a function or load an Eval script, the server does not know how it accesses the database.
By default, Valkey assumes that all scripts read and write data.
This results in the following behavior:

1. They can read and write data.
1. They can run in cluster mode, and are not able to run commands accessing keys of different hash slots.
1. Execution against a stale replica is denied to avoid inconsistent reads.
1. Execution under low memory is denied to avoid exceeding the configured threshold.

You can use the following flags and instruct the server to treat the scripts' execution differently:

* `no-writes`: this flag indicates that the script only reads data but never writes.

    By default, Valkey will deny the execution of flagged scripts (Functions and Eval scripts with [shebang](eval-intro.md#eval-flags)) against read-only replicas, as they may attempt to perform writes.
    Similarly, the server will not allow calling scripts with `FCALL_RO` / `EVAL_RO`.
    Lastly, when data persistence is at risk due to a disk error, execution is blocked as well.

    Using this flag allows executing the script:
    1. With `FCALL_RO` / `EVAL_RO`
    2. On read-only replicas.
    3. Even if there's a disk error (Valkey is unable to persist so it rejects writes).
    4. When over the memory limit since it implies the script doesn't increase memory consumption (see `allow-oom` below)

    However, note that the server will return an error if the script attempts to call a write command.
    Also note that currently `PUBLISH`, `SPUBLISH` and `PFCOUNT` are also considered write commands in scripts, because they could attempt to propagate commands to replicas and AOF file.

    For more information please refer to [Read-only scripts](programmability.md#read-only_scripts)

* `allow-oom`: use this flag to allow a script to execute when the server is out of memory (OOM).

    Unless used, Valkey will deny the execution of flagged scripts (Functions and Eval scripts with [shebang](eval-intro.md#eval-flags)) when in an OOM state.
    Furthermore, when you use this flag, the script can call any Valkey command, including commands that aren't usually allowed in this state.
    Specifying `no-writes` or using `FCALL_RO` / `EVAL_RO` also implies the script can run in OOM state (without specifying `allow-oom`)

* `allow-stale`: a flag that enables running the flagged scripts (Functions and Eval scripts with [shebang](eval-intro.md#eval-flags)) against a stale replica when the `replica-serve-stale-data` config is set to `no` .

    Valkey can be set to prevent data consistency problems from using old data by having stale replicas return a runtime error.
    For scripts that do not access the data, this flag can be set to allow stale Valkey replicas to run the script.
    Note however that the script will still be unable to execute any command that accesses stale data.

* `no-cluster`: the flag causes the script to return an error in Valkey cluster mode.

    Valkey allows scripts to be executed both in standalone and cluster modes.
    Setting this flag prevents executing the script against nodes in the cluster.

* `allow-cross-slot-keys`: The flag that allows a script to access keys from multiple slots.

    Valkey typically prevents any single command from accessing keys that hash to multiple slots.
    This flag allows scripts to break this rule and access keys within the script that access multiple slots.
    Declared keys to the script are still always required to hash to a single slot.
    Accessing keys from multiple slots is discouraged as applications should be designed to only access keys from a single slot at a time, allowing slots to move between Valkey servers.
    
    This flag has no effect when cluster mode is disabled.

Please refer to [Function Flags](functions-intro.md#function-flags) and [Eval Flags](eval-intro.md#eval-flags) for a detailed example.

### <a name="server.server_version"></a> `server.SERVER_VERISON`

* Available in scripts: yes
* Available in functions: yes

Returns the current Valkey server version as a Lua string.
The reply's format is `MM.mm.PP`, where:

* **MM:** is the major version.
* **mm:** is the minor version.
* **PP:** is the patch level.

### <a name="server.redis_version"></a> `server.REDIS_VERSION`

* Available in scripts: yes
* Available in functions: yes

Returns the current Redis compatibility version as a Lua string.
The reply's format is `MM.mm.PP`, where:

* **MM:** is the major version.
* **mm:** is the minor version.
* **PP:** is the patch level.

### <a name="server.redis_version_num"></a> `server.SERVER_VERSION_NUM`

* Available in scripts: yes
* Available in functions: yes

Returns the current Valkey server version as a number.
The reply is a hexadecimal value structured as `0x00MMmmPP`, where:

* **MM:** is the major version.
* **mm:** is the minor version.
* **PP:** is the patch level.

### <a name="server.redis_version_num"></a> `server.REDIS_VERSION_NUM`

* Available in scripts: yes
* Available in functions: yes

Returns the current Redis compatibility version as a number.
The reply is a hexadecimal value structured as `0x00MMmmPP`, where:

* **MM:** is the major version.
* **mm:** is the minor version.
* **PP:** is the patch level.

## Data type conversion

Unless a runtime exception is raised, `server.call()` and `server.pcall()` return the reply from the executed command to the Lua script.
Valkey' replies from these functions are converted automatically into Lua's native data types.

Similarly, when a Lua script returns a reply with the `return` keyword,
that reply is automatically converted to Valkey' protocol.

Put differently; there's a one-to-one mapping between Valkey' replies and Lua's data types and a one-to-one mapping between Lua's data types and the [Valkey Protocol](protocol.md) data types.
The underlying design is such that if a Valkey type is converted into a Lua type and converted back into a Valkey type, the result is the same as the initial value.

Type conversion from Valkey protocol replies (i.e., the replies from `server.call()` and `server.pcall()`) to Lua data types depends on the Redis Serialization Protocol version used by the script.
The default protocol version during script executions is RESP2.
The script may switch the replies' protocol versions by calling the `server.setresp()` function.

Type conversion from a script's returned Lua data type depends on the user's choice of protocol (see the `HELLO` command).

The following sections describe the type conversion rules between Lua and Valkey per the protocol's version.

### RESP2 to Lua type conversion

The following type conversion rules apply to the execution's context by default as well as after calling `server.setresp(2)`:

* [RESP2 integer reply](protocol.md#resp-integers) -> Lua number
* [RESP2 bulk string reply](protocol.md#resp-bulk-strings) -> Lua string
* [RESP2 array reply](protocol.md#resp-arrays) -> Lua table (may have other Valkey data types nested)
* [RESP2 status reply](protocol.md#resp-simple-strings) -> Lua table with a single _ok_ field containing the status string
* [RESP2 error reply](protocol.md#resp-errors) -> Lua table with a single _err_ field containing the error string
* [RESP2 null bulk reply](protocol.md#null-elements-in-arrays) and [null multi bulk reply](protocol.md#resp-arrays) -> Lua false boolean type

## Lua to RESP2 type conversion

The following type conversion rules apply by default as well as after the user had called `HELLO 2`:

* Lua number -> [RESP2 integer reply](protocol.md#resp-integers) (the number is converted into an integer)
* Lua string -> [RESP bulk string reply](protocol.md#resp-bulk-strings)
* Lua table (indexed, non-associative array) -> [RESP2 array reply](protocol.md#resp-arrays) (truncated at the first Lua `nil` value encountered in the table, if any)
* Lua table with a single _ok_ field -> [RESP2 status reply](protocol.md#resp-simple-strings)
* Lua table with a single _err_ field -> [RESP2 error reply](protocol.md#resp-errors)
* Lua boolean false -> [RESP2 null bulk reply](protocol.md#null-elements-in-arrays)

There is an additional Lua-to-Valkey conversion rule that has no corresponding Valkey-to-Lua conversion rule:

* Lua Boolean `true` -> [RESP2 integer reply](protocol.md#resp-integers) with value of 1.

There are three additional rules to note about converting Lua to Valkey data types:

* Lua has a single numerical type, Lua numbers. 
  There is no distinction between integers and floats.
  So we always convert Lua numbers into integer replies, removing the decimal part of the number, if any.
  **If you want to return a Lua float, it should be returned as a string**,
  exactly like Valkey itself does (see, for instance, the `ZSCORE` command).
* There's [no simple way to have nils inside Lua arrays](https://www.lua.org/pil/19.1.html) due 
  to Lua's table semantics.
  Therefore, when Valkey converts a Lua array to RESP, the conversion stops when it encounters a Lua `nil` value.
* When a Lua table is an associative array that contains keys and their respective values, the converted Valkey reply will **not** include them.

Lua to RESP2 type conversion examples:

```
127.0.0.1:6379> EVAL "return 10" 0
(integer) 10

127.0.0.1:6379> EVAL "return { 1, 2, { 3, 'Hello World!' } }" 0
1) (integer) 1
2) (integer) 2
3) 1) (integer) 3
   1) "Hello World!"

127.0.0.1:6379> EVAL "return server.call('get','foo')" 0
"bar"
```

The last example demonstrates receiving and returning the exact return value of `server.call()` (or `server.pcall()`) in Lua as it would be returned if the command had been called directly.

The following example shows how floats and arrays that cont nils and keys are handled:

```
127.0.0.1:6379> EVAL "return { 1, 2, 3.3333, somekey = 'somevalue', 'foo', nil , 'bar' }" 0
1) (integer) 1
2) (integer) 2
3) (integer) 3
4) "foo"
```

As you can see, the float value of _3.333_ gets converted to an integer _3_, the _somekey_ key and its value are omitted, and the string "bar" isn't returned as there is a `nil` value that precedes it.

### RESP3 to Lua type conversion

[RESP3](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md) is a newer version of the [Redis Serialization Protocol](protocol.md).
It is available as an opt-in choice as of Redis OSS v6.0.

An executing script may call the [`server.setresp`](#server.setresp) function during its execution and switch the protocol version that's used for returning replies from Valkey' commands (that can be invoked via [`server.call()`](#server.call) or [`server.pcall()`](#server.pcall)).

Once Valkey' replies are in RESP3 protocol, all of the [RESP2 to Lua conversion](#resp2-to-lua-type-conversion) rules apply, with the following additions:

* [RESP3 map reply](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#map-type) -> Lua table with a single _map_ field containing a Lua table representing the fields and values of the map.
* [RESP set reply](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#set-reply) -> Lua table with a single _set_ field containing a Lua table representing the elements of the set as fields, each with the Lua Boolean value of `true`.
* [RESP3 null](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#null-reply) -> Lua `nil`.
* [RESP3 true reply](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#boolean-reply) -> Lua true boolean value.
* [RESP3 false reply](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#boolean-reply) -> Lua false boolean value.
* [RESP3 double reply](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#double-type) -> Lua table with a single _double_ field containing a Lua number representing the double value.
* [RESP3 big number reply](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#big-number-type) -> Lua table with a single _big_number_ field containing a Lua string representing the big number value.
* [Valkey verbatim string reply](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#verbatim-string-type) -> Lua table with a single _verbatim_string_ field containing a Lua table with two fields, _string_ and _format_, representing the verbatim string and its format, respectively.

**Note:**
the RESP3 [big number](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#big-number-type) and [verbatim strings](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#verbatim-string-type) replies are only supported as of Redis OSS v7.0 and greater. 
Also, presently, RESP3's [attributes](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#attribute-type), [streamed strings](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#streamed-strings) and [streamed aggregate data types](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#streamed-aggregate-data-types) are not supported by the Valkey Lua API.

### Lua to RESP3 type conversion

Regardless of the script's choice of protocol version set for replies with the [`server.setresp()` function] when it calls `server.call()` or `server.pcall()`, the user may opt-in to using RESP3 (with the `HELLO 3` command) for the connection.
Although the default protocol for incoming client connections is RESP2, the script should honor the user's preference and return adequately-typed RESP3 replies, so the following rules apply on top of those specified in the [Lua to RESP2 type conversion](#lua-to-resp2-type-conversion) section when that is the case.

* Lua Boolean -> [RESP3 Boolean reply](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#boolean-reply) (note that this is a change compared to the RESP2, in which returning a Boolean Lua `true` returned the number 1 to the Valkey client, and returning a `false` used to return a `null`.
* Lua table with a single _map_ field set to an associative Lua table -> [RESP3 map reply](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#map-type).
* Lua table with a single _set_ field set to an associative Lua table -> [RESP3 set reply](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#set-type). Values can be set to anything and are discarded anyway.
* Lua table with a single _double_ field to an associative Lua table -> [RESP3 double reply](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#double-type).
* Lua nil -> [RESP3 null](https://github.com/redis/redis-specifications/blob/master/protocol/RESP3.md#null-reply).

However, if the connection is set use the RESP2 protocol, and even if the script replies with RESP3-typed responses, Valkey will automatically perform a RESP3 to RESP2 conversion of the reply as is the case for regular commands.
That means, for example, that returning the RESP3 map type to a RESP2 connection will result in the reply being converted to a flat RESP2 array that consists of alternating field names and their values, rather than a RESP3 map.

## Additional notes about scripting

### Using `SELECT` inside scripts

You can call the `SELECT` command from your Lua scripts, like you can with any normal client connection.
However, one subtle aspect of the behavior changed between Redis OSS versions 2.8.11 and 2.8.12.
Prior to Redis OSS version 2.8.12, the database selected by the Lua script was *set as the current database* for the client connection that had called it.
As of Redis OSS version 2.8.12, the database selected by the Lua script only affects the execution context of the script, and does not modify the database that's selected by the client calling the script.
This semantic change between patch level releases was required since the old behavior was inherently incompatible with Valkey' replication and introduced bugs.

## Runtime libraries

The Valkey Lua runtime context always comes with several pre-imported libraries.

The following [standard Lua libraries](https://www.lua.org/manual/5.1/manual.html#5) are available to use:

* The [_String Manipulation (string)_ library](https://www.lua.org/manual/5.1/manual.html#5.4)
* The [_Table Manipulation (table)_ library](https://www.lua.org/manual/5.1/manual.html#5.5)
* The [_Mathematical Functions (math)_ library](https://www.lua.org/manual/5.1/manual.html#5.6)
* The [_Operating System Facilities (os)_ library](#os-library)

In addition, the following external libraries are loaded and accessible to scripts:

* The [_struct_ library](#struct-library)
* The [_cjson_ library](#cjson-library)
* The [_cmsgpack_ library](#cmsgpack-library)
* The [_bitop_ library](#bitop-library)

### <a name="os-library"></a> _os_ library

* Since version: 8.0.0
* Available in scripts: yes
* Available in functions: yes

_os_ provides a set of functions for dealing with date, time, and system commands.
More details can be found in the [Operating System Facilities](https://www.lua.org/manual/5.1/manual.html#5.8).
Note that for sandbox security, currently only the following os functions is exposed:

* `os.clock()`

### <a name="struct-library"></a> _struct_ library

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: yes

_struct_ is a library for packing and unpacking C-like structures in Lua.
It provides the following functions:

* [`struct.pack()`](#struct.pack)
* [`struct.unpack()`](#struct.unpack)
* [`struct.size()`](#struct.size)

All of _struct_'s functions expect their first argument to be a [format string](#struct-formats).

#### <a name="struct-formats"></a> _struct_ formats

The following are valid format strings for _struct_'s functions:

* `>`: big endian
* `<`: little endian
* `![num]`: alignment
* `x`: padding
* `b/B`: signed/unsigned byte
* `h/H`: signed/unsigned short
* `l/L`: signed/unsigned long
* `T`: size_t
* `i/In`: signed/unsigned integer with size _n_ (defaults to the size of int)
* `cn`: sequence of _n_ chars (from/to a string); when packing, n == 0 means the
  whole string; when unpacking, n == 0 means use the previously read number as
  the string's length.
* `s`: zero-terminated string
* `f`: float
* `d`: double
* ` ` (space): ignored

#### <a name="struct.pack"></a> `struct.pack(x)`

This function returns a struct-encoded string from values.
It accepts a [_struct_ format string](#struct-formats) as its first argument, followed by the values that are to be encoded.

Usage example:

```
127.0.0.1:6379> EVAL "return struct.pack('HH', 1, 2)" 0
"\x01\x00\x02\x00"
```

#### <a name="struct.unpack"></a> `struct.unpack(x)`

This function returns the decoded values from a struct.
It accepts a [_struct_ format string](#struct-formats) as its first argument, followed by encoded struct's string.

Usage example:

```
127.0.0.1:6379> EVAL "return { struct.unpack('HH', ARGV[1]) }" 0 "\x01\x00\x02\x00"
1) (integer) 1
2) (integer) 2
3) (integer) 5
```

#### <a name="struct.size"></a> `struct.size(x)`

This function returns the size, in bytes, of a struct.
It accepts a [_struct_ format string](#struct-formats) as its only argument.

Usage example:

```
127.0.0.1:6379> EVAL "return struct.size('HH')" 0
(integer) 4
```

### <a name="cjson-library"></a> _cjson_ library

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: yes

The _cjson_ library provides fast [JSON](https://json.org) encoding and decoding from Lua.
It provides these functions.

#### <a name="cjson.encode()"></a> `cjson.encode(x)`

This function returns a JSON-encoded string for the Lua data type provided as its argument.

Usage example:

```
127.0.0.1:6379> EVAL "return cjson.encode({ ['foo'] = 'bar' })" 0
"{\"foo\":\"bar\"}"
```

#### <a name="cjson.decode()"></a> `cjson.decode(x)`

This function returns a Lua data type from the JSON-encoded string provided as its argument.

Usage example:

```
127.0.0.1:6379> EVAL "return cjson.decode(ARGV[1])['foo']" 0 '{"foo":"bar"}'
"bar"
```

### <a name="cmsgpack-library"></a> _cmsgpack_ library

* Since version: 2.6.0
* Available in scripts: yes
* Available in functions: yes

The _cmsgpack_ library provides fast [MessagePack](https://msgpack.org/index.html) encoding and decoding from Lua.
It provides these functions.

#### <a name="cmsgpack.pack()"></a> `cmsgpack.pack(x)`

This function returns the packed string encoding of the Lua data type it is given as an argument.

Usage example:

```
127.0.0.1:6379> EVAL "return cmsgpack.pack({'foo', 'bar', 'baz'})" 0
"\x93\xa3foo\xa3bar\xa3baz"
```

#### <a name="cmsgpack.unpack()"></a> `cmsgpack.unpack(x)`

This function returns the unpacked values from decoding its input string argument.

Usage example:

```
127.0.0.1:6379> EVAL "return cmsgpack.unpack(ARGV[1])" 0 "\x93\xa3foo\xa3bar\xa3baz"
1) "foo"
2) "bar"
3) "baz"
```

### <a name="bitop-library"></a> _bit_ library

* Since version: 2.8.18
* Available in scripts: yes
* Available in functions: yes

The _bit_ library provides bitwise operations on numbers.
Its documentation resides at [Lua BitOp documentation](https://bitop.luajit.org/api.html)
It provides the following functions.

#### <a name="bit.tobit()"></a> `bit.tobit(x)`

Normalizes a number to the numeric range for bit operations and returns it.

Usage example:

```
127.0.0.1:6379> EVAL 'return bit.tobit(1)' 0
(integer) 1
```

#### <a name="bit.tohex()"></a> `bit.tohex(x [,n])`

Converts its first argument to a hex string. The number of hex digits is given by the absolute value of the optional second argument.

Usage example:

```
127.0.0.1:6379> EVAL 'return bit.tohex(422342)' 0
"000671c6"
```

#### <a name="bit.bnot()"></a> `bit.bnot(x)`

Returns the bitwise **not** of its argument.

#### <a name="bit.ops"></a> `bit.bnot(x)` `bit.bor(x1 [,x2...])`, `bit.band(x1 [,x2...])` and `bit.bxor(x1 [,x2...])`

Returns either the bitwise **or**, bitwise **and**, or bitwise **xor** of all of its arguments.
Note that more than two arguments are allowed.

Usage example:

```
127.0.0.1:6379> EVAL 'return bit.bor(1,2,4,8,16,32,64,128)' 0
(integer) 255
```

#### <a name="bit.shifts"></a> `bit.lshift(x, n)`, `bit.rshift(x, n)` and `bit.arshift(x, n)`

Returns either the bitwise logical **left-shift**, bitwise logical **right-shift**, or bitwise **arithmetic right-shift** of its first argument by the number of bits given by the second argument.

#### <a name="bit.ro"></a> `bit.rol(x, n)` and `bit.ror(x, n)`

Returns either the bitwise **left rotation**, or bitwise **right rotation** of its first argument by the number of bits given by the second argument.
Bits shifted out on one side are shifted back in on the other side.

#### <a name="bit.bswap()"></a> `bit.bswap(x)`

Swaps the bytes of its argument and returns it.
This can be used to convert little-endian 32-bit numbers to big-endian 32-bit numbers and vice versa.
