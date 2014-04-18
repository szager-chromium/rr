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

## Strawman superfork algorithm
<pre>
for each process(address space) p:
  [*] inject a remote fork() syscall into p's main thread
  [**] run the fork() to create p'
  # we may also want to set up p' to deliver SIGCHLD to a non-default process.
  # It may be annoying to have the superfork sending SIGCHLD to its source tree.

  for each mmap m in p:
    if m is a shared mapping:
      [*] unmap m in p'  # injecting and running the required remote syscalls
      copy the EmuFS file f backing m to f'
      re-map m with the same attributes, but referring to f'

  for each task t in p that's not the thread-group leader:
    [*] inject a clone() syscall into the thread-group leader task
    set up the clone() to set the current cleartid futex address, stack pointer, and TLS addr from t
    run the clone() to create task t'
    set the registers of t' to be the same as t

  [*] set the registers of the thread-group leader p' to be the same as p
</pre>
This algorithm attempts to use the `fork()` approach discussed above to clone memory contents.  Because it takes that approach, it *will not* preserve the task tree.  And in fact, the superfork will be a child of its origin, which will look odd in `pstree`.  However, the attraction of CoW memory outweighs my desire for a clean `pstree`.

If we prefer to use the superfork for throwaway uses like #604 and #605, then the process tree will remain looking somewhat saner (i.e. the superfork will always die first, so the original task tree will always be intact).

This algorithm elides the mechanics of "restarting" emulated syscalls that are "interrupted" by the superfork operation.

There are asterisks `*` and `**` in the algorithm above that represent unknowns.  They are

* `*`: at these places, we rely on injecting code into the cloned process's main thread.  This is absolutely required for the `fork()` call, because we don't have any other way of setting up the new process's main thread stack/heap.  (For the other injected syscalls, it doesn't particularly matter which thread runs the syscalls.)  However, I think it's possible in theory for a linux process's main thread to die before its child threads.  We need to experiment to see if this can happen in practice.  If so, it's possible to fix the algorithm above: we fork from an arbitrary thread in the original tree, do the setup from that new main thread, and then SYS_exit that main thread instead of restoring its registers.
* `**`: if `p` was configured with a `CLEARTID` futex when it itself was forked, then we won't be able to set up a corresponding `CLEARTID` futex for `p'` using this simple algorithm.  (I think that glibc does this, for a reason I don't understand.)  If we need to support this in practice, then I don't believe we'll be able to use this efficient fork-based algorithm.  I think instead we'll have to walk the tracee process tree in top-down order, and replay the necessary fork/clone calls to recreate it in the superfork.  Then memory contents will need to be manually cloned.

## Prior art (CRIU)

[CRIU](http://criu.org/Main_Page) seems to be the state-of-the-art in linux checkpoint/restore software.  CRIU aims for fully-generic checkpointing, which is massive overkill for rr's replay checkpointing.  Even so, it would have been nice to use reuse CRIU, but this doesn't appear possible.  However, if the rr superfork implementation gets stuck, CRIU may be a source of inspiration on how to get un-stuck.