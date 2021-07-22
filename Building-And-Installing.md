## Dependencies installation

Fedora
```bash
sudo dnf install \
  ccache cmake make gcc gcc-c++ gdb libgcc libgcc.i686 \
  glibc-devel glibc-devel.i686 libstdc++-devel libstdc++-devel.i686 libstdc++-devel.x86_64 \
  python3-pexpect man-pages ninja-build capnproto capnproto-libs capnproto-devel
```
Ubuntu
```bash
sudo apt-get install ccache cmake make g++-multilib gdb \
  pkg-config coreutils python3-pexpect manpages-dev git \
  ninja-build capnproto libcapnp-dev
```
On some old distros such as Ubuntu 14.04 `libcapnp-dev` doesn't include `capnp.pc`. To build rr on such distros, manually install `capnproto` using [the instructions here](https://capnproto.org/install.html#installation-unix).

## Project building

rr uses the CMake build system, which is able to generate multiple build environments.  This enables you to choose whichever build driver you prefer to use.  The commands below show building rr in a separate `obj` directory.  This is recommended because `cmake` generates a *lot* of auxiliary files.:

```bash
git clone https://github.com/rr-debugger/rr.git
mkdir obj && cd obj
cmake ../rr
```

Then to use `make` and the system default compiler to build:
```bash
make -j8
sudo make install
```

Or to use clang and Ninja to build (faster!):
```bash
CC=clang CXX=clang++ cmake -G Ninja ../rr
cmake --build .
sudo cmake --build . --target install
```
RHEL7/CentOS7 with EPEL requires installing `python36-pexpect` and running `cmake3`; **it will only work with an [updated kernel](#os-configuration)**.

To use Eclipse:
```bash
mkdir obj && cd obj
cmake -G "Eclipse CDT4 - Unix Makefiles" ../rr
```

Next, import the project into Eclipse.  By default Eclipse will automatically build the project when files change.  You can force Eclipse to rebuild the project by pressing Ctrl-B.

PaX kernel:

If your kernel is a PaX kernel (if these words don't mean anything to you, you can skip this paragraph), then you will need to disable `MPROTECT` on the built files `bin/rr`, `bin/rr_exec_stub` and `bin/rr_exec_stub_32`.

If you use PT header flags, for example, you should run:
```bash
paxctl -cm bin/rr bin/rr_exec_stub bin/rr_exec_stub_32
```
in your build directory.

## Hardware/software configuration

For a quick check: [Will rr work on my system?](Will-rr-work-on-my-system)

### Supported hardware

Supported microarchitectures are Intel architectures newer than Merom and Penryn, i.e. Nehalem and beyond.

### OS configuration

Linux kernel 3.11 or higher is required (check with `uname -r`).

`/proc/sys/kernel/perf_event_paranoid` must be <= 1 for rr to work efficiently (i.e. be able to use `perf` counters). Some distros set it to 2 or higher, in which case you either need to set it to 1 or use `rr record -n`, which is slow.  Temporarily change the setting by running
```bash
$ sudo sysctl kernel.perf_event_paranoid=1
```
Apply the setup automatically on startup by running
```bash
# echo 'kernel.perf_event_paranoid=1' | sudo tee '/etc/sysctl.d/51-enable-perf-events.conf'
```

### Virtual machine guests

If you run rr in a virtual machine, **MAKE SURE VIRTUALIZATION OF PERF COUNTERS IS ENABLED**.  Virtual machines that work with rr and the settings required are listed below.
* VMWare Workstation/Fusion:
  * The default is for counter virtualization to be _disabled_. You have to enable it in the VM settings (advanced processor options).
  * Set the Preferred virtualization engine in the Advanced tab to Intel VT-x with EPT (it may default to Automatic).
  * Enable the code profiling applications in the Processors & Memory tab.
  * Add `monitor_control.disable_hvsim_clusters = true` to the VM's `.vmx` file ([more information](http://robert.ocallahan.org/2015/11/rr-in-vmware-solved.html)).
* Qemu: On QEMU command line use <pre>-cpu host</pre>
* Libvirt/KVM: Specify CPU passthrough in domain XML definition:<pre>\<cpu mode='host-passthrough'/\></pre>
* Hyper-V: Run the PowerShell command `Set-VMProcessor MyVMName -Perfmon @("pmu")`

VirtualBox **does not work** at this time because it doesn't support PMU virtualization. It would be great if someone contributed that to the open-source project...

Xen's PMU virtualization has [bugs](https://lists.xen.org/archives/html/xen-devel/2017-07/msg02242.html) that prevent rr from working.

### Cloud virtual guests

Some Digital Ocean instances have worked in the past.

Amazon EC2 instance types `c5[d].9xlarge`, `c5[d].18xlarge`, `m5[d].12xlarge`, and `m5[d].24xlarge` should work. All bare metal instance types should work. Some other instance types may work (those that use the "Nitro" hypervisor and where the instance occupies a whole CPU socket).

### Docker

Make sure rr works on the machine outside of Docker, then follow the [Docker-specific instructions](https://github.com/mozilla/rr/wiki/Docker).

### Troubleshooting

If rr isn't working at all, run `dmesg|grep PMU`. If you get output like
````
[    0.311273] Performance Events: Fam15h core perfctr, Broken PMU hardware detected, using software events only.
[    0.311279] Failed to access perfctr msr (MSR c0010201 is 25c6c8c489)
````
then something is disabled in your BIOS, or perhaps you have a broken hardware configuration, or you're in a VM without PMU virtualization.

## Usage

See [this page](Usage).

## Tests

Be sure to read the [usage instructions](Usage) before running tests.

Remember to set `perf_event_paranoid` to level 1 or lower, because otherwise many tests will fail.

rr has a suite of tests in `$rr/src/test`. To run them you first need to enter the build directory, and then, depending on the build system you chose, execute a `ninja test` or `make -j$(nproc) test`.

Alternatively, you can run them manually, by calling `ctest`. E.g. with minimal output:

```bash
ctest -j$(nproc)
```

or with full output:

```bash
ctest -j$(nproc) -VV
```

The `video_capture` test may briefly turn on an attached camera, if you have one --- do not be alarmed!

### Running individual tests

Easiest way is `ctest -R test_name`. For example:

```
 $ ctest -R 64bit_child-no-syscallbuf
Test project /home/constantine/Projects/rr/build
    Start 3: 64bit_child-no-syscallbuf
1/1 Test #3: 64bit_child-no-syscallbuf ........   Passed    0.62 sec

100% tests passed, 0 tests failed out of 1

Total Test time (real) =   0.72 sec
```

This runs tests matching a regular expression. So for example, given tests `pro-foo`, `foo`, `foo-2`, `foo-3`, running a `ctest -r foo` will result in all of them running. If you want to run exactly the tests `foo`, `bar`, `buzz`, but not anything else like a `foo-3`, you can execute the following:

```
 $ ctest -R '^(foo|bar|buzz)$'
```

See also `man ctest`.

Alternatively, each test consists of a C source file and a `.run` file, which is a shell script. To run an individual basic test outside the harness:

```bash
cd $rr/src/test
bash basic_test.run $test
```

To run a non-basic test:

```bash
cd $rr/src/test
bash $test.run
```

To run a non-basic test in 32-bit mode:

```bash
cd $rr/src/test
bash $test.run $test_32
```
