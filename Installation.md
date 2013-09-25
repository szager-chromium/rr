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

rr uses the CMake build system, which is able to generate multiple build environments.  So you can choose whichever build driver you prefer to use.  The commands below show building rr in a separate `obj` directory.  This is recommended because cmake generates a *lot* of auxiliary files.

It can be convenient to have a bare-bones VM that shares an rr source directory with its host, which is set up for day-to-day development.  So you hack in the host, then build and run in the VM.  If your workflow is similar, ensure that you *do not share the rr objdir* through your hypervisor.  This is tremendously slower, and at least in VMWare Workstation 9.0, extremely buggy.

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

rr has a suite of tests in `$rr/src/test`.  To run the tests with minimal output,

    make test

or with full output

    make check

Each test consists of a C source file and a `.run` file, which is a shell script.

## Appendix: Building on x86_64 systems other than Ubuntu

To build the 32-bit version of [libdisasm-0.23](http://sourceforge.net/projects/bastard/files/libdisasm/0.23/libdisasm-0.23.tar.gz/download), download and extract the package, then configure like so:

    ./configure --build=i686-pc-linux-gnu "CFLAGS=-m32" "CXXFLAGS=-m32" "LDFLAGS=-m32"

To build the 32-bit version of [libpfm-4.3.0](http://sourceforge.net/projects/perfmon2/files/libpfm4/libpfm-4.3.0.tar.gz/download), download and extract, then build
    make OPTIM="-m32 -O3"