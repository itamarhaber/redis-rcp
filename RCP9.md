Redis Lua debugger
===

````
Author: Itamar Haber <itamar@redislabs.com>
Creation date: 2015-10-21
Update date: 2015-10-26
Status: open
Version: 1.0
Implementation: none
````

History
---

* Version 1.0 (2015-10-26): Initial version.

Rationale
---

As of v2.6, Redis includes the ability for executing server-side Lua scripts. It is a shame, however, that there is no sane way to develop such scripts since tracing their execution, let alone debugging, is taxing.

This proposal describes how a full-blown Lua debugger can be integrated in the Redis project to make Lua scripts' development and maintenance fun.

Current state of developing Lua scripts for Redis
---

Two commonly used techniques that are employed by software engineers during development are [tracing][1] and [debugging][2].

There are multiple ways to trace the execution of Lua scripts running in Redis. First and foremost, the [`redis.log`][3] function is a convenient method for outputting trace information to the log file. Other approaches include using Redis Lists, PubSub or `MONITOR` & `ECHO` as described in [this post][4]. While tracing can be a useful tool, there are several disadvantages in using it with Redis Lua scripts anything other than simple development purposes. Most notably trace commands have to be explicitly added to the code, thus requiring effort and reducing code readability, and have to be disabled/commented out/stripped away when the code is deployed in production.

Lua provides the [`debug`][5] library that _"does not give you a debugger for Lua, but it offers all the primitives that you need for writing a debugger for Lua."_ Redis' Lua engine already includes the `debug` library and implements user scripts' error handling with it. [redis-lua-debugger][6] is a Lua script that uses the `debug` library to provide an automated tracing and profiling of a user script's execution in an attempt to address some of the above mentioned disadvantages. However, in response to [CVE-2015-4335][7] and as of [Redis v3][8] user scripts have been denied access to the `debug` library, thus rendering tools such as redis-lua-debugger useless.

Lastly, a [workflow][9] that simulates Redis' embedded Lua runtime has been developed to allow interactive debugging of scripts outside Redis. This approach is inspiring, especially considering the effort it took to develop it. However, not only does it requires some set up but, more importantly, it does not guarantee full compatibility with the project's actual behavior.

Commands introduced
---

    SCRIPT DEBUG [LOCAL|REMOTE host port] script numkeys key [key ...] arg [arg ...]

The `SCRIPT DEBUG` command initiates a debug client for the specified script. The `LOCAL` directive (default) indicates that that debug server will be run inside the Redis server and that current connection will be used for its IO. By specifying a remote server, the debug client will connect to that server.

Implementation details
---

The implementation relies on the integration of [MobDebug][10], a remote debugger for Lua, into Redis. MobDebug uses the `debug` and [`luasocket`][11] libraries to expose an interactive debugger interface (via its server) into a running Lua script (the client). In addition MobDebug is integrated with [ZeroBrane Studio][12], a popular Lua IDE by the same author.

### Debug client

For a script to be a debug client, whether debugged locally or remotely, the following should be staged:
* Both `mobdebug.lua` and `luasocket` are required
* [TODO: verify] The vanilla debug library should be available: `_G.debug = require("debug")`
* Call the `start` function (`require('mobdebug').start("remote-server-address")`)

#### MobDebug client API

MobDebug provides the following functions to use from the client:
* start - starts a debugging session (should only be called once at the beginning)
* loop - puts the client in an idle loop until the server sends a script for execution
* off - toggles debugging off
* on - toggles debugging on

Toggling debugging is useful and could be present in production scripts. An abstraction over MobDebug will keep the Redis Lua semantics consistent and facilitate executing the toggle or NOOP in the context of debugging or normal execution, respectively. Proposed semantics:
* redis.debug-start("remote-server-address") - hidden, auto-injected for debug sessions
* redis.debug-loop()
* redis.debug-off()
* redis.debug-on()

### Remote debug server

A remote debug server can be any MobDebug-compatible server such as running `require("mobdebug").listen()` or ZeroBrane Studio's.

### Local debug server

A local debug server in Redis is extremely interesting as it will allow debugging Lua scripts without the need for additional 3rd party tools. As Redis already ships with an embedded Lua engine, running the debug server in it makes more sense than, for example, adding Lua to redis-cli just for that purpose.

A *rough* outline for implementing the local debug server in Redis is as follows:
1. Create a Lua context for the debug server
2. In the debug server's context run `require("mobdebug").listen()`
3. Figure out how to connect the client and server via their sockets
4. Route the server's output to the client running `SCRIPT DEBUG` and the input to the client's messages

Open topics
---
* Debugged scripts should not be cached
* Debugged scripts can break replication
* Slow script protection needs to be disabled for debug sessions
* RESP may need to be tweaked to sustain the LOCAL mode back and forth
* redis-cli needs to be taught about "SCRIPT DEBUG LOCAL" mode
* luasocket in Redis Lua... interesting, should it remain exposed?
* Should the local debug socket be configurable or randomly chosen by Redis?

 [1]: https://en.wikipedia.org/wiki/Tracing_(software) "Tracing (software)"
 [2]: https://en.wikipedia.org/wiki/Debugging "Debugging"
 [3]: http://redis.io/commands/eval#emitting-redis-logs-from-scripts "EVAL: Emitting Redis logs from scripts"
 [4]: https://redislabs.com/blog/5-methods-for-tracing-and-debugging-redis-lua-scripts "5 Methods for Tracing and Debugging Redis Lua Scripts"
 [5]: http://www.lua.org/pil/23.html "The Debug Library"
 [6]: https://github.com/redislabs/redis-lua-debugger "redis-lua-debugger"
 [7]: https://redislabs.com/blog/cve-2015-4335-dsa-3279-redis-lua-sandbox-escape "Redis Lua Sandbox Escape"
 [8]:  https://github.com/antirez/redis/commit/30278061cc834b4073b004cb1a2bfb0f195734f7 "commit 30278061cc834b4073b004cb1a2bfb0f195734f7: hide access to debug table"
 [9]: http://www.trikoder.net/blog/redis-and-lua-scripts-new-workflow-89 "Redis and Lua scripts - new workflow"
 [10]: https://github.com/pkulchenko/MobDebug "MobDebug"
 [11]: https://github.com/diegonehab/luasocket "luasocket"
 [12]: http://studio.zerobrane.com/doc-remote-debugging "ZeroBrane Studio - Remote Debugging"
