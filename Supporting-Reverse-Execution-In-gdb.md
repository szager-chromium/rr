Let's assume we want to restrict ourselves for now to approaches based on checkpointing and execute-forward. Other approaches based on instruction emulation and massive logging (e.g. chroniclerr) are possible but a lot more work, and less robust to instruction set evolution etc.

gdbserver defines `bc` (backwards continue) and `bs` (backwards step) commands. That's all rr would have to implement.

## Implementing `bc`

During normal forward execution, rr periodically spawns checkpoints using existing `Session::clone` API.

Consider a `bc` request. Call the current registers+rbc the "origin". If there are no breakpoints set, just reset to the beginning of the trace. But generally there will be breakpoints set. Restore the previous checkpoint (call it N) and execute forward to the origin with breakpoints enabled, recording the execution state (registers+rbc) each time a breakpoint is hit. When we reach the origin, if any breakpoints were hit, resume from checkpoint N again and execute forward again but this time stopping at the registers+rbc where the last breakpoint hit occurred.

If no breakpoints were hit between checkpoint N and the origin, resume at checkpoint N-1 and execute forward to checkpoint N. Continue searching earlier checkpoint intervals until at least one breakpoint is hit. If no breakpoints are ever hit, reset to the beginning of the trace.

## Implementing 'bs`

Choose some skid constant K, restore to the previous checkpoint (or if necessary the one before that) and play forward until rbc is origin-rbc - K. Of course we'll overshoot that value but hopefully not the actual instruction we want to reach. Single-step forward until the origin point is reached, logging the register+rbc state at each instruction. Then restore the checkpoint again and execute forward until we hit the last instruction state before the origin.

## gdb issues

Currently whenever we reset state to a checkpoint gdb treats it as a full restart. This ensures shared library data is reloaded etc. However, gdb will not expect `bs`/`bc` to trigger a full restart. In fact, it's very likely that gdb will become hopelessly confused when reverse execution crosses the loading or unloading of a shared library.

I think at least initially we should just stop reverse execution when a shared library event occurs. gdb sets internal breakpoints in ld.so to detect these events, and we can use heuristics to detect these breakpoints. I hope debugging with reverse execution does not usually need to cross shared library events.

## Improving performance

The `bs` approach outlines above is likely to be slow, especially as gdb probably uses a lot of `bs` commands to implement features like `rnext`. One way to optimize a sequence of `bs` commands would be to preserve the list of per-instruction register+rbc states after the first `bs` command. Then the second command can use the same list and skip the list building step. Even better, we could make restoration of the full program state lazy. So after a `bs` we leave the current execution point wherever it was and report to gdb the register state found in our list. If another `bs` occurs immediately we just adjust the position in our list and we've eliminated a state restoration step.

Given a breakpoint (e.g. "$eip == NNN" or "watch of K bytes at address NNN") and an execution interval (e.g. delineated by start+end register+rbc states), how many times the breakpoint fires in that interval and the register+rbc states at which it fires are pure functions of the tracee recording. Program execution (both forward and backward) with breakpoints enabled implicitly gathers this information. We can therefore save it alongside the trace data as it is gathered, and use it to optimize both forward and reverse execution when the enabled breakpoints are a subset of the breakpoints we know about in the interval we're executing through.

## Risks

Lots of fine-grained stopping at specific destination states are more likely to expose situations where register+rbc states do not uniquely identify execution points. Whether this is a problem in practice remains to be seen. Also, if we implement `bs` as above, with a log of register+rbc states leading up to the origin, we can detect these problems. E.g. instead of stopping when we hit the register+rbc state corresponding to the origin the first time, carry on until rbc has incremented beyond the origin rbc. If the log does not contain any other occurrence of the origin state, we know we've found the right one. If it does, we can at least error out with a useful error message, and possibly do something to recover.