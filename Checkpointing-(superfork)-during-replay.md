This page describes the implementation plan for adding the capability to create a semantic copy of an entire tracee tree (including copying files etc. as necessary), an operation we're calling "superfork".  This operation is the mechanism needed to create checkpoints of the tracee tree during replay.  Replay checkpoints are in turn needed to optimize the gdb `restart` workflow, and allow running arbitrary code in tracee processes.  The actual work is tracked in #603.

Note, in the discussion that follows, "superfork" will be used both as a verb and a noun.  The verb describes the process of cloning a tracee tree, and the noun refers to that tracee tree cloned by the superfork verb.  The usage should be clear from context.  Apologies if this causes any confusion.

## Ensuring tracees are at a stable execution point

By "stable", I mean that rr can restore the superfork to the state of the original tracee tree.  Ensuring this in replay is relatively easy compared to recording.  The only thing rr needs to do to ensure the tracees are stable (AFAIK) is preventing any of them from being in an "executed" syscall (as opposed to emulated).  If a tracee were in an executed syscall at superfork time, the tracee would have to be interrupted out of the syscall in such a way the syscall could be restarted in both the superfork and the original tracee tree.  This may be difficult, and could cause divergence as well if not done carefully.

Luckily, all syscalls currently executed during replay are atomic.  So the stability problem reduces to ensuring that a superfork isn't initiated in the middle of an atomic syscall, which is relatively easy.

It's not yet known what will happen when an emulated syscall is "interrupted" by a superfork.  However, rr can always manually "finish" emulated syscalls that are "interrupted" by superfork, by (i) tracking the interrupted-ness state; (ii) issuing the PTRACE commands based on the state, or not; (iii) manually updating the register file to "finish" the interrupted call.

## Cloning a tracee tree

The first superfork step is to clone the tracee's task tree: create a new set of linux tasks that correspond to the old set.  rr currently preserves the parent/child relationships from recording in the replayed execution (because it's easier to do that).  There's no particular use case demanding that.  It's somewhat useful for reading `pstree` output, and debugging rr itself.  However, preserving the parent/child edges in a superfork is somewhat trickier than not.

Here are the invariants constraining tracee-tree cloning

* Each process `p` (thread-group leader) must have a clone process `p'` in the superfork.  (Duh.)
* Each non-process task `t` (non-thread-group leader) must have a clone task `t'` in the superfork.  (Duh.)
* Each clone task `t'` must have be in the address space `p'` corresponding to the original `t`/`p` relationship.  (Duh.)
* For each parent task `p'`: for each child task `c'`': if `c` was cloned with `CLONE_CLEARTID`, then `p'` must be configured to hold a `CLEARTID` futex for `c'`.  Or, CLEARTID must be fully emulated in replay.
* For each task `t` created with `CLONE_VM`, its clone `t'` must be created with the same stack pointer.  (This is thought to be required, but it might not be.)
* Fir each task `t' created with `CLONE_SETTLS`, its clone `t'` must be created with the same TLS pointer.  (This is thought to be required, but it might not be.)

## Cloning process memory

In theory, this sounds easy: recreate all the mmaps in the cloned process and copy the memory contents.  In practice there are some subtleties.  Here are the invariants; they're listed as if for a single process, but of course they apply to all address spaces being superforked.

* Each memory mapping `m` must have a corresponding clone `m'` in the clone, covering the same region, with the same protection and flags, and with the same memory contents.  (Duh.)
* The memory backing `m'' must be a *semantic copy* of the region backing `m`.  For private mappings, this is natural and obvious.  For shared mappings, the resource backing `m` must itself be cloned, and then the cloned region mapped at `m'`.

An interesting question is whether it's better to re-`mmap` all regions in the superfork and copy memory contents over, or attempt to do something clever with `fork`.  The fork approach is better in theory because the superfork and its superparent will share as many memory pages as possible.  For that reason, it should also be much faster.  And with programs like Firefox that map a *lot* of memory, can have many child processes, and expect to share a significant portion of mapped memory among all the processes, the fork-to-share approach may be important.

## Cloning file resources

Luckily, in replay, the only way a tracee can directly access file resources is by replaying an mmap call that create a shared region.  This causes rr to create a file in its "emulated file system", and then that file is mapped on behalf of the tracee.  So wrt the invariants mentioned above, cloning file resources just reduces to the problem of cloning a tracee's EmuFS.  In other words, no additional invariants are added here.