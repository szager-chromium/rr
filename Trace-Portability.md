## Using `rr pack`
rr traces can be made portable with the `rr pack` command. This will pack the trace directory to eliminate duplicate files and to include all files needed for transportation. As such, it allows you to gather traces on one machine and move them to another machine for debugging. It also makes it easy to run recordings for different versions of the software, since the trace no longer relies on hard links to the executable files.

## Issues
According to [Robert O'Callahan](https://robert.ocallahan.org/2017/09/rr-trace-portability.html)
> The user is responsible for ensuring the destination machine supports all instructions and other CPU features used by the recorded program. 
>
> At some point we could add an rr feature to mask the CPUID values reported during recording so you can limit the CPU features a recorded program uses.
>
>CPUID faulting is supported on most modern Intel CPUs, at least on Ivy Bridge and its successor Core architectures. Kyle also added support to upstream Xen and KVM to virtualize it, and even emulate it regardless of whether the underlying hardware supports it. However, VM guests running on older Xen or KVM hypervisors, or on other hypervisors, probably can't use it. And as mentioned, you will need a Linux 4.12 kernel or later. 