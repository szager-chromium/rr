This page describes the implementation plan for adding the capability to create a semantic copy of an entire tracee tree (including copying files etc. as necessary), an operation we're calling "deepfork".  This operation is the mechanism needed to create checkpoints of the tracee tree during replay.  Replay checkpoints are in turn needed to optimize the gdb `restart` workflow, and allow running arbitrary code in tracee processes.  The actual work is tracked in #603.

Note, in the discussion that follows, "deepfork" will be used both as a verb and a noun.  The verb describes the process of cloning a tracee tree, and the noun refers to that tracee tree cloned by the deepfork verb.  The usage should be clear from context.  Apologies if this causes any confusion.

## Ensuring tracees are at a stable execution point

By "stable", I mean that rr can restore the deepfork to the state of the original tracee tree.  Ensuring this in replay is relatively easy compared to recording.  The only thing rr needs to do to ensure the tracees are stable (AFAIK) is preventing any of them from being in an "executed" syscall (as opposed to emulated).  If a tracee were in an executed syscall at deepfork time, the tracee would have to be interrupted out of the syscall in such a way the syscall could be restarted in both the deepfork and the original tracee tree.  This may be difficult, and could cause divergence as well if not done carefully.

Luckily, all syscalls currently executed during replay are atomic.  So the stability problem reduces to ensuring that a deepfork isn't initiated in the middle of an atomic syscall, which is relatively easy.

It's not yet known what will happen when an emulated syscall is "interrupted" by a deepfork.  However, rr can always manually "finish" emulated syscalls that are "interrupted" by deepfork, by (i) tracking the interrupted-ness state; (ii) issuing the PTRACE commands based on the state, or not; (iii) manually updating the register file to "finish" the interrupted call.

## Cloning a tracee tree

The first deepfork step is to clone the tracee's task tree: create a new set of linux tasks that correspond to the old set.  rr currently preserves the parent/child relationships from recording in the replayed execution (because it's easier to do that).  There's no particular use case demanding that.  It's somewhat useful for reading `pstree` output, and debugging rr itself.  However, preserving the parent/child edges in a deepfork is somewhat trickier than not.

Here are the invariants constraining tracee-tree cloning

* Each process `p` (thread-group leader) must have a clone process `p'` in the deepfork.  (Duh.)
* Each non-process task `t` (non-thread-group leader) must have a clone task `t'` in the deepfork.  (Duh.)
* Each clone task `t'` must have be in the address space `p'` corresponding to the original `t`/`p` relationship.  (Duh.)
* For each parent task `p'`: for each child task `c'`': if `c` was cloned with `CLONE_CLEARTID`, then `p'` must be configured to hold a `CLEARTID` futex for `c'`.  Or, CLEARTID must be fully emulated in replay.
* For each task `t` created with `CLONE_VM`, its clone `t'` must be created with the same stack pointer.  (This is thought to be required, but it might not be.)
* Fir each task `t' created with `CLONE_SETTLS`, its clone `t'` must be created with the same TLS pointer.  (This is thought to be required, but it might not be.)

## Cloning process memory

In theory, this sounds easy: recreate all the mmaps in the cloned process and copy the memory contents.  In practice there are some subtleties.  Here are the invariants; they're listed as if for a single process, but of course they apply to all address spaces being deepforked.

* Each memory mapping `m` must have a corresponding clone `m'` in the clone, covering the same region, with the same protection and flags, and with the same memory contents.  (Duh.)
* The memory backing `m'' must be a *semantic copy* of the region backing `m`.  For private mappings, this is natural and obvious.  For shared mappings, the resource backing `m` must itself be cloned, and then the cloned region mapped at `m'`.

An interesting question is whether it's better to re-`mmap` all regions in the deepfork and copy memory contents over, or attempt to do something clever with `fork`.  The fork approach is better in theory because the deepfork and its superparent will share as many memory pages as possible.  For that reason, it should also be much faster.  And with programs like Firefox that map a *lot* of memory, can have many child processes, and expect to share a significant portion of mapped memory among all the processes, the fork-to-share approach may be important.

## Cloning file resources

Luckily, in replay, the only way a tracee can directly access file resources is by replaying an mmap call that create a shared region.  This causes rr to create a file in its "emulated file system", and then that file is mapped on behalf of the tracee.  So wrt the invariants mentioned above, cloning file resources just reduces to the problem of cloning a tracee's EmuFS.  In other words, no additional invariants are added here.

