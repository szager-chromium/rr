List of linux kernel work that would help rr.

### In-kernel buffering of syscall outparam data

Syscallbuf+vsyscall hooking works OK in practice, but the kernel can do so much better, so much more easily.  Essentially it "just" has to record copy_to_user() within syscall handlers, plus a bit of metadata.  Perhaps could be exposed through perf_event.

### Expose Branch Trace Store to userspace

Probably through perf_event.  There was a previous attempt to do this, but I think it fizzled out.

### Efficient way for ptrace tracers to wait on a tracee's private futex

rr wants to use this for efficiently waiting on CLONE_CLEAR_CHILDTID futexes.

### Bugfix: pwrite64/pread64 modify registers in a way visible to userspace

Counter to the syscall ABI.  Bug that we shouldn't let slip through the cracks.  [Ubuntu bug](https://bugs.launchpad.net/ubuntu/+source/linux-lts-quantal/+bug/1206746) isn't moving, need to test on kernel HEAD and upstream to kernel folks.

### Bugfix: agree a kernel/userspace contract for ptrace traps of PR_TSC_SIGSEGV

See #692: the kernel munges sigstate to enable SIGSEGV traps when SIGSEGV is blocked, but apparently doesn't un-munge the state.

### Bugfix: raise PTRACE_O_SYSGOOD traps when PTRACE_SINGLESTEP'ing into syscalls?

See #212.  Alternatively, create a new ptrace request analogous to PTRACE_SINGLESTEP_SYSEMU, perhaps PTACE_SINGLESTEP_SYSCALL.

### Bugfix: are drivers really allowed to use userspace stack as scratch memory?

See #876.  That's generally not a good idea because it can let sensitive data leak into userspace.

### Bugfix?: ability to open perf_event_open() fds through /proc/TID/fd/x

See #603.  We fall back on using SCM_RIGHTS, but it's considerably more complicated than needs to be.

### Bugfix?: wait status 0x80007f seems invalid

See #1198.  This wait_status (in response to a PTRACE_INTERRUPT) seems to contradict documentation. It's received when we interrupt a task blocked in an untraced system call.