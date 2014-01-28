Protips both for debugging rr and tracees.

#### Debug logging
All .cc files (should) have a line near the top that looks like
<pre>
//#define DEBUGTAG "Sched"
//...
#include "dbg.h"
</pre>

Uncomment the definition and you'll get full, verbose, debug-level logging output.

#### Send logging to non-default FILE
In the .cc file you want to redirect
<pre>
static FILE* locallog = fopen("/tmp/rr-sched.log", "w");
#define LOG_FILE locallog
//...
#include "dbg.h"
</pre>

#### Be sure to load the right executable image for gdb'ing a tracee
rr doesn't implement gdb multi-process support yet, so you're using 1980s/1990s-era debugging technology.  If you `gdb the-wrong-image`, gdb will get very confused.  If you're trying to attach to a tracee after seeing
<pre>(rr debug server listening on :X)</pre>
then you can use
<pre>ls -l /proc/X/exe</pre>
to print the tracee's executable image.

#### Dump a full tracee tree with pstree
Useful for many things.  For example
<pre>
$ pstree -p $(pidof rr)
rr(1969)───firefox(1975)─┬─Browser(2178)─┬─{Browser}(2179)
                         │               ├─{Browser}(2180)
                         │               ├─{Browser}(2181)
                         │               ├─{Browser}(2182)
                         │               ├─{Browser}(2183)
                         │               ├─{Browser}(2184)
                         │               ├─{Browser}(2185)
                         │               ├─{Browser}(2186)
                         │               ├─{Browser}(2187)
                         │               ├─{Browser}(2188)
                         │               ├─{Browser}(2189)
                         │               ├─{Browser}(2190)
                         │               ├─{Browser}(2191)
                         │               └─{Browser}(2192)
                         ├─{firefox}(1982)
                         ├─{firefox}(1983)
                         ├─{firefox}(1984)
                         ├─{firefox}(1985)
                         ├─{firefox}(1986)
                         ├─{firefox}(1987)
                         ├─{firefox}(1988)
                         ├─{firefox}(1989)
                         ├─{firefox}(1990)
                         ├─{firefox}(1991)
                         ├─{firefox}(1992)
                         ├─{firefox}(1993)
                         ├─{firefox}(1994)
                         ├─{firefox}(1995)
                         ├─{firefox}(1996)
                         ├─{firefox}(1997)
                         ├─{firefox}(1998)
                         ├─{firefox}(2000)
                         ├─{firefox}(2001)
                         ├─{firefox}(2002)
                         ├─{firefox}(2003)
                         ├─{firefox}(2004)
                         ├─{firefox}(2005)
                         ├─{firefox}(2006)
                         ├─{firefox}(2008)
                         ├─{firefox}(2010)
                         ├─{firefox}(2011)
                         ├─{firefox}(2012)
                         ├─{firefox}(2013)
                         ├─{firefox}(2014)
                         ├─{firefox}(2016)
                         └─{firefox}(2115)
</pre>
Listed tasks that are in `{curly-brackets}(tid)` are clone children, aka threads. Other tasks listed `not-in-curly-brackets(tid)` are fork children, aka subprocesses.

#### Dump attributes of tracee tasks in rr debugger session
`(gdb) call 'Task::dump_all'(0)` dumps information about all known tracees to stderr.  F.e.
<pre>
Task<0x81bae80>(tid:7373 rec_tid:7373 status:0x857f)
[INFO] (log_event) SYSCALL: futex
</pre>
Or to dump a specific task, `(gdb) call r->dump(0)`.

#### Force rr to always launch a gdbserver, even if it thinks that's a bad idea
Sometimes you'll be debugging and see output like
<pre>
[FATAL] (/home/cjones/rr/rr/src/replayer/replayer.cc:1729:emergency_debug: errno: None) (trace line 297399)
 -> (session doesn't look interactive, aborting emergency debugging)
</pre>
This is to guard against rr deadlocking when run from a script, where the user may not be able to see that an rr assertion has failed.  For unit test scripts that tee output, e.g., where the user can see it but rr's can-the-user-see-this heuristic fails, this behavior is not at all what you want.  Pass the command line flag `rr -f ...` to override the heuristic and always launch a gdbserver when requested. 

#### Use assert_exec() to launch a gdbserver for a tracee
If you want to debug a tracee `t` a known point in rr code, add a call like the following `assert_exec(t, false, "")`.  A gdbserver will launch for `t` with a message like
<pre>
[EMERGENCY] (file:line:function: errno: None) (task X (rec:Y) at trace line Z)
 -> Assertion `false' failed to hold: '
(rr debug server listening on :X)
</pre>
A `gdb tracee-program` then `(gdb) target remote :X` will attach to the gdbserver.

#### Look up a Task whose (recorded!) tid you know
In your debugger `(gdb) p Task::find([tid])`.

#### Launch a gdbserver for arbitrary tasks at arbitrary times
Sometimes there's not a single point at which you know you'll want to launch the gdbserver.  This often happens in live-lock-esque scenarios.  You can do this by attaching a debugger to rr and then manually starting a gdbserver for a given task
<pre>
$ gdb -p $(pidof rr)
...
(gdb) p 'Task::find'([tid])
$1 = (Task *) 0x818c1c8
(gdb) call start_debug_server($1)
(rr debug server listening on :X)
</pre>
Then in another shell, follow the instructions above for attaching to a tracee.

#### Iterate through all tracee Tasks
`Task` has a helper method `next_roundrobin()` that returns a successor task in round-robin order, meaning each task is cycled through circularly.  So if you can locate a single `Task*` in a debugger, then you can find all the others as well by successive calls to `t->next_roundrobin()`.

#### You may find this helper repository useful
[cgjones/rr-workbench](https://github.com/cgjones/rr-workbench): collection of helpers for automating rr tasks (especially running FF unit tests), and for random things like stringifying `waitpid` status codes (status2text).  It also contains an llvm Bell-Larus path logger and some proof-of-concept programs that have been written to prototype rr features.