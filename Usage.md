## tl;dr

```bash
rr record /path/to/my/program --args
rr replay
```

## Malicious code

*rr is not designed to handle malicious code safely!* In particular if you record an application that runs untrusted code in a sandbox based on `seccomp` or kernel namespaces, rr intentionally makes holes in the sandbox to facilitate the recording of the sandboxed code. In theory, an attacker could design malicious code to identify when it's being run under rr and use those holes to escape the sandbox.

## Getting the best performance on your machine (especially laptops!)

If you're running rr on an (un-docked) laptop, the CPU scaling governor can make a big difference in recording overhead; at least up to 2x.  This has been observed to happen whether or not your laptop is on AC power.  (However, laptops running in desktop docks have been observed not to be affected by this issue.)

The easiest way to squeeze out max perf is to change the CPU governor to 'performance'.

On Fedora:
<pre>
sudo yum install kernel-tools
sudo cpupower frequency-set -g performance
</pre>

On Ubuntu:
<pre>
sudo apt-get install cpufrequtils
sudo cpufreq-set -g performance
</pre>

This setting is only temporary however; it only lasts until the next reboot. 

Frequency scaling can also be disabled in the BIOS.

## Recording an execution

Running rr in "record" mode saves the execution history of your application into a "trace directory".  To invoke the recorder, run

    rr record /path/to/your/application [arguments to application]

