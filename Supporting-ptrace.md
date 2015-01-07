rr supports recording processes that use a subset of `ptrace` --- enough to record Firefox's crash reporter (which uses Google Breakpad). Functionality currently supported:
* `PTRACE_ATTACH`
* `PTRACE_DETACH` (with no signal)
* `PTRACE_CONT` (with no signal)
* `PTRACE_GETREGS`
* `PTRACE_GETFPREGS`
* `PTRACE_GETFPXREGS`
* `PTRACE_PEEKUSER`
* `PTRACE_PEEKDATA`

Linux doesn't support multiple ptracers for the same thread, and that wouldn't make sense anyway, so rr is the sole real ptracer of tracee threads. When one tracee tries to ptrace another tracee, rr emulates the ptrace functionality ourselves. So rr tracks an "emulated ptracer" for each task, and when ptrace events occur in a task with an emulated ptracer, we forward those events to the ptracer. Currently only signal events are supported.

Supporting more ptrace functionality would be interesting:
* Supporting more calls to read/write process state would be pretty easy.
* Resuming tracees with a signal would be tricky but not too hard. Need to create a new signal-delivery path that avoids triggering a new ptrace-stop.
* Supporting breakpoints is probably not very hard.
* `PTRACE_SINGLESTEP` would be tricky but not too hard.
* `PTRACE_SYSCALL` would be very hard. The basic problem is that right now, in certain syscall-handling states, the tracee is not switchable and we depend on it not being switched away from. With `PTRACE_SYSCALL` we must *always* be able to switch away from the tracee when we emulate a ptrace-stop and need to run its ptracer. So to implement `PTRACE_SYSCALL` we would first need to eliminate `PREVENT_SWITCH`.
* Other miscellaneous ptrace events like `PTRACE_EVENT_EXIT` and `PTRACE_EVENT_CLONE` would have similar problems.