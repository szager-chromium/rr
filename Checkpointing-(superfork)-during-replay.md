This page describes the implementation plan for adding the capability to create a semantic copy of an entire tracee tree (including copying files etc. as necessary), an operation we're calling "superfork".  This operation is the mechanism needed to create checkpoints of the tracee tree during replay.  Replay checkpoints are in turn needed to optimize the gdb `restart` workflow, and allow running arbitrary code in tracee processes.

Note, in the discussion that follows, "superfork" will be used both as a verb and a noun.  The verb describes the process of cloning a tracee tree, and the noun refers to that tracee tree cloned by the superfork verb.  The usage should be clear from context.  Apologies if this causes any confusion.

## Ensuring tracees are at a stable execution point

By "stable", I mean that rr can restore the superfork to the state of the original tracee tree.  Ensuring this in replay is relatively easy compared to recording.  The only thing rr needs to do to ensure the tracees are stable (AFAIK) is preventing any of them from being in an "executed" syscall (as opposed to emulated).  If a tracee were in an executed syscall at superfork time, the tracee would have to be interrupted out of the syscall in such a way the syscall could be restarted in both the superfork and the original tracee tree.  This may be difficult, and could cause divergence as well if not done carefully.

Luckily, all syscalls currently executed during replay are atomic.  So the stability problem reduces to ensuring that a superfork isn't initiated in the middle of an atomic syscall, which is relatively easy.

It's not yet known what will happen when an emulated syscall is "interrupted" by a superfork.  However, rr can always manually "finish" emulated syscalls that are "interrupted" by superfork, by (i) tracking the interrupted-ness state; (ii) issuing the PTRACE commands based on the state, or not; (iii) manually updating the register file to "finish" the interrupted call.