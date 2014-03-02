"chroniclerr" is my code name for the functionality of [chronicle-recorder](http://code.google.com/p/chronicle-recorder) embedded into rr.

## Backend overview

At a high level, chroniclerr will reuse the current replayer code to advance execution to a given execution point (target-process + target-event).  And/or, it can inherit a superforked checkpoint tree.  At that point, "chronicle mode" is enabled.  chroniclerr will stop having tracees run code directly, and will instead emulate execution of all tracee tasks and processes in the chroniclerr process.

What happens at the mode switch?  First chroniclerr inspects the page mappings of the tracee tree and ships them off to the indexer to create the DB's initial view of the application tree (maybe along with thread information?).  Then it schedules the next task (`read_next_trace()`) and checks to see if the task has a TCB for the emulated CPU yet.  If not, rr builds one from the task's current register state.

At this point chroniclerr is ready to run the emulated task.  The dispatch is modeled after valgrind's: chroniclerr examines the task's (emulated) `$ip` and sees if there's already a translation for that BB. If not, chroniclerr fetches the block's machine code from the task (the OS task is still hanging around, remember?) and uses libvex to translate it.  Here, the chronicle instrumentation is inserted, in as similar a manner as is practical to original valgrind-chronicle-recorder.

An implementation detail here is that chroniclerr will almost always inherit a tracee tree in which all tracees are *not* at the start of a basic block.  In all probability, they'll all be in the middle of basic blocks.  That means chroniclerr may instrument "partial blocks" one time at mode switch.  This should be perfectly fine though.

Chroniclerr will also frequently inherit tracees in the middle of syscalls.  As long as the syscalls are interruptible, that's also perfectly fine.  See discussion below about atomic syscalls.

Now chroniclerr has a scheduled task ready to run an instrumented block of code.  It hands off execution to the instrumented code, a la valgrind dispatch.  The JIT'd code returns on various conditions such as encountering an untranslated block, reaching syscall entry, or exceeding its retired-insns or retired-branch allotment.  The latter two cases result in a (possibly) new task being scheduled, and the steps above repeating.

Chroniclerr will inherit all processes' page maps from the initial replay stage.  These maps will be used to build "page tables" for an emulated MMU.  The first time that an emulated load/store touches a page that's not "mapped" in the emulator, and only exists in the tracee, then chroniclerr will `read(mem-fd)` the page into a local "physical page" and update the "page table entry" appropriately, taking shared mappings into account  Note: directly-mapped non-copy files, like /usr/lib/libc.so, can be directly mmap'd into the emulator as well, to avoid wasting memory.  From there on, only the local copy of the page is modified.  If another task later maps the same "physical page" (as indicated by the process maps), then rr will update that process's "PTE" with a reference to the same "physical page" as the first process's.

## Implementation issues

* <s>Need to create an "address space" data structure a la current `struct sighandlers`.  Address space == process.  Each task has a (possibly shared) address space.</s>

* <s>Maintain task tree that's available to all tools.  All code already in rep_sched.c, just centralize.</s>

* <s>Maintain map info per address space.  Known-good optimization (parsing /proc/maps shows up on profiles) and makes many things much simpler.  Must handle shmem and be deterministic.  Assign backings with a deterministic ID; something analogous to physical page / underlying resource.</s>

* Tracees can't be interrupted in the middle of atomic syscalls like mmap et al.  At most one task can be in the middle of atomic event sequence, so fast-forwarding ahead to a safe point should be a simple operation.

* Syscalls like fork/clone/exec have somewhat nontrivial kernel effects.  How to support?
    * emulate.  Chroniclerr is already going to maintain accurate map; indexer already consumes mmap events; might as well take the next step to full OS and emulate everything.  Syscalls like exec can be supported by recording pseudo-mmap events that represent kernel effects.
    * if there any syscalls we can't emulate, then we can always fall back on running the commands in the "real" tracees in parallel.

* <s>Need to log event numbers at stdio writes to correlate with execution.</s>

* How to represent mremap() in chronicle trace events?

* <s>`CLONE_CHILD_CLEARTID`: rr isn't capable of recording this accurately, so the emulator will have to make the same guess that the recorder does.  Obviously, if the recorder gets it wrong, so will the emulator (but that would be a deeper general bug).</s>

* Can chronicle represent pages of memory that were last written "before the beginning of time"?  (i.e. event at which mode switch occurs.)  rr can send the complete memory map, but it doesn't seem useful to send the entire memory contents.

* Based on prior art from valgrind, we should plan ahead on implementing an emulated TLB as a perf meliorization.  Not complicated.  TLB flush on task switching into different address space and munmap/mremap/mprotect operations.

* Need to integrate libvex with rr.  <s>Split out of valgrind repository?  Phone our friends.</s>  libvex has its own standalone repo.

## Big issue: "canonical" memory

Tracee tree is around as long as we want it, and it can be trivially kept up to date with the emulated state.  (In fact, that's a good idea for consistency checking!)  But where should the canonical values live?

If we ever want to cache memory in the emulator process (we do!), then we need to know when page X and page Y point at the same resource.  If we're going to be doing that, we might as well keep canonical resources in emulator.

exec() is a bit annoying because don't know what the kernel is going to stick in there, a priori.  So we could run the emulator until an exec() is hit, then have the "real tree" advance up to the exec().  That's a good place for a consistency check too.  Could also punt this since unlikely anyone will chroniclerr across an exec.  But recording the effects of exec as a series of pseudo-mmaps should make this problem go away.

So basically we want to roll our own "page tables".  This overlaps with the process-map-tracker referred to above.

We'll want to lazily allocate the pages.  If we're lucky, only a small portion of the mapped pages will be touched during chroniclerring.

Chroniclerr will want to directly mmap shared read-only pages to avoid wasting memory.  It will probably want to implement CoW `fork` semantics, for the same reason.  (This is pretty easy when rr has its own pram and MMU.)

If chroniclerr ever reaches the point where it's mapped all of a tracee process's rw pages, then it could discard the process.  This is rather delicate so would need to be motivated strongly.

## Parallelism

It would be really really nice to write chroniclerr as a normal rr tool.  Doing so, however, limits the amount of tracee memory that can be emulated (~1/2 system RAM, less than 2GB on x86).  Maybe this is a problem we can punt until full x86-64 support.  If not, a hybrid solution is rr running on a 64-bit kernel as a 32-bit process, but chroniclerr being forked/exec'd as a 64-bit process.  That's nasty engineering-wise, because it introduces IPC and cross-arch concerns over sharing rr library code.  (The touch points of rr into the trace are not minimal.)

We *absolutely must* have the indexer/compressor running in parallel to the emulator.  This part is extremely flexible though (simple threaded producer/consumer maybe best).

## C++

<s>All this new code is going to want C++.  Are we OK paying the small-ish cost of making the existing code compile under g++?  I think I am.  (Note: that does *not* entail a full rewrite; cf. the mozilla-central js/src evolution towards C++.)</s>

## Follow-up work

If done generally enough, this instrumentation framework can be used in the same way as valgrind, but *retroactively* on a recorded trace.  That is, a user could repeatedly, quickly, record a buggy workload until an intermittent crash is triggered, then hand off to a memcheck-like tool for analysis.  (Built similarly to the chroniclerr tool described above.)  On the plus side, the instrumentation tools wouldn't share (real) address space with the instrument-ees, so they could be written more simply in a higher-level language.  The downside is that redirecting memory accesses through an emulated MMU adds overhead.  It remains to be seen whether these will be good tradeoffs.

The pseudo-mmap machinery may allow us to drop the requirement of disabling ASLR, which would be a big usability win.