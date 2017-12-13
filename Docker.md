rr can run inside [docker](https://www.docker.com/). If rr can run on the same system outside of docker, simply start your container with the additional arguments ```--cap-add=SYS_PTRACE --security-opt seccomp=unconfined```.  You should be aware of the security implications of these flags before using them.

rr needs to be able to ptrace the tracee process group. By [default](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities) docker drops the ```SYS_PTRACE``` capability which prevents ptrace from being used inside the container.  That capability must be restored.

Docker also includes by default a seccomp profile that [disables](https://docs.docker.com/engine/security/seccomp/#significant-syscalls-blocked-by-the-default-profile) a number of syscalls needed by rr, including ```ptrace```, ```perf_event_open```, and ```process_vm_writev```. It would be possible to audit rr and produce a seccomp profile for Docker that is the default profile with only the syscalls rr requires added back in, but we have not done that work. ```--security-opt seccomp=unconfined``` will skip all seccomp filtering of the container's processes.

