Protips both for debugging rr and tracees.

**Debug logging**: all .cc files (should) have a line near the top that looks like
<pre>
//#define DEBUGTAG "Sched"
//...
#include "dbg.h"
</pre>

Uncomment the definition and you'll get full, verbose, debug-level logging output.

**Send logging to non-default FILE**: in the .cc file you want to redirect
<pre>
static FILE* locallog = fopen("/tmp/rr-sched.log", "w");
#define LOG_FILE locallog
//...
#include "dbg.h"
</pre>

**Use assert_exec() to launch a gdbserver for a tracee**: if you want to debug a tracee `t`, add a call like the following `assert_exec(t, false, "")`.  A gdbserver will launch for `t` with a message like
<pre>
[EMERGENCY] (file:line:function: errno: None) (task X (rec:Y) at trace line Z)
 -> Assertion `false' failed to hold: '
(rr debug server listening on :X)
</pre>
A `gdb tracee-program` then `(gdb) target remote :X` will attach to the gdbserver.

**Lookup a Task**: whose (recorded!) tid you know*: in a debugger `(gdb) p Task::find([tid])`.

**Launch a gdbserver for arbitrary tasks at arbitrary times**: sometimes there's not a single point at which you want to launch the gdbserver.  You can do this by attaching a debugger to rr and then manually starting a gdbserver for a given task
<pre>
$ gdb -p $(pidof rr)
...
(gdb) p 'Task::find'([tid])
$1 = (Task *) 0x818c1c8
(gdb) call emergency_debug($1)
(rr debug server listening on :X)
</pre>
Then in another shell, follow the instructions above for attaching to a tracee.

**Iterate through all tracee Tasks**: `Task` has a helper method `next_roundrobin()` that returns a successor task in round-robin order, meaning each task is cycled through circularly.  So if you can locate a single `Task*` in a debugger, then you can find all the others as well by successive calls to `t->next_roundrobin()`.