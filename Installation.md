## Hardware/software requirements

rr has two mode of operation: 

0. Plain Mode: Requires a Linux kernel version 2.6.39 and higher.  To enable PEBS support for the hardware performance counters (HPC) a kernel version >= 3.0 is required. Was developed under Ubuntu 11.04 &
Ubuntu 12.04.
0. Filter Mode (--filter_lib): Requires kernel version 3.5 to support the seccomp-bpf filter, or Ubuntu 12.04 as it was backported to its 3.2. Developed and tested under Ubuntu 12.04.

Currently rr supports only recording and replaying of 32-bit x86 Linux processes.  It can be built and run as a 32-bit application on an x86_64 system; see below.

rr can run in a virtual machine that supports virtualization of performance counters, such as VMWare Workstation 9.  If you use a VM, **MAKE SURE VIRTUALIZATION OF PERF COUNTERS IS ENABLED**.  On at least
VMWare Workstation 9, the default is for counter virtualization to be disabled.  Installing the latest 32-bit Ubuntu in a VM is an easy way to get rr working.

<font color="red" size="70pt">WARNING</font>: rr needs to disable some modern OS security features to run correctly; see below for more details.  Disabling these features in a "production" system will put that system at significantly higher
security risk.  This is another very good reason to run rr in a virtual machine.  Disabling network access to the VM is further recommended.

rr requires libpfm-4.3.0 and libdisasm-0.23.  Follow the instructions below to install the libraries for your distribution.  It's easiest to set up rr on Ubuntu x86 >= 12.04, so that configuration is recommended.

### Ubuntu >= 11.10

    sudo apt-get install cmake gcc g++

plus, on x86 systems:
    sudo apt-get install libc-dev libdisasm-dev
    wget http://people.mozilla.org/~gal/libpfm_4.3.0-1_i386.deb
    sudo dpkg -i libpfm_4.3.0-1_i386.deb

or on x86-64 systems:

    sudo apt-get install gcc-multilib libc6-dev:i386 libdisasm-dev:i386
    wget http://people.mozilla.org/~gal/libpfm_4.3.0-1_amd64.deb
    sudo dpkg -i libpfm_4.3.0-1_amd64.deb

### Fedora 18

    sudo yum install libpfm

Go to http://sourceforge.net/projects/bastard/files/libdisasm/0.23/ and download libdisasm-0.23.tar.gz.  Then run these commands
    tar zxvf libdisasm-0.23.tar.gz
    cd libdisasm-0.23
    ./configure && make
    sudo make install

If you're using a distribution other than Ubuntu and cross compiling from x86-64 to x86, see the appendix below.

## Building

rr uses the CMake build system, which is able to generate multiple build environments.  So you can choose whichever build driver you prefer to use.

### To use Eclipse

    cmake -G "Eclipse CDT4 - Unix Makefiles"

Next, import the project into Eclipse.  By default Eclipse will automatically build the project when files change.  You can force Eclipse to rebuild the project by pressing Ctrl-B.

### To use Makefiles

    cmake .

Then the command

    make

will build the project.

## Usage

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

The build system generates an executable at the location `$rr/obj/bin/rr`.  For convenience, it's recommended to add that directory to your `$PATH`, or create a symbolic link to that location in
another directory in your `$PATH`.  For example, if `~/bin` is in your `$PATH`
    cd ~/bin
    ln -s $rr/obj/bin/rr rr

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
wrapper is compiled as a shared object called `$rr/obj/lib/librrwrap_syscalls.so` (as part of rr compilation process).

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

## Tests

rr has a suite of tests in `$rr/src/test`.  To run the tests with minimal output,

    make test

or with full output

    make check

Each test consists of a C source file and a `.run` file, which is a shell script.

## Appendix: Building on x86_64 systems other than Ubuntu

On a 64-bit system, rr expects to find the 32-bit versions of those libraries installed under `$(HOME)/32bit`, so you should download those libraries and build them yourself.

To build the 32-bit version of libdisasm-dev, run configure like so:

    ./configure --prefix=$HOME/32bit --build=i686-pc-linux-gnu "CFLAGS=-m32" "CXXFLAGS=-m32" "LDFLAGS=-m32"

To build the 32-bit version of libpfm-4.3.0, run

    make OPTIM="-m32 -O3" PREFIX="$HOME/32bit"

You will need to run rr with `LD_LIBRARY_PATH=$HOME/32bit/lib`.