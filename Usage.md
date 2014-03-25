rr can currently only record and replay 32-bit objects.  This restriction will be lifted in a future version.

## Set up your machine

**For Ubuntu**: rr cannot run with ptrace hardening, because rr must write to /proc/$pid/mem.  Currently Ubuntu enables ptrace hardening but Fedora does not. To disable ptrace hardening "permanently" (persistent across reboots), create a file `/etc/sysctl.d/60-rr.conf` with these contents
<pre>
kernel.yama.ptrace_scope = 0
</pre>
and reboot.

**IMPORTANT**: If you built rr from source, then `$rrobjdir/lib` must be in your `LD_LIBRARY_PATH`.  Otherwise you'll see an error message like the following when you run rr

    ERROR: ld.so: object 'librrpreload.so' from LD_PRELOAD cannot be preloaded: ignored.

And you may see rr abort with a message like `-> Unhandled IPC call 23`.  To fix this problem, add a line like the following to your .bashrc

    export LD_LIBRARY_PATH="$HOME/rr/obj/lib:${LD_LIBRARY_PATH}"

where `$HOME/rr/obj/` is your rr objdir.

You don't have to worry about this if you installed rr from a distribution package.  

## Recording an execution

Running rr in "record" mode creates a path in the current directory into which the trace files are saved.  To invoke the recorder, run

    rr record /path/to/binary [arguments to binary]

The trace is saved to the path `trace_$n`.

The overhead of recording an application with rr is pretty low, but it depends on the application's workload.  In general you should expect around a 1.2x to 1.4x slowdown.  At the lowest end, a purely CPU-bound tracee will incur around 1.1x-1.2x overhead.  At the high end, a tracee that just makes syscalls in a tight loop can have 4x slowdown or more.

rr doesn't record shared-memory multithreading, so it forces your application's threads to execute serially.  Your application can see an additional slowdown if it takes advantage of multicore parallelism.

## Debugging a recording

To begin debugging a recording `trace_$n`, run
<pre>
$ rr replay trace_0
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

The target process and/or event persist through your debugging session.  That means issuing the gdb `run` command to restart replay will go back to the targeted process/event.
<pre>
(gdb) run
The program being debugged has been started already.
Start it from the beginning? (y or n) y
...
[rr 2789 163548]LoadPlugin() /tmp/tmpUrp7e7/plugins/libnptest.so returned 5a7c7140

--------------------------------------------------
 ---> Reached target process 0 at event 164788.
--------------------------------------------------
</pre>

If you want to restart replay at a different event, you can pass the event number as an argument to the gdb `run` command, like so
<pre>
(gdb) run 159937
...
[rr 2734 159846]For application/x-test found plugin libnptest.so

--------------------------------------------------
 ---> Reached target process 0 at event 159937.
--------------------------------------------------
</pre>

If you just want to replay your recording without attaching a debugger client, invoke
<pre>
rr -m replay -a trace_$n
</pre>

## Limitations

Unlike in "normal" gdb, the rr debug server doesn't allow you to set register or memory values.  A corollary of this restriction is that you can't call tracee functions from within a debugging session.  For example, `(gdb) call Foo()` won't work.  Eventually this will be supported.

Currently, the `/path/to/binary` image you recorded *must not change* before you replay the recording.  If the executable image changes, all kinds of bad things will happen.  This will be fixed in the future.

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

## Getting the best performance on your machine (especially laptops!)

If you're running rr on an (un-docked) laptop, the CPU scaling governor can make a big difference in recording overhead; at least up to 2x.  This has been observed to happen whether or not your laptop is on AC power.  (However, laptops running in desktop docks have been observed not to be affected by this issue.)

The easiest to squeeze out max performance is to change the CPU governor to 'performance'.  For example, on Fedora:
<pre>
sudo yum install kernel-tools
sudo cpupower frequency-set -g performance
</pre>
This setting is only temporary however; it persists until the next reboot. 

Frequency scaling can also be disabled in the BIOS.