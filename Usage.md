<font color="red" size="70pt">WARNING</font>: rr needs to disable some modern OS security features to run correctly; see below for more details.  Disabling these features in a "production" system will put that system at significantly higher
security risk.  This is another very good reason to [run rr in a virtual machine](Installation).  Disabling network access to the VM is further recommended.

rr can currently only record and replay 32-bit objects.  This restriction will be lifted in a future version.

## Set up your machine

rr cannot run with address space randomization (for obvious reasons) or ptrace hardening, because rr must write to /proc/$pid/mem.  These must be disabled before calling either the recorder or the replayer.
You can do this "permanently" (persistent across reboots) by creating a file `/etc/sysctl.d/60-rr.conf` with these contents
<pre>
kernel.randomize_va_space = 0
kernel.yama.ptrace_scope = 0
</pre>

and rebooting.  This is only recommended if you're using an isolated VM.

rr also doesn't handle shared memory correctly currently, so X11 applications can't use the MIT-SHM extension.  To disable this, create a file called `/etc/X11/xorg.conf.d/60-rr.conf` with these contents
<pre>
Section "Extensions"
	Option "MIT-SHM" "disable"
EndSection

Section "Module"
	Disable "dri"
EndSection
</pre>

then reboot.

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

Run `rr -h` or `rr --help` to see the most up-to-date list.  The options below are most useful for rr users, as opposed to developers of rr.

General options:
* `-v, --verbose`: log messages that may not be urgently critical to the user.

Recorder scheduler parameters:
* `-c, --num-cpu-ticks=NUM`: maximum number of 'CPU ticks' (currently retired conditional branches) to allow a task to run before interrupting it.
* `-e, --num-events=<NUM>`: maximum number of events (syscall enter/exit, signal, CPU interrupt, ...) to allow a task before descheduling it.