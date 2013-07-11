<font color="red" size="70pt">WARNING</font>: rr needs to disable some modern OS security features to run correctly; see below for more details.  Disabling these features in a "production" system will put that system at significantly higher
security risk.  This is another very good reason to [run rr in a virtual machine](Installation).  Disabling network access to the VM is further recommended.

rr can currently only record and replay x86 binaries.  This restriction will be lifted in a future version.  rr also requires a relatively recent chip, Merom or so and later.  That, unfortunately, may be a permanent restriction.

## Set up your machine

rr cannot run with address space randomization (for obvious reasons) or ptrace hardening, because rr must write to /proc/$pid/mem.  These must be disabled before calling either the recorder or the replayer.
You can do this "permanently" (persistent across reboots) by adding these lines to your system's /etc/sysctl.conf
<pre>
###################################################################
## Options for rr
kernel.randomize_va_space = 0
kernel.yama.ptrace_scope = 0
</pre>

and rebooting.  This is only recommended if you're using an isolated VM.

rr also doesn't handle shared memory correctly currently, so X11 applications can't use the MIT-SHM extension.  To disable this, create a file called `/usr/share/X11/xorg.conf.d/60-rr.conf` with these contents
<pre>
Section "Extensions"
	Option "MIT-SHM" "disable"
EndSection

Section "Module"
	Disable "dri"
EndSection
</pre>

then reboot.

## Recording an execution

Running rr in "record" mode creates a path in the current directory which contains the trace file(s).  To invoke the recorder, run

    rr record /path/to/binary [arguments to binary]

The trace is saved to the path `trace_$n`.

## Debugging a recording

To start a debugging server for the recording `trace_$n`, run

    rr replay trace_$n

By default, rr will print a message like

    [INFO] rr debug server listening on :[port]

where `[port]` will be a number like `17226`.  This signifies that the debug server is ready.  To connect the debug client,

    gdb /path/to/binary
    (gdb) target remote :[port]

When your client connects, you can set breakpoints within the recording, step, continue, interrupt, etc.  What you *cannot* do is set register or memory values.  This can cause the replay to diverge.

However, in exchange, each debugging session on an rr recording is **entirely deterministic**.  The order of execution of each machine instruction, the value of every bit of memory, will be identical from one run to the next, as seen by your debugger client.  The replayed execution will also often be considerably faster than "real time".  This allows you to quickly "binary search" over a recording to see where some value in memory went bad.  Of course, in the future rr will be able to do this itself.

**Note**: currently, the `/path/to/binary` image you recorded *must not change* before you replay the recording.  If the executable image changes, all kinds of bad things will happen.  This limitation will be lifted in the future.

If for some reason you just want to replay your recording outside a debugger,

    rr replay -a trace_$n

will do the trick.

It's recommended to run rr from within a scratch directory outside the $rr clone.  For example

    cd $rr/..
    mkdir scratch-rr
    cd scratch-rr

## Note about GPU drivers

Note that talking to particular kernel drivers (by `ioctl(...)`) requires the rr to have knowledge about what the driver is doing. For example, if you record GUI applications that make use a graphics driver with 3d-support rr will fail. If you're using NVidia binary drivers, removing the xorg-x11-drv-nvidia-libs package will avoid this problem.

## Further machine configuration

It is advised to turn off any CPU frequency scaling capabilities, such as Intel Speed Step.  This can be achieved by either turning it off in the BIOS or by disabling it in the kernel, e.g. by setting the CPU governor to 'performance'. If neither of these solutions is feasible, rr and its child processes should be pinned on a certain core, e.g. by running rr with:

    taskset 0x1 rr <options>

or, in the case of a CPU that is able to run multiple threads on one core in parallel, such as Intel CPUs with Hyperthreading capabilities, it is better to use:

    taskset 0x3 rr <options>

Otherwise, due to frequent context switches between rr and the traced process, the different cores will often scale down their frequency which can lead to slowdowns of up to 2x.

## Other command line options

Run `rr -h` or `rr --help` to see the most up-to-date list.

Scheduler parameters:
* `-c, --num-cpu-ticks=NUM`: maximum number of 'CPU ticks' (currently retired conditional branches) to allow a task to run before interrupting it.
* `-e, --num-events=<NUM>`: maximum number of events (syscall enter/exit, signal, CPU interrupt, ...) to allow a task before descheduling it.

Syscall buffer:
* <font color="red">**NOTE**</font>: this library is not well supported yet and its use is not recommended.  `-b` : Only for kernels >= 3.5; this installs a wrapper for several system calls (look at share/wrap_syscalls.c) to speed up their handling and save on the ptrace() context switch. The
wrapper is compiled as a shared object called `$rr/obj/lib/librrwrap_syscalls.so` (as part of rr compilation process).

Debug command line options:
* `--dump_on=<syscall|-signal>` : dump the entire memory to file every time the system call numbered <syscall> is encountered or a signal numbered by <signal> is encountered. The negation if signal is
required to distingish between syscalls and signals of the same number.  example: --dump_on=79 : dumps all memory every time gettimeofday is entered. --dump_on=-17: dumps all memory every time signal SIGCHLD is recieved.
* `--dump_at=<time>` : dump the entire memory at global time <time>
* `--checksum={on-syscalls,on-all-events}|<from-time>` : For record: calculated a checksum of the entire memory either on every event (on-all-events) or on every syscall (on-syscalls) or starting from
global time <from-time>. Fore replay: verifies the calculated checksums, should be run with same arguments as the record.
* `--no_redirect_output` : by default, rr will write to stdout and stderr output that was sent to those files during recording.  In theory this could affect the integrity of the replay, however this is usually the behavior one expects.  Providing this switch disables writing to stdout/stderr during replay.