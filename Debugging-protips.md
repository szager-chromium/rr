Protips both for debugging rr and tracees.

#### Debug logging
Run RR as follows:
<pre>
RR_LOG=ClassName,ClassName2 rr ./executable # ClassName is the name of the class(es) where you would like to enable debug info.
</pre>

To enable redirection to a file:
<pre>
RR_LOG_FILE=path/to/log RR_LOG=ClassName rr ./executable
</pre>

Example:
<pre>
RR_LOG_FILE=/tmp/rr_log.log RR_LOG=ReplaySession rr replay
</pre>

#### Send logging to non-default FILE
In the .cc file you want to redirect
<pre>
#include &lt;stdio.h&gt;
static FILE* locallog = fopen("/tmp/rr-sched.log", "w");
#define LOG_FILE locallog
//...
#include "dbg.h"
</pre>

#### Dump a trace event or range of events

It's often useful to see raw recorded event information.  For example, if you see a failure like
<pre>
[EMERGENCY] (/home/cjones/rr/rr/src/share/util.cc:628:compare_register_files: errno: None) (task 14611 (rec:14257) at trace line 116336)
</pre>
`trace line 116336` is the failing event.  To see that event,
<pre>
$ rr dump trace_0/ 116336
{
  global_time:116336, event:`futex' (state:0), tid:14257, thread_time:5243
  hw_ints:0, faults:0, rbc:243, insns:3131
  eax:0xffffffda ebx:0x467d17c0 ecx:0x87 edx:0x4a76e000 esi:0x467d17c0 edi:0x467d17c0 ebp:0x467d17c0
  eip:0x40000424 esp:0x5857aef0 eflags:0x200286 orig_eax:240
}
</pre>
Or to see a few events leading up to that one
<pre>
$ rr dump trace_0/ 116331-116336
{
  global_time:116331, event:`USR_SYSCALLBUF_FLUSH' (state:0), tid:14257, thread_time:5238
}
{
  global_time:116332, event:`socketcall' (state:0), tid:14257, thread_time:5239
  hw_ints:0, faults:0, rbc:279, insns:2780
  eax:0xffffffda ebx:0x9 ecx:0x5857ae70 edx:0x4a76e000 esi:0x0 edi:0x467adeec ebp:0x14
  eip:0x40000424 esp:0x5857ae58 eflags:0x200293 orig_eax:102
}
{
  global_time:116333, event:`USR_SYSCALLBUF_RESET' (state:0), tid:14257, thread_time:5240
}
{
  global_time:116334, event:`socketcall' (state:1), tid:14257, thread_time:5241
  hw_ints:0, faults:0, rbc:0, insns:0
  eax:0x14 ebx:0x9 ecx:0x5857ae70 edx:0x4a76e000 esi:0x0 edi:0x467adeec ebp:0x14
  eip:0x40000424 esp:0x5857ae58 eflags:0x200293 orig_eax:102
}
{
  global_time:116335, event:`USR_SYSCALLBUF_FLUSH' (state:0), tid:14257, thread_time:5242
}
{
  global_time:116336, event:`futex' (state:0), tid:14257, thread_time:5243
  hw_ints:0, faults:0, rbc:243, insns:3131
  eax:0xffffffda ebx:0x467d17c0 ecx:0x87 edx:0x4a76e000 esi:0x467d17c0 edi:0x467d17c0 ebp:0x467d17c0
  eip:0x40000424 esp:0x5857aef0 eflags:0x200286 orig_eax:240
}
</pre>
Invoking `rr dump` without any arguments dumps all events, which is useful for grepping over a trace.

#### Be sure to load the right executable image for gdb'ing a tracee
rr doesn't implement gdb multi-process support yet, so you're using 1980s/1990s-era debugging technology.  If you `gdb the-wrong-image`, gdb will get very confused.  If you're trying to attach to a tracee after seeing
<pre>(rr debug server listening on :X)</pre>
then you can make use of the fact that *X* is the tid of the task for which a debugger server has been started.  Launch gdb using the following command
<pre>gdb /proc/X/exe</pre>
and gdb will load the right executable image for the tracee.

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

#### rr and tracees can be debugged simultaneously

For example, if an `assert_exec()` fails and you want to inspect both the tracee and rr, it's perfectly find to `gdb -p $(pidof rr)` in one session and `gdb failing-prog; (gdb) target remote :X` in another.  Obviously though, rr has to be not-stopped to respond to debugger requests.

#### Finding the precise event during recording at which some tracee state changes

Add code that reads the tracee state to `collect_exec_info()`.  Save the previous state somewhere.  Compare previous state to newly-read state.  If they differ, the current event, `get_global_time()`, is the culprit, and you can abort (or whatever) and debug.

#### Test failures that only reproduce during "make check"

During `make check`, the tests run in parallel and really beat on your machine.  Running a test by itself is much less stressful.  To stress your machine in a similar way with just one test, you can run just that test in parallel with itself
<pre>
for i in $(seq 20); do (cd $rr/src/test && bash test.run) & done
</pre>

#### You may find this helper repository useful
[cgjones/rr-workbench](https://github.com/cgjones/rr-workbench): collection of helpers for automating rr tasks (especially running FF unit tests), and for random things like stringifying `waitpid` status codes (status2text).  It also contains an llvm Bell-Larus path logger and some proof-of-concept programs that have been written to prototype rr features.