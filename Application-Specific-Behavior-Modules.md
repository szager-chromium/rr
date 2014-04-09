For issue #1044 and others it might be a good idea to add application-specific configuration to rr. This could include

* default treatment of certain signal handlers
* application-specific overrides of various heuristics
** which files get copied or don't get copied

Proposal: Let the filename of the binary image be F. Then search ~/.rr/behaviors/F.ini and path_to_rr_binary/../behaviors/F.ini (the latter being a set of configurations that is bundled with rr and lives in the rr github repo). Use a simple INI parser like https://code.google.com/p/inih/.

Each feature gets a section. So for signal handlers, we can write
    [sighandlers]
    AsmJSFaultHandler = continue
to indicate that when AsmJSFaultHandler is invoked as a signal handler, we should continue without breaking to the debugger. We'd allow the value "stop" (the default) to override this behavior, so users can override it for particular functions.