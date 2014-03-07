List of linux kernel work that would help rr.

### In-kernel buffering of syscall outparam data

Syscallbuf+vsyscall hooking works OK in practice, but the kernel can do so much better, so much more easily.  Essentially it "just" has to record copy_to_user() within syscall handlers, plus a bit of metadata.  Perhaps could be exposed through perf_event.

### Trapping on CPUID, like the RDTSC traps

Modern Intel chips can do this.

### Expose Branch Trace Store to userspace

Probably through perf_event.  There was a previous attempt to do this, but I think it fizzled out.

### Efficient way for ptrace tracers to wait on a tracee's private futex

rr wants to use this for efficiently waiting on CLONE_CLEAR_CHILDTID futexes.

### Bugfix: pwrite64/pread64 modify registers in a way visible to userspace

Counter to the syscall ABI.  Bug that we shouldn't let slip through the cracks.  [Ubuntu bug](https://bugs.launchpad.net/ubuntu/+source/linux-lts-quantal/+bug/1206746) isn't moving, need to test on kernel HEAD and upstream to kernel folks.

### Bugfix: agree a kernel/userspace contract for ptrace traps of PR_TSC_SIGSEGV

See #692: the kernel munges sigstate to enable SIGSEGV traps when SIGSEGV is blocked, but apparently doesn't un-munge the state.

### Bugfix: raise PTRACE_O_SYSGOOD traps when PTRACE_SINGLESTEP'ing into syscalls?

See #212.