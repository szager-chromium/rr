It would often be useful to be able to gather traces on one machine and move them to another machine for debugging.

## Known Issues

* rr tries to create hardlinks (or copies) of all used files in the trace directory, but it doesn't always do that completely (e.g. if the trace directory is on a different filesystem to system files, the hardlinks will fail and rr doesn't follow up by copying them, since this is inefficient and unnecessary for common use-cases).
* rr creates multiple hardlinks to the same file, so just copying the trace directory naively will make multiple copies of many files, bloating the trace unnecessarily.
* Copying hardlinked files is also suboptimal because tracees often don't need the entire file contents. The trace knows which file regions are needed.

rr should probably have a `rr pack` command which prepares a trace directory for transportation by copying all necessary system files into the trace, de-duping file data, and cutting out unused file data.

* `CPUID` can't be virtualized by rr so moving traces between systems with different CPUID values probably won't work; machines will need to have exactly the same CPU (including stepping etc), or very close to it. Possibly this could be worked around to some extent with some careful dynamic glibc patching.

It's also possible on at least some Intel CPUs for the kernel to trap userspace `CPUID`s using MSRs. Someone should write a kernel patch adding `prctl` commands to disable/virtualize `CPUID` like we have for `RDTSC` --- then this would be solved for rr.

* Currently rr assumes that if we give unvarying inputs to `execve` (and specify the right syscall options to disable ASLR and force compat layout), the initial address space after exec will always be the same. This might not be the case across kernel versions. This can probably be addressed in rr without much trouble by rewriting memory after exec.

There are probably unknown issues as well.

Moving traces between identically configured VM instances can probably be made to work without much trouble, since VMs are usually able to virtualize `CPUID`.