## Strawman deepfork algorithm
<pre>
for each process(address space) p:
  [*] inject a remote fork() syscall into p's main thread
  run the fork() to create p'
  # we may also want to set up p' to deliver SIGCHLD to a non-default process.
  # It may be annoying to have the deepfork sending SIGCHLD to its source tree.

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
This algorithm attempts to use the `fork()` approach discussed above to clone memory contents.  Because it takes that approach, it *will not* preserve the task tree.  And in fact, the deepfork will be a child of its origin, which will look odd in `pstree`.  However, the attraction of CoW memory outweighs my desire for a clean `pstree`.

If we prefer to use the deepfork for throwaway uses like #604 and #605, then the process tree will remain looking somewhat saner (i.e. the deepfork will always die first, so the original task tree will always be intact).

This algorithm elides the mechanics of "restarting" emulated syscalls that are "interrupted" by the deepfork operation.

There are asterisks `*` in the algorithm above that represent unknowns.  They are

* `*`: at these places, we rely on injecting code into the cloned process's main thread.  This is absolutely required for the `fork()` call, because we don't have any other way of setting up the new process's main thread stack/heap.  (For the other injected syscalls, it doesn't particularly matter which thread runs the syscalls.)  However, I think it's possible in theory for a linux process's main thread to die before its child threads.  We need to experiment to see if this can happen in practice.  If so, it's possible to fix the algorithm above: we fork from an arbitrary thread in the original tree, do the setup from that new main thread, and then SYS_exit that main thread instead of restoring its registers.

## Implementation in rr

The key design questions are

* which process "owns" the deepfork: the original rr tracer process, or a new tracer process?
* depending on that answer, how is a separate view of the trace files created?  The state of the files objects in the TraceIfstream used by the original tracee tree must remain intact after the deepfork.

Only the original replayer process can force-inject `clone` calls.  And the child tasks created by those calls will be traced by that replayer process too.  It's quite annoying to transfer "ownership" of a tracee from an old tracer to a new one.  That all leans towards the side of abstracting trace-replay state and hosting the original trace and deepfork in the same replay process.

The first use case for deepfork checkpoint/restore is optimizing the gdb `restart` command.  Currently on restarts, rr does a little dance to exec a new replayer process over the old one (prepping debugger socket state in shmem, to be re-mapped intact after the exec).  This isn't too complicated but it's not simple.  (Even with deepfork, we'll still have to do from-the-beginning replay restarts when the user restarts at an earlier event than the original session.)

Considering this, it seems best to invest some time in encapsulating all global (replay?) state in a new replay session class.  In this way we can have multiple "live" replay sessions coexisting within the same replayer process.  Sessions being controlled by the same gdb client would explicitly share some kind of gdb client session state, instead of magically sharing it through shmem between restarts.  The replayer outer loop would then maintain a current session, passed along where needed, and replay restart would simply change that "current session" (either to a stashed deepfork, or to a fresh session).

Part of a replay session would be its TraceIfstream.  The deepfork operation would act at the level of a replay session, perhaps through a `clone()` interface.  Then to ensure multiple sessions don't interfere with each other's trace-file state, the session `clone()` would also clone the TraceIfstream.  A TraceIfstream `clone()` is relatively trivial to implement.

So the proposed changes are

* create a new class `ReplaySession`.  Migrate relevant global state into ReplaySession.  This includes at least the Task map.
* (most likely) create a new class `RecordSession` to provide the global state moved into ReplaySession, where needed.  They should subclass a common class that can provide common state like the task map, e.g.
* pass Replay(Record)Session around where needed
* before starting the replay of a session, `clone()` the session in the expectation that it will be restarted.  (Except for `--autopilot`.)/  Stash the deepfork somewhere.
* when a session is restarted to event `x`, see if the stashed session is for `x`.  If so, set it as the new current session.  Otherwise, discard it and set the current session to a fresh session.
* (continue replaying the session up to the target event/process.  For the deepfork fastpath, this will simply be a no-op.)

## Prior art (CRIU)

[CRIU](http://criu.org/Main_Page) seems to be the state-of-the-art in linux checkpoint/restore software.  CRIU aims for fully-generic checkpointing, which is massive overkill for rr's replay checkpointing.  Even so, it would have been nice to use reuse CRIU, but this doesn't appear possible.  However, if the rr deepfork implementation gets stuck, CRIU may be a source of inspiration on how to get un-stuck.