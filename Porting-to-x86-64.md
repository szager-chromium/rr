This page describes the plan to add x86-64 support to rr.
See [Nathan's blog article](https://blog.mozilla.org/nfroyd/2014/10/30/porting-rr-to-x86-64/#compiling-rr-for-64bit) for some notes on progress.

# General sketch:

1. Build rr 64-bit (but still only supporting 32bit tracees)
 * Add multi-arch support to Registers and ExtraRegisters, to break dependence on `user_regs_struct`
 * Finish moving `SYS_` numbers to Arch
 * Fix other breakage we'll inevitably find
We don't want to support 64bit tracees from 32bit rr, but we probably should keep being able to build rr 32-bit since there are some static checks for kernel_abi.h correctness that that enables.

2. Support basic record/replay of 64bit tracees --- no syscallbuf
 * Add tracee arch detection
 * 64bit trace records
 * Audit code for remaining issues that need templated code paths for 64bit tracees
 * Audit code to make sure we use `uintptr_t`/`intptr_t` for machine words, in non-templated code
 * Build test suite in both 32bit and 64bit mode and run both sets of tests

3. gdb support for 64bit tracees

4. syscallbuf
 * Build separate `librrpreload` for 32-bit and 64-bit targets and get the right ones injected, and support both from rr
 * Adding syscallbuf patching in lieu of vsyscall

# Specific technical issues

## Figuring out how to record "virtual syscalls"

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

## Refactoring away hard-coded x86-isms with arch-neutral indirection

We have introduced `kernel_abi.h` defining one class per architecture. Kernel ABI structs and enums are defined as members of these classes. Code can be templated over the architecture class. The process of converting code to use this template structure as needed (e.g. to remove all uses of `SYS_` #defines) is ongoing.

There is also a `supported_arch` enum and `arch()` getters to support dynamic architecture checks when appropriate.

## Making remaining x86-isms conditional on tracee address space type

We have some further x86-isms like monkeypatching the `__kernel_vsyscall()` interface that need to be made conditional on tracee image type.  Another one is decoding the instruction that caused a `SEGV` trap to see if it's a `rdtsc`.  (Hopefully the encoding is the same in x86-64, but we need to check.)

## Checking the Task's image type at exec time and updating accordingly

This is essentially the "turn everything on" step.  Note, it's possible for a 32-bit process to exec a 64-bit image and vice versa, so Tasks have to be able to change personality.