This page describes the plan to add x86-64 support to rr.

## Step 1: Figure out how to record "virtual syscalls"

The x86-64 kernel ABI exposes functionality that's conceptually pretty simple, but the terminology around it is extremely confusing.  Let's call this thing "pseudosyscalls".

The way pseudosyscalls work is, the kernel maps a special code page called the "vDSO" into userspace processes.  The vDSO acts like a regular shared object, with ELF information and so forth, but it doesn't exist anywhere on disk.  Relevant to this discussion, the vDSO exposes some helper functions that are named as `__vdso_foo()`.

The `__vdso_foo()` functions implement syscalls that are performance-critical and don't require a privilege escalation to execute.  The two most interesting ones are `__vdso_gettimeofday()` and `__vdso_clock_gettime()`.  Instead of trapping/entering the kernel, these helpers instead read non-security-critical kernel data that's mapped into userspace and use that data to finish the syscall.  For example, `__vdso_clock_gettime()` reads the kernel's idea of the clock and then writes its outparams accordingly.

So glibc's implementation of the POSIX `clock_gettime()` interface will end up calling `__vdso_clock_gettime()`, and not entering the kernel.

This is super cool and fast and so forth but a major problem for rr.  This is the kind of shared-memory race that rr can't record efficiently, in general.  We have some options to resolve this issue; experimentation is required to see which is the best approach.

### Possible solution: interpose __vdso_* wrappers

Simple solution: using `clock_gettime()` as an example, add an implementation of `__vdso_clock_gettime()` to `librrpreload.so`.  This implementation would call the rrpreload `sys_clock_gettime()` helper, which usually results in an untraced buffered syscall.  Then we hope that at dynamic link time, glibc resolves the `__vdso_clock_gettime()` symbol from librrpreload instead of the vDSO.

It's known that rr is unable to interpose the kernel entry point helper, `__kernel_vsyscall()`, which is exposed similarly to the pseudosyscalls.

This implementation falls over if tracees explicitly resolve __vdso_* symbols from the vDSO, or if rr is unable to interpose its own __vdso_* implementation.  It also falls over if new helpers are exposed through the vDSO.

### Possible solution: interpose libc wrappers

Also extremely simple: take the `clock_gettime()` case above.  Add a `clock_gettime()` implementation to `librrpreload.so`.  That implementation would call the rrpreload `sys_clock_gettime()` helper, i.e., make the buffered untraced syscall.  Do this for the other __vdso_* interfaces.

This scheme falls over if there are glibc-internal calls to `__vdso_clock_gettime()` that don't go through a public POSIX interface.  It also falls over if there are __vdso_* symbols not exposed under POSIX APIs that rrpreload can interpose on.  Both of these are highly probable.

### Possible solution: trap on access of kernel data

Map the kernel data page that the __vdso_*() helpers read from at PROT_NONE.  Access of the page raises a trap to the rr tracer.  rr does the read on  behalf of the tracee, updates its registers appropriately, records the data to trace, then sets its $ip past the load instruction.

This is known to be slow in general.  However, it may be good enough for the (likely) single loads needed for the pseudosyscalls.  We would have to bake load-instruction semantics into rr, which is annoying but not a major obstacle.

I don't know of any ways this could fall over.

### Possible solution: monkeypatch more of the vDSO

rr already monkeypatches the `__kernel_vsyscall()` helper in the vDSO to jump into rr.  Our nuclear option for the __vdso_*() pseudosyscalls is to similarly monkeypatch them to jump into rr code.  This is far more delicate and complicated than the __kernel_vsyscall patch, so should only be the absolute last resort.

I don't know of any ways this could fall over.

## Step 2: replace hard-coded x86-isms with arch-neutral indirection

rr has many direct references to x86 register names, like `regs.eax`.  We would need to replace this with a layer of indirection that hides the raw register manipulation.

Further, we'd like to eventually support mixed-architecture tracees, where some are running 32-bit images and others 64-bit images.  That means we need yet another level of indirection.

The helper interface might look like
<pre>
class Registers {
  virtual int& call() = 0;
  virtual long& arg1() = 0;
  virtual void*& arg1p() = 0;
};
</pre>

We would then create implementations `RegistersX86` and `RegistersX64` or whatever.  Access to the `Task` registers would go through the virtual method calls to the right implementation.

With that in hand, we would finally "just" rewrite all of the direct uses of register names to the helper :).