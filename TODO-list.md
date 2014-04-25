## Good First Bugs
We maintain a set of issues that should be easy to resolve, but aren't necessarily critically important.  These make "good first bugs" to work on because you learn how the code and build/test/pull-request processes work, without simultaneously having to worry about a nontrivial patch.

https://github.com/mozilla/rr/issues?labels=goodfirstbug&page=1&state=open

## Projects
Projects fall into a space of easy vs. hard, small vs. big, important vs. not-so-important, and well-defined vs. open-ended.  For example, all of the "good first bugs" above are considered easy/small/not-so-important/well-defined.  On the other hand, a pure-research topic for an academic paper might be hard/large/important/open-ended.  The projects are listed along with their classification.

Project | Difficulty | Size | Importance | Defined-ness
--------|------------|------|------------|-------------
Remove dependencies on libpfm (issue #974) | Pretty easy | Small | Pretty important for new devs | Well defined |
When btrfs is being used, use the equivalent of "cp --reflink" to cheaply copy all input files into the trace directory (#602) | Moderate | Medium | Important[1] | Well defined[1]
Figure out problems with some Haswell systems (#1054) | Pretty hard | Small | Critical | Well defined
Compress trace data (#700) | Moderate | Medium | Important | Less well defined
Port to x86-64 (#606) | Moderate[2] | Large | Important for new users | Moderately well defined
Finish implementing gdb machine interface (#488 and followups) | Moderate | Medium | Nice for users | Pretty well defined
Profile rr with real-time-sampling profiler (like Zoom) | Easy | Small | Nice to have | Well defined
Somehow show recorded graphics during replay | Hard | Large | Nice for users | Open ended
Have non-perf-counter fallback mode where signals are delivered at preemption points | Easy | Small | Unclear | Well defined
Write profiler that analyzes rr traces | Hard | Large | Nice to have | Open ended
Get qemu/UML running under rr | Moderate | Moderate | Nice to have | Well defined
Get WINE running under rr (#158) | Hard | Moderate | Important | Well defined
Determine if a tool like rr can be written for Windows NT | Hard | Large | Nice to have for now | Open ended
Have rr inject random system faults (#778) | Moderate | Medium | Important | Open ended
Enable rr to record itself recording something | Hard | Large | Nice to have | Well defined
Get rr working with valgrind (or vice versa) #16 | Moderate | Small | Important | Well defined
Get rr to record Rust programs | Hard | Large | Will be important | Well defined
Support mutable replay (replay of a program with a patch) | Hard | Medium | Important | Open ended
Debug a high-level language (e.g. JS) during rr replay | Hard | Medium | Important | Open ended

0. The actual work is not too difficult and is well-defined, but btrfs isn't the default fs for any shipping linux distro as of this writing.  The quality of btrfs isn't known either.  So this this code may not be used for a long time.
0. On the surface, porting to x86-64 is easy.  However, there may be some difficulties that arise from x64 "virtual syscalls" that make this project very hard.