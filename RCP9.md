Redis Lua debugger
===

````
Author: Itamar Haber <itamar@redislabs.com>
Creation date: 2015-10-21
Update date: 2015-11-01
Status: wip
Version: 1.0.1
Implementation: none
````

History
---

* Version 1.0 (2015-10-26): Initial version.
* Version 1.0.1 (2015-11-01): Internal review - conceptual API fleshout, clearer implementation guidelines.

Rationale
---

As of v2.6, Redis includes the ability for executing server-side Lua scripts. It is a shame, however, that there is no sane way to develop such scripts since tracing their execution, let alone debugging, is taxing.

This proposal describes how a Lua debugger can be added to the Redis project to make Lua scripts' development fun.

Current state of developing Lua scripts for Redis
---

Two commonly used techniques that are employed by software engineers during development are [tracing][1] and [debugging][2].

There are multiple ways to trace the execution of Lua scripts running in Redis. First and foremost, the [`redis.log`][3] function is a convenient method for outputting trace information to the log file. Other approaches include using Redis Lists, PubSub or `MONITOR` & `ECHO` as described in [this post][4]. While tracing can be a useful tool, there are several disadvantages in using it with Redis Lua scripts anything other than simple development purposes. Most notably trace commands have to be explicitly added to the code, thus requiring effort and reducing code readability, and have to be disabled/commented out/stripped away when the code is deployed in production.

Lua provides the [`debug`][5] library that _"does not give you a debugger for Lua, but it offers all the primitives that you need for writing a debugger for Lua."_ Redis' Lua engine already includes the `debug` library and implements user scripts' error handling with it. [redis-lua-debugger][6] is a Lua script that uses the `debug` library to provide an automated tracing and profiling of a user script's execution in an attempt to address some of the above mentioned disadvantages. However, in response to [CVE-2015-4335][7] and as of [Redis v3][8] user scripts have been denied access to the `debug` library, thus rendering tools such as redis-lua-debugger useless.

Lastly, a [workflow][9] that simulates Redis' embedded Lua runtime has been developed to allow interactive debugging of scripts outside Redis. This approach is inspiring, especially considering the effort it took to develop it. However, not only does it requires some set up but, more importantly, it does not guarantee full compatibility with the project's actual behavior.

Commands introduced
---

    DEVAL script numkeys key [key ...] arg [arg ...]

The `DEVAL` command (Debug + EVAL, and because the devil's in the details) initiates a debug session for the specified script. The session begins after the script is set up for execution and before its first line is executed. When put in debug as well as upon returning to debug mode from executing the script (e.g. when encountering a breakpoint, after stepping into/over/out, ...), the reply would be the current line of code where execution stopped. The following example demonstrates a simple debug session:

````
127.0.0.1:6379> DEVAL "return 42" 0
"1"
127.0.0.1:6379 (debug)> RUN
"OK"
127.0.0.1:6379 (debug)> GETRETURN
"42"
127.0.0.1:6379 (debug)> END
"42"
127.0.0.1:6379>
````

During a debug session, the user can only use the following debug-specific commands.

### Debugger control

    RUN [line [line...]]

Runs the script until it exits, or a breakpoint is encountered. Specifying one or more line numbers is the equivalent of creating breakpoints for these lines. Possible replies:

* If the script exits normally (end of script or `return` statement), the return value is `OK`
* If the script errors, the return value is the error raised
* If the script breaks, the return value is the relevant breakpoint-id

A smart debugging client will automatically call `GETRETURN` when getting the `OK` to display the script's return value after successful execution. Similarly, the smart ones will call `BLIST`  with the relevant breakpoint-id and/or `GETCODE` to provide the context after breaking.

    RESTART [FLUSH]

Restarts the execution of the debugged script while keeping the debug session context intact (watches, breakpoints). Given the `FLUSH` switch, the context is reset as well. This command never fails :) Should be followed by `RUN` for example.

    STEPINTO / STEPOVER / STEPOUT

Breaks on the execution of the next statement, stepping into/over/out of function calls. Returns the current line number.

    END

Ends the debugging session, but completes the script's execution before. Any breakpoints are ignored. Returns the script's return value or an error if raised.

    ABORT

Aborts the debugging session, halting the script's execution if it is still ongoing.

### Inspection commands

    TRON [LINE | [CODE | STACK | WATCH | ALL]]

Turns on tracing, namely returning information about the steps of execution. The default `LINE` switch traces the line number. When `CODE`, `STACK` or `WATCH` are used, the reply consists of the current LoC number and Lua code, stack dump or the evaluated watch list, respectively . `ALL` means just that. When called the first time, it returns the requested information and repeats that for every step until `TROFF`, `QUIT` or `ABORT` are used.

    TROFF

Turns off tracing.

    GETCODE [line]

Returns the actual Lua code at the specified line. If no line is given, the current line is returned.

    GETRETURN

Returns the debugged script return value, unless the script is still running, in which case -ERR is returned.

    GETSTACK

Returns a stack trace [TBD].

    DOEVAL expression

Evaluates the expression in the current Lua context and returns its value or an error.

    DOPRINT expression

