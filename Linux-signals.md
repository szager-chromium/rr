## Signal handlers

Each linux task *points at* a sighandler table.  The table says whether signals have either: default disposition, are ignored, or have a userspace handler.  The tricky bit is that tasks can share tables, according to the following rules

0. After `exec`, the task gets a copy of its previous table, *except* all handled signals are reset to default (ignored signals aren't changed).
0. After `fork`, the new child gets a copy of its parent's table.
0. After `clone`, the new child gets a copy of its parent's table, *unless* CLONE_SIGHAND is passed.  In that case the table is shared.

To the best of my knowledge, only the `signal` and `sigaction` syscalls can alter sighandler tables.  Parsing `struct sigaction` is a bit annoying, but we only need the first word from the struct (the sa_handler / sa_sigaction field).

To implement these semantics in rr, we can create a struct representing the sighandler table and give each task a (ref)pointer to one.  When the first child is forked, we know what its table will be: a copy of the rr parent's.  From there on we assign sighandler tables to tasks by the rules above.  Upon a clone(CLONE_SIGHAND), we simply give the new task a pointer to its parent's handler table and bump the refcount.

In my testing, ptrace didn't notify rr of signals queued because they were masked off.  So it seems that we don't need to track each tasks signal mask, which would add a lot of complication.

## Restart semantics

These are the kinds of signal "interruption" that are relevant to this discussion

0. Signal delivered, but ignored; i.e., no userspace handler invoked and default action not taken.
0. Signal delivered and userspace handler invoked.
0. Signal ready to deliver but ptrace tracer declines to deliver it.

Wrt to those interruptions, syscalls fall into the following classes

0. Never interrupted, from userspace's perspective; never return -EINTR.  Example: `clock_gettime()`.
0. Interrupted when a delivered signal invokes a user sighandler; return -EINTR in that case.  *Unless* the signal handler is registered with SA_RESTART, in which case the syscall behaves as if the signal was ignored; i.e. the syscall is automatically restarted.  Example: `read()`.
0. Always interrupted when a delivered signal invokes a user sighandler; always return -EINTR.  SA_RESTART has no effect.  Automatically restarted on delivery of ignored signal.  Example: `nanosleep()`.

When a syscall is interrupted by a signal, the kernel first exits the original syscall with a special error code.  Tasks themselves can never observe these error codes (except when there are kernel bugs), but ptrace tracers can observe them in tracees.  The codes are listed below.

After exiting the original syscall, the user sighandler is invoked, if there was one.  (Which can recursively be interrupted by signals in other syscalls, etc.)  Next the kernel checks to see if the original syscalls should be restarted per the rules above.

Different mechanisms are required to restart different syscalls.  Some syscalls can be restarted simply by invoking the syscall again with the same arguments.  `read()` is one example.  Other syscalls need special treatment, for example if their arguments are time varying.  One example is nanosleep: if a 2-second sleep is interrupted by a signal 1 second into the wait, then restarting the sleep with the same arguments would result in a 3 second wait.  Linux solves this by creating a "restart block" for the syscall, and the syscall impl writes updated args that can be used to restart the syscall to the restart block.  Then to restart the syscall, instead of simply trapping to the original syscall, the kernel sets things up so that the task enters the kernel through a special SYS_restart_syscall entry point, which resumes by using the restart block.

The magic restart codes are

* ERESTARTNOHAND: don't attempt to do anything clever with the signal, just always return EINTR.  **TODO**: not sure if this is entirely correct.
* ERESTARTNOINTR: always restart the syscall with no errno.
* ERESTARTSYS: if the signal action includes SA_RESTART, use the ERESTARTNOINTR behavior.  Otherwise, use the ERESTARTNOHAND behavior.
* ERESTART_RESTARTBLOCK: restart the syscall by using the restart block, through a call to `SYS_restart_syscall`.

Examples: an interrupted `read()` results in ERESTARTSYS.  An interrupted `nanosleep()` results in ERESTART_RESTARTBLOCK.  An interrupted `pselect()` results in ERESTARTNOHAND.

Note: syscalls that are restarted because they were SIG_IGN or were registered as SA_RESTART are restarted *with their original arguments*, *not* the fudged arguments that rr sets up (redirection to scratch buffers).

`ptrace` makes things slightly more complicated.  In addition to the SIG_IGN and SA_RESTART mechanisms available to tracees, ptrace allows the *tracer* to decline to deliver signals.  When tracers are notified of pending tracee signals, the tracee has already exited any in-progress syscall with one of the codes above.  Then the tracer can either deliver the signal or not.  If the signal isn't delivered, then the tracee behaves *almost* as if the signal was SIG_IGN or SA_RESTART, with these exceptions

* the syscall is entered through the original entry point (as seen by the tracer), SYS_poll etc; *not* the SYS_restart_syscall entry point
* the restarted syscall enters with the *fudged* arguments, the ones that may be redirected to scratch
* as for SIG_IGN/SA_RESTART, the exit point is observed to be SYS_restart_syscall by the tracer

Squelching incoming signals is generally a silly thing to do, but rr is forced to do this because it uses some tracee signals for internal implementation details (rdtsc, time-slice interrupts, and desched notifications, to be precise).

Sources
* http://stackoverflow.com/questions/9576604/what-does-erestartsys-used-while-writing-linux-driver
* `man ptrace`
* linux source

## SIG_IGN signals restarting blocked syscalls on non-main threads

Let's say that a `read()` blocks and is interrupted by an SA_RESTART signal, *without* a signal handler.  The read exits with ERESTARTSYS, then rr sees a signal event and attempts to single-step the tracee to determine whether a sighandler is entered.  The subsequent waitpid on the thread just hangs.

Repeatedly `waitpid(-1)`ing shows the main thread running a few of its syscalls.  Finally the original thread becomes runnable with status trace-trap.  However, it shows 0 instructions retired, which is what rr keys off of to know if a signal handler was established.

Also, the `read()` from the original thread returns 1 ... **but without entering read() again**.  So from rr's perspective, the read exited with ERESTARTSYS and might restart, but to the tracee it looks like the read magically returned 1.

Restarting the blocked thread with PTRACE_SYSCALL delivering the blocked signal works more as expected: the read() restarts, the thread blocks, and we switch it out.

## ABRT (core-dumping signals?)

[This issue](https://github.com/mozilla/rr/issues/288): an ABRT raised on a non-main thread appears to block the non-main thread until the main thread is exited.  Behavior not observed with fatal, but not core-dumping, signal TERM.

## Tentative algorithm

<pre>
// Step 0
finish_syscall(task, syscall):
  if ret is ERESTART*:
    task.push_event(restart_syscall(ret, syscall))

// Step 1
signal_pending(task, sig):
  go_to_happy_place(task)

  if task.sig_disposition(sig) == HANDLER:
    task.events.push(sighandler(sig))

  task.resume(CONT or SYSCALL, sig)

enter_syscall(task, syscall):
  if syscall is sigreturn:
     task.events.pop_sighandler()
     // finish, ...
     return

  if task.events.top is restart_syscall:
    restarted_syscall = task.events.pop()
    if syscall ~= restarted_syscall:
      log "restarting %s"
      syscall = restarted_syscall
    else:
      log "not restarting %s"
      restarted_syscall.ret = -EINTR
      // process_syscall(), or otherwise cleanup restarted_syscall state
      record_event(restarted_syscall)
  
  task.push(syscall)
  // continue with syscall  
</pre>

## Example

Sequences of curated events, to get a sense of what's going on.

### Ignore (or no handler)

    -->enter read
        [block]
        [recv USR1/CHLD]
    <--exit read: return -512 (ERESTARTSYS)
    -->sighandler
        -->enter read
            [block]
        <--exit read: return 1
    <--sigreturn
    HANG: trying to single-step into handler

### Default semantics (no SA_RESTART), one level of interrupt

    -->enter read
        [block]
        [recv USR1]
    <--exit read: return -512 (ERESTARTSYS)
    -->sighandler
        -->enter read
        [block]
        <--exit read: return 1
    <--sigreturn
    (continue normally ...)

### SA_RESTART, one level of interrupt

    -->enter read
        [block]
        [recv USR1]
    <--exit read: return -512 (ERESTARTSYS)
    (-->sighandler)
        -->enter read
            [block]
        <--exit read: return 1
    <--sigreturn
    -->enter read
        [block]
        (HANG; expected; waiting for progress in scheduler)

### Default semantics (no SA_RESTART), two levels of interrupt

    -->enter read
        [block]
        [recv USR1]
    <--exit read: return -512 (ERESTARTSYS)
    (-->sighandler)
        -->enter read
            [block]
            [recv USR2]
        <--exit read: return -512 (ERESTARTSYS)
        (-->sighandler)
            -->enter read
                [block]
            <--exit read: return 1
        <--sigreturn
    <--sigreturn
    (continue normally ...)

### SA_RESTART, two levels of interrupt

    -->enter read
        [block]
        [recv USR1]
    <--exit read: return -512 (ERESTARTSYS)
    (-->sighandler)
        -->enter read
            [block]
            [recv USR2]
        <--exit read: return -512 (ERESTARTSYS)
        (-->sighandler)
            -->enter read
                [block]
            <--exit read: return 1
        <--sigreturn
        -->enter read
            [block]
            (HANG; expected; waiting for progress in scheduler)

### SA_RESTART, two levels of interrupt, three writes to read() calls

All three `read()` syscalls are restarted and successfully read a byte, as the above case would suggest.