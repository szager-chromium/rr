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

then reboot.  Alternatively, if you're running rr "one-off" on a production machine (not recommended), you can temporarily configure your kernel
for rr by running

    $rr/src/script/setup.sh

Running rr in "record" mode creates a path in the current directory which contains the trace file(s).  It's recommended to run rr from within a scratch directory outside the $rr clone.  For example
    cd $rr/..
    mkdir scratch-rr
    cd scratch-rr

Then to invoke the recorder, run
    rr --record /path/to/binary

The trace is saved to the path `trace_$n`.

To replay an application trace saved to the directory trace_$n, run
    rr --replay trace_$n

Other command line options:
* `--filter_lib=<path>` : Only for kernels >= 3.5; this installs a wrapper for several system calls (look at share/wrap_syscalls.c) to speed up their handling and save on the ptrace() context switch. The
wrapper is compiled as a shared object called `$rr/obj/lib/librrwrap_syscalls.so` (as part of rr compilation process).  <font color="red">**NOTE**</font>: this library is not well supported yet and its use is not recommended.

Debug command line options:
* `--dump_on=<syscall|-signal>` : dump the entire memory to file every time the system call numbered <syscall> is encountered or a signal numbered by <signal> is encountered. The negation if signal is
required to distingish between syscalls and signals of the same number.  example: --dump_on=79 : dumps all memory every time gettimeofday is entered. --dump_on=-17: dumps all memory every time signal SIGCHLD is recieved.
* `--dump_at=<time>` : dump the entire memory at global time <time>
* `--checksum={on-syscalls,on-all-events}|<from-time>` : For record: calculated a checksum of the entire memory either on every event (on-all-events) or on every syscall (on-syscalls) or starting from
global time <from-time>. Fore replay: verifies the calculated checksums, should be run with same arguments as the record.
* `--no_redirect_output` : by default, rr will write to stdout and stderr output that was sent to those files during recording.  In theory this could affect the integrity of the replay, however this is usually the behavior one expects.  Providing this switch disables writing to stdout/stderr during replay.

Note that talking to particular kernel drivers (by `ioctl(...)`) requires the rr to have knowledge about what the driver is doing. For example, if you record GUI applications that make use a graphics driver with 3d-support rr will fail. If you're using NVidia binary drivers, removing the xorg-x11-drv-nvidia-libs package will avoid this problem.

Also, it is advised to turn off any CPU frequency scaling capabilities, such as Intel Speed Step.  This can be achieved by either turning it off in the BIOS or by disabling it in the kernel, e.g. by setting the CPU governor to 'performance'. If neither of these solutions is feasible, rr and its child processes should be pinned on a certain core, e.g. by running rr with:

    taskset 0x1 rr <options>

or, in the case of a CPU that is able to run multiple threads on one core in parallel, such as Intel CPUs with Hyperthreading capabilities, it is better to use:

    taskset 0x3 rr <options>

Otherwise, due to frequent context switches between rr and the traced process, the different cores will often scale down their frequency which can lead to slowdowns of up to 2x.