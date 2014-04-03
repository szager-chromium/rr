## Hardware/software requirements

Currently rr only supports recording and replaying of 32-bit x86 Linux processes.  rr and its dependencies must be built as a 32-bit objects on x86-64 kernels; instructions are below.

If you run rr in a virtual machine, **MAKE SURE VIRTUALIZATION OF PERF COUNTERS IS ENABLED**.  On at least VMWare Workstation 9, the default is for counter virtualization to be disabled.

## Build prerequisites

First, install the compiler toolchain and additional packages.  `python-pexpect` is required to run unit tests.  `man-pages` is optional but strongly recommended if you'll be doing rr development. 
<pre>
sudo yum install \
  ccache cmake gcc gcc-c++ \
  glibc-devel glibc-devel.i686 \
  libstdc++-devel libstdc++-devel.i686
sudo yum install python-pexpect
sudo yum install man-pages
</pre>
<pre>
sudo apt-get install ccache cmake g++-multilib
sudo apt-get install python-pexpect
sudo apt-get install manpages-dev
</pre>

rr requires libpfm-4.5.0 and libdisasm-0.23 for using performance counters and disassembling x86 code, respectively.  In the officially supported configuation, you'll need to build 32-bit versions of these libraries.

* [Download libdisasm-0.23](http://sourceforge.net/projects/bastard/files/libdisasm/0.23/libdisasm-0.23.tar.gz/download) extract the package.
* <pre>
./configure --build=i686-pc-linux-gnu "CFLAGS=-m32" "CXXFLAGS=-m32" "LDFLAGS=-m32"
make
sudo make install
</pre>

* [Download libpfm-4.5.0](http://sourceforge.net/projects/perfmon2/files/libpfm4/libpfm-4.5.0.tar.gz/download) and extract the package.
* <pre>
make OPTIM="-m32 -O3"
sudo make install
</pre>

On Ubuntu 13.10, you can also use binary packages. (Older Ubuntu versions do not have a libpfm package.)
<pre>
sudo apt-get install libdisasm-dev libpfm4-dev # 32-bit
sudo apt-get install libdisasm-dev:i386 libpfm4-dev:i386 # 64-bit
</pre>

## Building

rr uses the CMake build system, which is able to generate multiple build environments.  This enables you to choose whichever build driver you prefer to use.  The commands below show building rr in a separate `obj` directory.  This is recommended because cmake generates a *lot* of auxiliary files.

If you intend to use rr to debug Firefox, follow [these instructions](https://developer.mozilla.org/en-US/docs/Compiling_32-bit_Firefox_on_a_Linux_64-bit_OS#Instructions_for_Fedora_19) to create a Firefox build for Fedora 19 that's compatible with rr.

**NB**: you must disable the gstreamer media backend to record Firefox.  gstreamer uses some features that aren't currently supported (mmaps of unlinked inodes and exec'ing 64-bit processes).  Add this option to your mozconfig

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

rr has a suite of tests in `$rr/src/test`.  To run the tests with minimal output,

    make test

or with full output

    make check

Each test consists of a C source file and a `.run` file, which is a shell script.