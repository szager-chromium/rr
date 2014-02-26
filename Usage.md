rr can currently only record and replay 32-bit objects.  This restriction will be lifted in a future version.

## Set up your machine

**Ubuntu only**: rr cannot run with ptrace hardening, because rr must write to /proc/$pid/mem.  Currently Ubuntu enables ptrace hardening but Fedora does not. To disable ptrace hardening "permanently" (persistent across reboots), create a file `/etc/sysctl.d/60-rr.conf` with these contents
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

To start a debugging server for the recording `trace_$n`, run

    rr replay trace_$n

By default, rr will automatically spawn gdb to debug the process. You can set breakpoints within the recording, step, continue, interrupt, etc.  What you *cannot* do is set register or memory values.  This can cause the replay to diverge.

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