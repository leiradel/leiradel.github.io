---
layout: post
title: Embedding Lua Modules in C
---

Lua provides the [`luaL_requiref`](https://www.lua.org/manual/5.3/manual.html#luaL_requiref) function to allow the embedding of modules, either native or written in Lua, into the executable instead of relying on the file system and [search paths](https://www.lua.org/manual/5.3/manual.html#pdf-package.path) to locate and load modules into a Lua state.

Besides the Lua state, `luaLrequiref` needs the module name as a C string, and a `lua_CFunction` argument that is responsible for providing the module. Its signature is:

```c
LUALIB_API void luaL_requiref (lua_State *L, const char *modname,
                               lua_CFunction openf, int glb);
```

## Tired: `luaLrequiref`

`luaLrequiref` works by checking if a module with the name passed in `modname` was already loaded and, if not, by calling the `openf` function and setting the module as the result of that call. It can also set a global variable with `modname` with the result of the call when `glb` is not zero:

```c
/*
Requires the socket core module. If a module named "socket.core" was already
loaded, it just returns the previously loaded module. If not, it calls the
provided function and sets the module as the result of that call.
*/
luaL_requiref(L, "socket.core", luaopen_socket_core, 0);

/* Removes the copy of the module from the stack. */
lua_pop(L, 1);
```

To use `luaL_requiref` with modules written in Lua, an intermediary function to compile the Lua source code is needed. Since `luaL_requiref` passes `modname` to `openf`, it's trivial to write such function:

```c
/* Our Lua module, which has only the writeobj function. */
static const char writemod[] =
    "return {\n"
    "    writeobj = function(obj)\n"
    "        io.write(tostring(obj))\n"
    "    end\n"
    "}\n";

static int openf(lua_State* L) {
    /* Get the module name as passed to luaL_requiref. */
    char const* const modname = lua_tostring(L, 1);

    int res;

    /*
    Check if we know the module. We can use this function to load many
    different Lua modules uniquely identified by modname.
    */
    if (strcmp(modname, "write") == 0) {
        /*
        Parses the Lua source code and leaves the compiled function on the top
        of the stack if there are no errors.
        */
        res = luaL_loadbufferx(L,
                               writemod,
                               sizeof(writemod) - 1,
                               "write",
                               "t");
    }
    else {
        /* Unknown module. */
        return luaL_error(L, "unknown module \"%s\"", modname);
    }

    /* Check if the call to luaL_loadbufferx was successful. */
    if (res != LUA_OK) {
        return lua_error(L);
    }

    /*
    Runs the Lua code and returns whatever it returns as the result of openf,
    which will be used as the value of the module.
    */
    lua_call(L, 0, 1);
    return 1;
}

int main() {
    lua_State* L = luaL_newstate();
    luaL_openlibs(L);
    luaL_requiref(L, "write", openf, 0);

    luaL_dofile(
        L,
        "local write = require 'write'\n"
        "write.writeobj(_G)\n"
    );

    lua_close(L);
    return EXIT_SUCCESSFUL;
}
```

This is all easy and works as intended, but introduces a dependency problem when many modules are embedded, as some modules may depend on other modules. The problem is that when a module is required via `luaL_requiref`, all the modules that it [`require`](https://www.lua.org/manual/5.3/manual.html#pdf-require)s must be already known to the Lua state, otherwise `require` will fail to load the module.

The only way around this issue is calling `luaL_requiref` in a order that satisfies these dependencies, so that when a module is loaded, all modules that are dependencies of that module were already loaded.

Although the correct order of the `luaL_requiref` calls is not difficult to achieve, another problem with this approach is that all modules are forcibly loaded even though the Lua code ends up not requiring all of them.

## Wired: register our own searcher

In Lua, a searcher is a function that is responsible for locating a module by its name and loading it. When created, a Lua state already comes with [four searches defined](https://www.lua.org/manual/5.3/manual.html#pdf-package.searchers) by default.

It's possible to add new searchers to the list and load modules in ways not supported by the default loaders. So instead of using `luaL_requiref` to pre-load all modules that we want to embed in the application, we can register a search that knows how to locate and load them.

The code below registers a searcher that can load any module from the LuaSocket library:

```c
#include <stddef.h>
#include <string.h>

#include <lua.h>
#include <lauxlib.h>

/*
These includes were generated from their respective Lua source code using xxd
and sed.
*/
#include "luasocket/src/ftp.lua.h"
#include "luasocket/src/headers.lua.h"
#include "luasocket/src/http.lua.h"
#include "luasocket/src/ltn12.lua.h"
#include "luasocket/src/mbox.lua.h"
#include "luasocket/src/mime.lua.h"
#include "luasocket/src/smtp.lua.h"
#include "luasocket/src/socket.lua.h"
#include "luasocket/src/tp.lua.h"
#include "luasocket/src/url.lua.h"

/*
Declare the functions that open the mime.core and socket.core native modules.
*/
int luaopen_mime_core(lua_State*);
int luaopen_socket_core(lua_State*);

typedef struct {
    const char* name;

    union {
        const char* source;
        lua_CFunction openf;
    };

    size_t length;
}
module_t;

/*
Adds a Lua module to the modules array (length = length of the Lua source
code).
*/
#define MODL(name, array) {name, {array}, sizeof(array)}

/* Adds a native module to the modules array (length = 0). */
#define MODC(name, openf) {name, {(char*)openf}, 0}

static const module_t modules[] = {
    MODL("ltn12", socket_ltn12_lua),
    MODL("mbox", socket_mbox_lua),
    MODL("mime", socket_mime_lua),
    MODC("mime.core", luaopen_mime_core),
    MODL("socket", socket_socket_lua),
    MODC("socket.core", luaopen_socket_core),
    MODL("socket.ftp", socket_ftp_lua),
    MODL("socket.headers", socket_headers_lua),
    MODL("socket.http", socket_http_lua),
    MODL("socket.smtp", socket_smtp_lua),
    MODL("socket.tp", socket_tp_lua),
    MODL("socket.url", socket_url_lua)
};

#undef MODL
#undef MODC

static int searcher(lua_State* const L) {
    /* Get the module name. */
    const char* const modname = lua_tostring(L, 1);

    /* Iterates over all modules we know. */
    for (size_t i = 0; i < sizeof(modules) / sizeof(modules[0]); i++) {
        if (strcmp(modname, modules[i].name) == 0) {
            /* Found the module! */
            if (modules[i].length != 0) {
                /*
                It's a Lua module, return the chunk that defines the module.
                */
                const int res = luaL_loadbufferx(L,
                                                 modules[i].source,
                                                 modules[i].length,
                                                 modname,
                                                 "t");

                if (res != LUA_OK) {
                    /* Compilation error. */
                    return lua_error(L);
                }
            }
            else {
                /*
                It's a native module, return the native function that defines
                the module.
                */
                lua_pushcfunction(L, modules[i].openf);
            }

            return 1;
        }
    }

    /* Oops... */
    lua_pushfstring(L, "unknown module \"%s\"", modname);
    return 1;
}

/* Registers the searcher function. */
void registersearcher(lua_State* const L) {
    /* Get the package global table. */
    lua_getglobal(L, "package");
    /* Get the list of searchers in the package table. */
    lua_getfield(L, -1, "searchers");
    /* Get the number of existing searchers in the table. */
    const size_t length = lua_rawlen(L, -1);

    /* Add our own searcher to the list. */
    lua_pushcfunction(L, searcher);
    lua_rawseti(L, -2, length + 1);

    /* Remove the seachers and the package tables from the stack. */
    lua_pop(L, 2);
}
```

Now when someone `require`s any LuaSocket module, our searcher function will end up being called, and will return the function that defines the module. If that module in its turn `require`s another module, the searcher function will be called again and all the dependencies will be satisfied without requiring manual ordering of the dependencies.

Also, only the modules effectively used by the Lua code will be loaded into the Lua state.
