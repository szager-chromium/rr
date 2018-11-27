## tl;dr

Fedora
```bash
sudo dnf install \
  ccache cmake make gcc gcc-c++ gdb libgcc libgcc.i686 \
  glibc-devel glibc-devel.i686 libstdc++-devel libstdc++-devel.i686 \
  python-pexpect python3-pexpect man-pages ninja-build capnproto capnproto-libs capnproto-devel
```
Ubuntu
```bash
sudo apt-get install ccache cmake make g++-multilib gdb \
  pkg-config coreutils python3-pexpect python-pexpect manpages-dev git \
  ninja-build capnproto libcapnp-dev
```
On Ubuntu 14.04 (and maybe other distros) `libcapnp-dev` doesn't include `capnp.pc`. To build rr on such distros, manually install `capnproto` using [the instructions here](https://capnproto.org/install.html#installation-unix).

If you are using Python 3 then replace the above `python-pexpect` package with `python3-pexpect`.

Then
```bash
git clone https://github.com/mozilla/rr.git
mkdir obj
cd obj
````
Then to use `make` and the system default compiler to build:
```
cmake ../rr
make -j8
make test
make install
```
Or to use clang and Ninja to build (faster!):
````
CC=clang CXX=clang++ cmake -G Ninja ..
cmake --build .
cmake --build . --target test
sudo cmake --build . --target install
````

## Hardware/software configuration

### Supported hardware

Supported microarchitectures are Intel architectures newer than Merom and Penryn, i.e. Nehalem and beyond.

### OS configuration

Linux kernel 3.11 or higher is required.

`/proc/sys/kernel/perf_event_paranoid` must be <= 1 for rr to work efficiently (i.e. be able to use `perf` counters). Some distros set it to 2 or higher, in which case you either need to set it to 1 or use `rr record -n`, which is slow.  Temporarily change the setting by running
```bash
$ sudo sysctl kernel.perf_event_paranoid=1
```
Apply the setup automatically on startup by running
```bash
$ sudo bash 
# echo 'kernel.perf_event_paranoid=1' > '/etc/sysctl.d/51-enable-perf-events.conf'
# exit
```

### Virtual machine guests

If you run rr in a virtual machine, **MAKE SURE VIRTUALIZATION OF PERF COUNTERS IS ENABLED**.  The virtual machines that do work with rr and the settings required are listed below.  If a virtual machine isn't on this list, then it cannot be used with rr.
* VMWare Workstation 9 / Fusion 7 & 8:
  * The default is for counter virtualization to be _disabled_. You have to enable it in the VM settings (advanced processor options).
  * Set the Preferred virtualization engine in the Advanced tab to Intel VT-x with EPT (it may default to Automatic).
  * Enable the code profiling applications in the Processors & Memory tab.
  * Add `monitor_control.disable_hvsim_clusters = true` to the VM's `.vmx` file ([more information](http://robert.ocallahan.org/2015/11/rr-in-vmware-solved.html)).

* Qemu: On QEMU command line use <pre>-cpu host</pre>
* Libvirt/KVM: Specify CPU passthrough in domain XML definition:<pre>\<cpu mode='host-passthrough'/\></pre>

VirtualBox **does not work** at this time because it doesn't support PMU virtualization. It would be great if someone contributed that to the open-source project...

Hyper-V does not seem to support PMU virtualization.

Xen's PMU virtualization has bugs that prevent rr from working.

### Cloud virtual guests

Some Digital Ocean instances have worked in the past.

Amazon EC2 instance types `c5[d].9xlarge`, `c5[d].18xlarge`, `m5[d].12xlarge`, and `m5[d].24xlarge` should work. All bare metal instance types should work. Some other instance types may work (those that use the "Nitro" hypervisor and where the instance occupies a whole CPU socket).

### Troubleshooting

If rr isn't working at all, run `dmesg|grep PMU`. If you get output like
````
[    0.311273] Performance Events: Fam15h core perfctr, Broken PMU hardware detected, using software events only.
[    0.311279] Failed to access perfctr msr (MSR c0010201 is 25c6c8c489)
````
then something is disabled in your BIOS, or perhaps you have a broken hardware configuration, or you're in a VM without PMU virtualization.

## Build prerequisites

First, install the compiler toolchain and additional packages.  `python-pexpect` is required to run unit tests.  `man-pages` is optional but strongly recommended if you'll be doing rr development. See the package lists at [the top of this page](#tldr).

## Building

rr uses the CMake build system, which is able to generate multiple build environments.  This enables you to choose whichever build driver you prefer to use.  The commands below show building rr in a separate `obj` directory.  This is recommended because cmake generates a *lot* of auxiliary files.

### To use Eclipse

    cd ../
    mkdir obj
    cd obj
    cmake -G "Eclipse CDT4 - Unix Makefiles" ../rr

Next, import the project into Eclipse.  By default Eclipse will automatically build the project when files change.  You can force Eclipse to rebuild the project by pressing Ctrl-B.

### To use Makefiles

    cd ../
    mkdir obj
    cd obj
    cmake ..

Then the command

    make

will build the project.

### Building and installing with a custom prefix

    cmake -DCMAKE_INSTALL_PREFIX=<prefix> ..
    make install

### PaX kernels

If your kernel is a PaX kernel (if these words don't mean anything to you, you can skip this paragraph), then you will need to disable MPROTECT on the built files `bin/rr`, `bin/rr_exec_stub` and `bin/rr_exec_stub_32`.

If you use PT header flags, for example, you should run

    paxctl -cm bin/rr bin/rr_exec_stub bin/rr_exec_stub_32

in your build directory.

## Usage

See [this page](Usage).

## Tests

Be sure to read the [usage instructions](Usage) before running tests.

rr has a suite of tests in `$rr/src/test`. To run the tests with minimal output,

    make test

or with full output

    make check

The `video_capture` test may briefly turn on an attached camera, if you have one --- do not be alarmed!

Each test consists of a C source file and a `.run` file, which is a shell script. To run an individual BASIC_TEST $test outside the harness:

    cd $rr/src/test
    bash basic_test.run $test

To run a non-basic test:

    cd $rr/src/test
    bash $test.run

To run a non-basic test in 32-bit mode:

    cd $rr/src/test
    bash $test.run $test_32