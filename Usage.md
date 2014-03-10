rr can currently only record and replay 32-bit objects.  This restriction will be lifted in a future version.

## Set up your machine

**For Ubuntu**: rr cannot run with ptrace hardening, because rr must write to /proc/$pid/mem.  Currently Ubuntu enables ptrace hardening but Fedora does not. To disable ptrace hardening "permanently" (persistent across reboots), create a file `/etc/sysctl.d/60-rr.conf` with these contents
<pre>
kernel.yama.ptrace_scope = 0
</pre>
and reboot.

If you see an error message like the following when you run rr

    ERROR: ld.so: object 'librrpreload.so' from LD_PRELOAD cannot be preloaded: ignored.

then rr's objdir isn't in your `LD_LIBRARY_PATH`.  You should never see this if you installed rr from a distribution package.  To fix this problem, add a line like the following to your .bashrc

    export LD_LIBRARY_PATH="$HOME/rr/obj/lib:${LD_LIBRARY_PATH}"

where `$HOME/rr/obj/` is your rr objdir.

## Recording an execution

Running rr in "record" mode creates a path in the current directory which contains the trace file(s).  To invoke the recorder, run

    rr record /path/to/binary [arguments to binary]

The trace is saved to the path `trace_$n`.

## Debugging a recording

To begin debugging a recording `trace_$n`, run
<pre>
$ rr replay trace_$n
GNU gdb (GDB) Fedora 7.6.50.20130731-19.fc20
...
0x4cee2050 in _start () from /lib/ld-linux.so.2
(gdb)
</pre>
rr automatically spawns a gdb client and connects it to the replay server. You can set breakpoints within the gdb session, step, continue, interrupt, etc. as you would normally in gdb.

What's special about rr is *restarting* the replayed execution.  Within the same gdb session, you can replay execution from the beginning again by issuing the `run` command:
<pre>
(gdb) run
The program being debugged has been started already.
Start it from the beginning? (y or n) y
...
</pre>
gdb preserves all your breakpoints and debugging state across the `run` command.  Because rr replay is entirely deterministic, the restarted replay session will execute exactly the same sequence of instructions as the previous replay session.  This allows you to take debugging "shortcuts" like hard-coding memory addresses to examine: all pointers have the same values in every replaying of your recorded trace.

To more precisely target your debugging session, you can set things up so that the debugger is only spawned after a target OS process is `fork`d or `exec`d, and/or when a particular execution event is reached.  If your test suite prints a message that reveals the process IDs of launched subprocesses, like for example
<pre>
[2789] WARNING: '!compMgr', file /home/cjones/rr/mozilla-central/xpcom/glue/nsComponentManagerUtils.cpp, line 59
 ^^^^
</pre>
then you can program rr to launch gdb when that process is `exec`d by using the `-p PID` option:
<pre>
$ rr replay -p 2789 trace_0
...
For application/x-test found plugin libnptest.so

--------------------------------------------------
 ---> Reached target process 2789 at event 159937.
--------------------------------------------------
...
(gdb) 
</pre>
(Remember, rr sets things up so that processes look like they have the same PID in replay as they did during recording.)  If you instead want to launch the debugger when that process is `fork`d, use the `-f 
PID` option.

rr associates an "event number" with each event it records.  You can have rr mark writes to stdio with the corresponding event number and PID that made the write by using the `-m` switch
<pre>
$ rr -m replay trace_0
...
[rr 2789 163548]LoadPlugin() /tmp/tmpUrp7e7/plugins/libnptest.so returned 5a7c7140
[rr 2789 164788][2789] WARNING: '!compMgr', file /home/cjones/rr/mozilla-central/xpcom/glue/nsComponentManagerUtils.cpp, line 59
</pre>
Note that rr tagged the output with `[rr PID EVENT-NUMBER]`, in this case 2789/164788.  You can program rr to launch the debugger at a specific event by using the `-g EVENT` options
<pre>
$ rr -m replay -g 164788 trace_0
...
[rr 2789 163548]LoadPlugin() /tmp/tmpUrp7e7/plugins/libnptest.so returned 5a7c7140

--------------------------------------------------
 ---> Reached target process 0 at event 164788.
--------------------------------------------------
...
(gdb) 
</pre>

What you *cannot* do is set register or memory values.  This can cause the replay to diverge.

However, in exchange, each debugging session on an rr recording is **entirely deterministic**.  The order of execution of each machine instruction, the value of every bit of memory, will be identical from one run to the next, as seen by your debugger client.  The replayed execution will also often be considerably faster than "real time".  This allows you to quickly "binary search" over a recording to see where some value in memory went bad.  Of course, in the future rr will be able to do this itself.

**Note**: currently, the `/path/to/binary` image you recorded *must not change* before you replay the recording.  If the executable image changes, all kinds of bad things will happen.  This limitation will be lifted in the future.

If for some reason you just want to replay your recording outside a debugger, invoke

    rr replay -a trace_$n

It's recommended to run rr from within a scratch directory outside the $rr clone.  For example

    cd $rr/..
    mkdir scratch-rr
    cd scratch-rr

## Other command line options

Run `rr -h` or `rr --help` to see the most up-to-date list.  The options below are most useful for rr users, as opposed to developers of rr.

General options:
* `-v, --verbose`: log messages that may not be urgently critical to the user.
* `-m, --mark-stdio`: write "current event number" before every stdio output line (see `rr replay -g` below).

Recorder parameters:
* `-c, --num-cpu-ticks=<NUM>`: maximum number of 'CPU ticks' (currently retired conditional branches) to allow a task to run before interrupting it.
* `-e, --num-events=<NUM>`: maximum number of events (syscall enter/exit, signal, CPU interrupt, ...) to allow a task before descheduling it.

Recording traces under different scheduling params can help reproduce nondeterministic bugs.  rr's scheduler is relatively deterministic.

Replay parameters:
* `-f, --onfork=<PID>`: debug <PID> when forked
* `-p, --onprocess=<PID>`: debug <PID> when execed
* `-g, --goto=<EVENT-NUM>`: execute forward until event <EVENT-NUM> is reached before debugging

## Getting the best performance on your machine

rr doesn't take advantage of hardware parallelism yet.  So it often runs faster when the tracer and tracee processes are pinned to single virtual CPU.  This will probably end up the default for 1.0, but you can always manually pin rr by running

    taskset 0x1 rr ...

It may also help to disable CPU frequency scaling, such as Intel Speed Step.  It can be disabled in either the BIOS or the kernel, e.g. by setting the CPU governor to 'performance'.

Otherwise, frequent context switches between rr and tracees on different cores are expensive, and can cause unlucky migrations to scaled-down cores.