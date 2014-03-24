## Good First Bugs
https://github.com/mozilla/rr/issues?labels=goodfirstbug&page=1&state=open

## Simple Projects
* Remove dependencies on libdisasm and libpfm (issues #974 and #954)
* When btrfs is being used, use the equivalent of "cp --reflink" to cheaply copy all input files into the trace directory (issue #602)

## Small But More Difficult Projects
* Figure out if Haswell has a deterministic performance counter we can use (issue #973)
* Compress trace data (issue #700)

## Big Important Projects
* Port to x86-64 (issue #606)
* Implement fork-based checkpointing of an entire replay session (issue #603)