The recorded trace is saved to the path `$_RR_TRACE_DIR/application-$n` by default. See [environment variables below](#environment-variables).

The overhead of recording an application with rr is pretty low, but it depends on the application's workload.  In general you should expect around a 1.2x to 1.4x slowdown.  At the lowest end, a purely CPU-bound tracee will incur around 1.1x-1.2x overhead.  At the high end, a tracee that just makes syscalls in a tight loop can have 4x slowdown or more.

rr forces your application's threads to execute on a single core, so your application can see an additional slowdown if it takes advantage of multicore parallelism.

The `record` token is optional.

## Debugging a recording

To debug your most recent recording, run
<pre>
$ rr replay
GNU gdb (GDB) Fedora 7.6.50.20130731-19.fc20
...
0x4cee2050 in _start () from /lib/ld-linux.so.2
(gdb)
</pre>
rr automatically spawns a gdb client and connects it to the replay server. You can set breakpoints within the gdb session, step, continue, interrupt, etc. as you would normally in gdb.

To replay a recording other than your most recently recorded one, specify its trace directory: `rr replay some-other-trace-dir`.

What's special about rr is *restarting* the replayed execution.  Within the same gdb session, you can replay execution from the beginning again by issuing the `run` command:
<pre>
(gdb) run
The program being debugged has been started already.
Start it from the beginning? (y or n) y
...
</pre>
gdb preserves all your breakpoints and debugging state across the `run` command.  Because rr replay is entirely deterministic, the restarted replay session will execute exactly the same sequence of instructions as the previous replay session.  This allows you to take debugging "shortcuts" like hard-coding memory addresses to examine: all pointers have the same values in every replaying of your recorded trace.

To more precisely target your debugging session, you can set things up so that the debugger is only spawned after a target OS process is `fork`d or `exec`d, and/or when a particular execution event is reached.  If your test suite prints a message that reveals the process IDs of launched subprocesses, like for example
<pre>
[2789] WARNING: '!compMgr', file /home/cjones/rr/mozilla-central/xpcom/glue/nsComponentManagerUtils.cpp, line 59
 ^^^^
</pre>
then you can program rr to launch gdb when that process is `exec`d by using the `-p PID` option:
<pre>
$ rr replay -p 2789
...
For application/x-test found plugin libnptest.so

--------------------------------------------------
 ---> Reached target process 2789 at event 159937.
--------------------------------------------------
...
(gdb) 
</pre>
(Remember, rr sets things up so that processes look like they have the same PID in replay as they did during recording.)  If you instead want to launch the debugger when that process is `fork`d, use the `-f 
PID` option.

rr associates an "event number" with each event it records.  You can have rr mark writes to stdio with the corresponding event number and PID that made the write by using the `-M` switch
<pre>
$ rr -M replay
...
[rr 2789 163548]LoadPlugin() /tmp/tmpUrp7e7/plugins/libnptest.so returned 5a7c7140
[rr 2789 164788][2789] WARNING: '!compMgr', file /home/cjones/rr/mozilla-central/xpcom/glue/nsComponentManagerUtils.cpp, line 59
</pre>
Note that rr tagged the output with `[rr PID EVENT-NUMBER]`, in this case 2789/164788.  You can program rr to launch the debugger at a specific event by using the `-g EVENT` options
<pre>
$ rr -M replay -g 164788
...
[rr 2789 163548]LoadPlugin() /tmp/tmpUrp7e7/plugins/libnptest.so returned 5a7c7140

--------------------------------------------------
 ---> Reached target process 0 at event 164788.
--------------------------------------------------
...
(gdb) 
</pre>

The target process and/or event persist through your debugging session.  That means issuing the gdb `run` command to restart replay will go back to the targeted process/event.
<pre>
(gdb) run
The program being debugged has been started already.
Start it from the beginning? (y or n) y
...
[rr 2789 163548]LoadPlugin() /tmp/tmpUrp7e7/plugins/libnptest.so returned 5a7c7140

--------------------------------------------------
 ---> Reached target process 0 at event 164788.
--------------------------------------------------
</pre>

If you want to restart replay at a different event, you can pass the event number as an argument to the gdb `run` command, like so
<pre>
(gdb) run 159937
...
[rr 2734 159846]For application/x-test found plugin libnptest.so

--------------------------------------------------
 ---> Reached target process 0 at event 159937.
--------------------------------------------------
</pre>

If you just want to replay your recording without attaching a debugger client, invoke
<pre>
rr -M replay -a
</pre>

### Reverse execution

Where rr really shines is reverse execution! gdb's `reverse-continue`, `reverse-step`, `reverse-next`, and `reverse-finish` commands all work under rr. They're especially powerful combined with hardware data watchpoints. For example:
<pre>
Breakpoint 1, nsCanvasFrame::BuildDisplayList (this=0x2aaadd7dbeb0, aBuilder=0x7fffffffaaa0, aDirtyRect=..., aLists=...)
    at /home/roc/mozilla-inbound/layout/generic/nsCanvasFrame.cpp:460
460   if (GetPrevInFlow()) {
(gdp) p mRect.width
12000
</pre>
We happen to know that that value is wrong. We want to find out where it was set. rr makes that quick and easy:
<pre>
(gdb) watch -l mRect.width
(gdb) reverse-cont
Continuing.
Hardware watchpoint 2: -location mRect.width
Old value = 12000
New value = 11220
0x00002aaab100c0fd in nsIFrame::SetRect (this=0x2aaadd7dbeb0, aRect=...)
    at /home/roc/mozilla-inbound/layout/base/../generic/nsIFrame.h:718
718       mRect = aRect;
</pre>
Here the "New value" is actually the value before this statement was executed, since we just executed backwards past it.

Using `watch -l` is important. Without `-l`, reverse execution is often very slow or apparently buggy, because gdb will try to reevaluate the expression as the program executes through different scopes.

### Calling program functions from gdb

When you call a program function from the debugger, rr temporarily clones the current program state, runs your function in the clone (echoing console output), and then throws away the clone so replay can continue from the state before running the function. Therefore attempting to alter program state persistently by calling functions will not work.

### Checkpointing

rr supports gdb's `checkpoint`, `restart` and `delete checkpoint` commands.

### The 'when' command

During replay, the `when` command returns rr's internal current event number. Event numbers can be used as a parameter to the `run` command. E.g.
````
(gdb) when
$12 = 33818
(gdb) run 32818
... restarts 1000 events earlier ...
````

### Debugging a child process

rr automatically records all processes forked by the initial process. Use `rr ps` to dump the pids of all processes recorded by rr. Use `rr replay -p <pid>` to attach gdb to a particular process after its first exec, or `rr replay -f <pid>` to attach gdb immediately after the fork. As a shortcut you can use `rr replay -p <filename>` to attach gdb after the first exec of `<filename>` (excluding directory components).

## Limitations

The `/path/to/your/application` image you recorded *must not change* before you replay the recording.  If the executable image changes, all kinds of bad things will happen.  You can use `rr pack` to copy the files into the trace and thus remove any dependencies the trace has on files elsewhere on your system.  Because this can substantially increase the disk space a trace requires, it is not done by default.

## Other command line options

Run `rr -H` or `rr --help` to see the most up-to-date list.  The options below are most useful for rr users, as opposed to developers of rr.

General options:
* `-V, --verbose`: log messages that may not be urgently critical to the user.
* `-M, --mark-stdio`: write "current event number" before every stdio output line (see `rr replay -g` below).

Recorder parameters:
* `-c, --num-cpu-ticks=<NUM>`: maximum number of 'CPU ticks' (currently retired conditional branches) to allow a task to run before interrupting it.
* `-e, --num-events=<NUM>`: maximum number of events (syscall enter/exit, signal, CPU interrupt, ...) to allow a task before descheduling it.

Recording traces under different scheduling params can help reproduce nondeterministic bugs.  rr's scheduler is relatively deterministic.

Replay parameters:
* `-f, --onfork=<PID>`: debug <PID> when forked
* `-p, --onprocess=<PID>`: debug <PID> when execed
* `-g, --goto=<EVENT-NUM>`: execute forward until event <EVENT-NUM> is reached before debugging

## Environment Variables

`TMPDIR`: rr needs plenty of space for temporary files, in a filesystem that is *not* mounted `noexec`. So if `/tmp` is mounted `noexec` on your system, set `TMPDIR` to a directory in a different filesystem.

`RR_TMPDIR`: rr uses this instead of `TMPDIR` if it's set. Use this to give rr a different TMPDIR than the program you're recording.

`_RR_TRACE_DIR`: set this to the directory where traces will be recorded. Defaults to `$HOME/.local/share/rr` for new users. If `$HOME/.rr` exists (e.g. after you used rr < 4.0.1), it is used instead. rr also respects [`$XDG_DATA_HOME`](http://standards.freedesktop.org/basedir-spec/basedir-spec-latest.html) environment variable if it's set.

`RR_LOG`: set `RR_LOG=<module>` to enable full logging of a specific rr module (source file name minus directories and suffix, e.g. `Task`), or `RR_LOG=all` to enable all logging.

`RUNNING_UNDER_RR`: rr sets this to `1` in recorded processes, so you can check this to trigger different behavior when running under rr.