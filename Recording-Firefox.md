This page is intended to help Firefox/Gecko developers get started using rr to debug Firefox.

## Prerequisites

Before beginning, ensure that you've [installed](http://rr-project.org/) or [built](https://github.com/mozilla/rr/wiki/Installation) rr and have [used it successfully](https://github.com/mozilla/rr/wiki/Usage).

rr can only record 32-bit processes.  That means that if you're running a 64-bit kernel, you'll need to build Firefox as a 32-bit binary.  Luckily, it's pretty straightforward to set this up.

Here are the instructions for [Fedora](https://developer.mozilla.org/en-US/docs/Compiling_32-bit_Firefox_on_a_Linux_64-bit_OS#Instructions_for_Fedora_20_and_19) (recommended) and [Ubuntu](https://developer.mozilla.org/en-US/docs/Compiling_32-bit_Firefox_on_a_Linux_64-bit_OS#Instructions_for_Ubuntu_13.10).

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

## Recording test suites

There are a few annoyances that need to be worked around to record Firefox test suites.  First, you need to use the test runners' `--debugger` feature to punch rr down through the layers of python script to where Firefox is launched.  This is used in the same way you would use `--debugger` to run valgrind or gdb, for example.

Second, several test harnesses create temporary profiles in which to run the tests, and then `chdir` to that profile directory.  That's bad because rr will save the recording to the tmpdir, but then the recording will be blown away at the end of the test run!  rr allows you to save traces to some directory other than the cwd by setting the `_RR_TRACE_DIR=` environment variable.

Third, the test harnesses disable the slow-script timeout when the `--debugger` argument is passed.  That's usually sensible, because you don't want those warnings being generated while Firefox is stopped in gdb.  However, this has been [observed to change Gecko behavior](https://bugzilla.mozilla.org/show_bug.cgi?id=986673).  rr doesn't need to have the slow-script timeout disabled, so to avoid those kinds of pitfalls, pass the `--slowscript` argument to the test harness.

Fourth, you need to pass the "record" argument to rr when it's launched, so that it knows it's supposed to record Firefox.  You can use the test harness `--debugger-args` option to do this, however see below.

Obviously you'll want to script away these distractions.  [This example setup](https://github.com/cgjones/rr-workbench/blob/master/Makefile) is used by one rr developer.  Another rr developer has a script called `rrrecord` that looks like
<pre>
#!/usr/bin/bash

export _RR_TRACE_DIR="$HOME/ff-working-dir"  # whatever your working dir is when debugging FF
exec rr record "$@"
</pre>
Put this script in a directory in your `$PATH`.  Then use it as the `--debugger` option for the test harnesses, for example
<pre>
make -C $ff-objdir EXTRA_TEST_ARGS="--debugger=rrrecord --slowscript" mochitest-plain
</pre>
and at the end of the test run, a recording will be located in whichever `$HOME/ff-working-dir` you configured in the script.

## Get help!

If you encounter a problem with rr, please [file an issue](https://github.com/mozilla/rr/issues).  Firefox bugs are high priority, so usually your issue can be fixed very quickly.

If you want to chat with rr developers, because you need more help or want to contribute or want to complain, we hang out in the #research channel on moznet.

You also may find [these debugging protips](https://github.com/mozilla/rr/wiki/Debugging-protips) helpful, though many are for rr developers, not users.

Happy debugging!