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

## Step 2: Refactor away hard-coded x86-isms with arch-neutral indirection

Currently pointers to tracee memory have the same type as pointers to rr memory. I find this makes code harder to read. I suggest introducing a remote_ptr&lt;T&gt; type which is a pointer to T in tracee memory. This would be a struct wrapping a uintptr_t. We'd use it for all tracee pointers.

* Then we can have a Task::record(remote_ptr&lt;T&gt; p) variant that records sizeof(T) bytes.
* read_mem can type-check its remote_ptr parameter by taking (remote_ptr&lt;T&gt;, T*).
* We can also have a read_mem(remote_ptr&lt;T&gt;) variant that returns T directly, replacing read_word.

rr has many direct references to x86 register names, like `regs.eax`.  We would need to replace this with a layer of indirection that hides the raw register manipulation.

Further, we'd like to eventually support mixed-architecture tracees, where some are running 32-bit images and others 64-bit images.  That means we need yet another level of indirection.

The helper interface might look like
<pre>
class Registers {
  virtual int& call() = 0;
  virtual long& arg1() = 0;
  virtual remote_ptr<T>& arg1p() = 0;
};
</pre>

We would then create implementations `RegistersX86` and `RegistersX64` or whatever.  Access to the `Task` registers would go through the virtual method calls to the right implementation.

With that in hand, we would finally "just" rewrite all of the direct uses of register names to the helper :).  Change the SYSCALL_DEF macros to take syscall arg indices instead of register names.

Note: syscall numbers aren't all the same across x86 and x64.  So there need to be *two* `switch` statements that dispatch syscall processing, one for x86 and one for x64.  The common cases `SYSCALL_DEFN()` should be generated from the same code, and the irregular cases should be templatized on architecture.

This probably means rr will have to maintain its own list of syscalls, as it will have to do for arch-dependent kernel ABI structs.  See below.

## Step 3: Make remaining x86-isms conditional on tracee address space type

We have some further x86-isms like monkeypatching the `__kernel_vsyscall()` interface that need to be made conditional on tracee image type.  Another one is decoding the instruction that caused a `SEGV` trap to see if it's a `rdtsc`.  (Hopefully the encoding is the same in x86-64, but we need to check.)

## Step 4: Ensure rr records the correct-sized structure for outparams

Problems

0. Many kernel ABI structs have different layouts for 32-bit vs 64-bit, e.g. iovec, stat64, getrusage
0. Some kernel ABI structs of different layouts can be used within the same architecture, for different syscalls.

I don't think we can pull the definitions of these structs from #include files in a way that makes both the 32-bit definition and the 64-bit definition visible to rr. It would also be a little painful to use two separate definitions. So how about:

* Create ArchX86 and ArchX86_64 classes, each containing typedefs for architecture-dependent types like "long" and pointers (e.g. ArchX86(_64)::Long, ArchX86(_64)::Ptr<T>).
* Create a header file kernel_ABI.h with templated definitions for kernel ABI structs, in the rr namespace, e.g.
```C++
template <class Arch>
struct iovec : public Arch {
  Ptr<void> iov_base;
  SizeT iov_len;
};
```

* Introduce implicit conversions from Arch::Ptr<T> to remote_ptr<T>
* Make some functions like rec_process_syscall wrappers around templatized helper functions taking Arch as a template parameter. Then in syscall_defs.h we can write
```C++
SYSCALL_DEF1(getrusage, EMU, rusage<Arch>, 2 /* 2nd syscall arg */)
```

## Step 5: Check the Task's image type at exec time and update it accordingly

This is essentially the "turn everything on" step.  Note, it's possible for a 32-bit process to exec a 64-bit image and vice versa, so Tasks have to be able to change personality.