Just like `DOEVAL` but returns a pretty print of the value (but the error remains ugly). Pretty print of values mainly means table handling.

    DOEXEC statement

Executes the statement in the current Lua context. Returns "OK" or an error.

### Watches management

    WADD "expression" ["expression"...]

Adds a watched expression(s). Returns the watch-id(s), which is(are) the sequence number(s) of the added watch(s) in the list of watches (zero-based of course).

    WREM watch-id [watch-id...]

Removes watch(es) from the list by id. If watch is also a breakpoint, removes it from the breakpoints list as well. Reply is the new length of the watches list.

    WCOUNT

Returns the length of the watches list.

    WEVAL [watch-id [watch-id...]]

Evaluates all (default) or specific watch-ids and returns their results.

    WLIST [start end] [RAW]

Returns the list of watched expressions as an array. Allows ranges using the usual conventions and by default performs 0 -1. When the `RAW` switch is used, output is a single string suitable for using with the `WADD` command

    WFLUSH

Clears the watches list.

### Breakpoint/watchpoint management

    BADD [[LINE] line [line...] | WATCH expression [expression...]]

Adds one or more breakpoints, returns the breakpoint id(s). Breakpoints come in two flavors: line and watch. A line breakpoint (the default, also explicitly definable with the `LINE` switch) is a line number that will cause the debugger to stop when it reaches it during the script's execution. A watch breakpoint (a.k.a watchpoint) is explicitly added with the `WATCH` switch and are Lua expressions. Watch breakpoints are evaluated at every step of the script's execution and break when true .

    BREM breakpoint-id [breakpoint-id...]

Removes breakpoint(s) from the list by id. Reply is the new length of the breakpoints list.

    BON / BOFF [breakpoint-id [breakpoint-id...]]

Turns all (the default) or specific breakpoints on / off.

    BCOUNT

Returns the length of the breakpoints list.

    BLIST [start end] [RAW]

Returns the list of breakpoints. Allows ranges using the usual conventions and by default performs 0 -1. When the `RAW` switch is used, output is an array with two strings, the first for line breakpoints and the second for watch breakpoints, where each string is suitable for using with the `BADD` command

    BFLUSH

Clears the breakpoints list.

### Redis Lua debug API

**Note:** these inline debugger instructions should resolve to NOOP, unless in executed in a debug session.

    redis.debugmode(enabled = True)

Is used to disable the debugger (equivalent to issuing `END`) when used with a `False`. Can be used to re-enable the debugger again before the script ends.

    redis.debugbreak([expression])

Breaks the execution of the script on that line if `expression` is true or if none provided.

    redis.debugtrace(redis.traceall | redis.traceline | redis.tracecode | redis.tracestack | redis.traceoff)

Is the equivalent of issuing `TRON` with the respective switch, or `TROFF`.

    redis.debugeval(expression)

Is the equivalent of calling `DOEVAL` with the expression.

    redis.debugprint(expression)

Is the equivalent of calling `DOPRINT` with the expression.

Implementation details/guidelines
---

* External dependencies are usually not worth it.
* Redis core should provide an entire debugging environment (that does not mean, however, that an external UI can't be developed/adapted for it - we provide the tools...)
* Avoid sandbox pollution & heisenbugs by implementing the debugger outside the Lua VM.
* Debugger could be implemented in its own Lua VM or in C.
* Teach redis-cli some debugging finesse.
* Debug sessions are tricky in the context of an operational Redis server. Because a debug session is highly blocking, special consideration should be given to the following topics:
 * Replication: the default `REPL_ALL` mode isn't likely to tolerate an loose random debugger. We could possibly enforce the `REPL_NONE` (or `REPL_AOF`) mode, but that may result in different behaviors in development and production. Alternatively, debug could be disabled if the instance has any slaves connected.
 * Connected clients
 * High availability in general and Sentinel in particular
 * Redis cluster
* Caching debugged scripts appears to be of little or no value.
* Slow script protection may need to be disabled for debug sessions, but `SCRIPT KILL` should do the trick regardless.

 [1]: https://en.wikipedia.org/wiki/Tracing_(software) "Tracing (software)"
 [2]: https://en.wikipedia.org/wiki/Debugging "Debugging"
 [3]: http://redis.io/commands/eval#emitting-redis-logs-from-scripts "EVAL: Emitting Redis logs from scripts"
 [4]: https://redislabs.com/blog/5-methods-for-tracing-and-debugging-redis-lua-scripts "5 Methods for Tracing and Debugging Redis Lua Scripts"
 [5]: http://www.lua.org/pil/23.html "The Debug Library"
 [6]: https://github.com/redislabs/redis-lua-debugger "redis-lua-debugger"
 [7]: https://redislabs.com/blog/cve-2015-4335-dsa-3279-redis-lua-sandbox-escape "Redis Lua Sandbox Escape"
 [8]:  https://github.com/antirez/redis/commit/30278061cc834b4073b004cb1a2bfb0f195734f7 "commit 30278061cc834b4073b004cb1a2bfb0f195734f7: hide access to debug table"
 [9]: http://www.trikoder.net/blog/redis-and-lua-scripts-new-workflow-89 "Redis and Lua scripts - new workflow"
