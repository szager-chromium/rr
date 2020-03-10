rr supports recording processes that use a subset of `ptrace` --- enough to record Firefox's crash reporter (which uses Google Breakpad), and rr replaying a simple test. Functionality currently supported:
* `PTRACE_ATTACH`
* `PTRACE_DETACH` (with no signal)
* `PTRACE_CONT` (with no signal)
* `PTRACE_SINGLESTEP` (with no signal)
* `PTRACE_GETREGS`
* `PTRACE_GETFPREGS`
* `PTRACE_GETFPXREGS`
* `PTRACE_PEEKUSER`
* `PTRACE_PEEKDATA`
* Breakpoints
* Ptrace options: `PTRACE_O_TRACEEXEC`, `PTRACE_O_TRACESYSGOOD`, `PTRACE_O_TRACEEXIT`, `PTRACE_O_TRACEFORK`, `PTRACE_O_TRACECLONE`, `PTRACE_O_TRACEVFORK`

Linux doesn't support multiple ptracers for the same thread, and that wouldn't make sense anyway, so rr is the sole real ptracer of tracee threads. When one tracee tries to ptrace another tracee, rr emulates the ptrace functionality ourselves. So rr tracks an "emulated ptracer" for each task, and when ptrace events occur in a task with an emulated ptracer, we forward those events to the ptracer. Currently only signal events are supported.

Supporting more ptrace functionality would be interesting:
* Supporting more calls to read/write process state would be pretty easy.
* `PTRACE_SYSCALL` would be very hard. The basic problem is that right now, in certain syscall-handling states, the tracee is not switchable and we depend on it not being switched away from. With `PTRACE_SYSCALL` we must *always* be able to switch away from the tracee when we emulate a ptrace-stop and need to run its ptracer. So to implement `PTRACE_SYSCALL` we would first need to eliminate `PREVENT_SWITCH`.

Ultimately it would be neat to have rr support enough `ptrace` functionality that it can record itself recording a testcase; however, this seems like a ton of work for little practical benefit, since if there is a recording bug we probably won't be able to record the recording either.