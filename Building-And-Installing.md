## tl;dr

Fedora
```bash
sudo yum install \
  ccache cmake make gcc gcc-c++ gdb \
  glibc-devel glibc-devel.i686 libstdc++-devel libstdc++-devel.i686 zlib-devel \
  python-pexpect man-pages
```
Ubuntu
```bash
sudo apt-get install ccache cmake make g++-multilib gdb \
  pkg-config libz-dev realpath python-pexpect manpages-dev git zlib1g-dev
```
then
```bash
mkdir rr
cd rr
git clone https://github.com/mozilla/rr.git
cd rr
mkdir obj
cd obj
cmake ..
make -j8
make check
```

## Hardware/software requirements

Supported microarchitectures are Intel architectures newer than Merom and Penryn, i.e. Nehalem and beyond.

`/proc/sys/kernel/perf_event_paranoid` must be <= 1 for rr to work efficiently. Some distros set it to 2 or higher, in which case you either need to set it to 1 or use `rr record -n`, which is slow.

If you run rr in a virtual machine, **MAKE SURE VIRTUALIZATION OF PERF COUNTERS IS ENABLED**. 
* VMWare Workstation 9 / Fusion 7: The default is for counter virtualization to be _disabled_. You have to enable it in the VM settings (advanced processor options). Also add `monitor_control.disable_hvsim_clusters = true` to the VM's `.vmx` file ([more information](http://robert.ocallahan.org/2015/11/rr-in-vmware-solved.html)).
* Qemu: On QEMU command line use <pre>-cpu host</pre>
* Libvirt: Specify CPU passthrough in domain XML definition:<pre>\<cpu mode='host-passthrough'/\></pre>
* Digital Ocean: The only VPS provider known to work with RR.

If rr isn't working at all, run `dmesg|grep PMU`. If you get output like
````
[    0.311273] Performance Events: Fam15h core perfctr, Broken PMU hardware detected, using software events only.
[    0.311279] Failed to access perfctr msr (MSR c0010201 is 25c6c8c489)
````
then something is disabled in your BIOS, or perhaps you have a broken hardware configuration, or you're in a VM without PMU virtualization.

## Build prerequisites

First, install the compiler toolchain and additional packages.  `python-pexpect` is required to run unit tests.  `man-pages` is optional but strongly recommended if you'll be doing rr development. 

On Fedora:
<pre>
sudo yum install \
  ccache cmake gcc gcc-c++ \
  glibc-devel libstdc++-devel zlib-devel \
  python-pexpect man-pages
</pre>

On Ubuntu:
<pre>
sudo apt-get install ccache cmake make g++-multilib \
  pkg-config libz-dev realpath python-pexpect manpages-dev git zlib1g-dev
</pre>

## Building

rr uses the CMake build system, which is able to generate multiple build environments.  This enables you to choose whichever build driver you prefer to use.  The commands below show building rr in a separate `obj` directory.  This is recommended because cmake generates a *lot* of auxiliary files.

    ac_add_options --disable-gstreamer

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
    cmake ../rr

Then the command

    make

will build the project.

## Usage

See [this page](Usage).

## Tests

Be sure to read the [usage instructions](Usage) before running tests.

rr has a suite of tests in `$rr/src/test`. To run the tests with minimal output,

    make test

or with full output

    make check

Each test consists of a C source file and a `.run` file, which is a shell script. To run an individual BASIC_TEST $test outside the harness:

    cd $rr/src/test
    bash basic_test.run -b $rr '' $test

To run a non-basic test:

    cd $rr/src/test
    bash $test.run -b $rr '' $test

To run a non-basic test in 32-bit mode:

    cd $rr/src/test
    bash $test.run -b $rr '' $test_32