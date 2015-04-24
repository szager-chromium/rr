## tl;dr

Fedora
```bash
sudo yum install \
  ccache cmake make gcc gcc-c++ \
  glibc-devel glibc-devel.i686 libstdc++-devel libstdc++-devel.i686 zlib-devel \
  python-pexpect man-pages
```
Ubuntu
```bash
sudo apt-get install ccache cmake make g++-multilib \
  pkg-config libz-dev realpath python-pexpect manpages-dev git zlib1g-dev
```
then
```bash
mkdir rr
cd rr
git clone https://github.com/mozilla/rr.git
mkdir obj
cd obj
cmake ../rr
make -j8
make check
```

## Hardware/software requirements

Supported microarchitectures are Intel architectures newer than Merom and Penryn, i.e. Nehalem and beyond.

If you run rr in a virtual machine, **MAKE SURE VIRTUALIZATION OF PERF COUNTERS IS ENABLED**. 
* VMWare Workstation 9: The default is for counter virtualization to be disabled.
* Qemu: On QEMU command line use <pre>-cpu host</pre>
* Libvirt: Specify CPU passthrough in domain XML definition:<pre>\<cpu mode='host-passthrough'/\></pre>

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
sudo apt-get install ccache cmake g++-multilib \
  libz-dev realpath python-pexpect manpages-dev git zlib1g-dev
</pre>

## Building

rr uses the CMake build system, which is able to generate multiple build environments.  This enables you to choose whichever build driver you prefer to use.  The commands below show building rr in a separate `obj` directory.  This is recommended because cmake generates a *lot* of auxiliary files.

**NB**: you must disable the gstreamer media backend to record Firefox.  gstreamer uses some features that aren't currently supported (mmaps of unlinked inodes).  Add this option to your mozconfig

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

rr has a suite of tests in `$rr/src/test`.  The test harness expects that your rr disk environment is layed out like the following
<pre>
$rr/     # toplevel
  rr/    # srcdir
  $obj/  # objdir
</pre>
The actual names of the `$rr` toplevel and `$obj` directories don't actually matter.  However, the `rr` srcdir actually has to be named "rr" for the tests to run correctly.  If you run the tests and see errors like
<pre>
27: Test 'breakpoint' FAILED
27: Test breakpoint failed, leaving behind /tmp/rr-test-breakpoint-nChiktKdP
27: python: can't open file '/home/tinuviel/github/mozilla/rr/obj/../rr/src/test/breakpoint.py': [Errno 2] No such file or directory
</pre>
then your directories aren't laid out as rr expects.

To run the tests with minimal output,

    make test

or with full output

    make check

Each test consists of a C source file and a `.run` file, which is a shell script.