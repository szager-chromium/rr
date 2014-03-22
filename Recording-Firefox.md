This page is intended to help Firefox/Gecko developers get started using rr to debug Firefox.

## Prerequisites

Before beginning, ensure that you've [installed](http://rr-project.org/) or [built](https://github.com/mozilla/rr/wiki/Installation) rr and have [used it successfully](https://github.com/mozilla/rr/wiki/Usage).

rr can only record 32-bit processes.  That means that if you're running a 64-bit kernel, you'll need to build Firefox as a 32-bit binary.  Luckily, it's pretty straightforward to set this up.  There are instructions for [Fedora](https://developer.mozilla.org/en-US/docs/Compiling_32-bit_Firefox_on_a_Linux_64-bit_OS#Instructions_for_Fedora_20_and_19) (recommended) and [Ubuntu](https://developer.mozilla.org/en-US/docs/Compiling_32-bit_Firefox_on_a_Linux_64-bit_OS#Instructions_for_Ubuntu_13.10).

Another option is set up a virtual machine in which to record Firefox.  Be forewarned though that

* rr requires a VM hypervisor that virtualizes CPU performance counters.  VMWare Workstation supports that.
* there's a 20% or so performance hit from running in a VM; generally speaking recorder overhead increases from ~1.2x to ~1.4x.  (It's a feather in the cap of the hypervisor authors that the hit is that small, though!)

If you do install a VM, a 32-bit Fedora installation is recommended because it has the best debugging environment and can build Firefox "natively" as a 32-bit binary.  However, any of { Fedora, Ubuntu } x { 32-bit, 64-bit } will work.

## Recording Firefox

To record Firefox running normally, simply launch it under rr as you would if running it under valgrind or gdb
<pre>
rr record $ff-objdir/dist/bin/firefox -no-remote ...
</pre>
This will save a trace to your working directory as described in the [usage instructions](https://github.com/mozilla/rr/wiki/Usage).  Please refer to [those instructions](https://github.com/mozilla/rr/wiki/Usage) for details on how to debug the recording, which isn't covered in this document.

## Recording unit test suites