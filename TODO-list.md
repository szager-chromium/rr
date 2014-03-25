## Good First Bugs
We maintain a set of issues that should be easy to resolve, but aren't necessarily critically important.  These make "good first bugs" to work on because you learn how the code and build/test/pull-request processes work, without simultaneously having to worry about a nontrivial patch.

https://github.com/mozilla/rr/issues?labels=goodfirstbug&page=1&state=open

## Projects
Projects fall into a space of easy vs. hard, small vs. big, important vs. not-so-important, and well-defined vs. open-ended.  For example, all of the "good first bugs" above are considered easy/small/not-so-important/well-defined.  On the other hand, a pure-research topic for an academic paper would might be hard/important/open-ended.  The projects are listed along with their classification.

Project | Difficulty | Size | Importance | Defined-ness
--------|------------|------|------------|-------------
Remove dependencies on libdisasm and libpfm (issues #974 / #954) | Pretty easy | Small | Pretty important for new devs | Well defined |
When btrfs is being used, use the equivalent of "cp --reflink" to cheaply copy all input files into the trace directory (issue #602) | Moderate | Medium | Important[1] | Well defined[1]
Figure out if Haswell has a deterministic performance counter we can use (issue #973) | Pretty hard | Small | Critical | Well defined
Compress trace data (issue #700) | Moderate | Medium | Important | Less well defined
Port to x86-64 (issue #606) | Moderate[2] | Large | Important for new users | Moderately well defined
Implement fork-based checkpointing of an entire replay session (issue #603) | Hard | Large | Important | Somewhat open ended

0. The actual work is not too difficult and is well-defined, but btrfs isn't the default fs for any shipping linux distro as of this writing.  The quality of btrfs isn't known either.  So this this code may not be used for a long time.
0. On the surface, porting to x86-64 is easy.  However, there may be some difficulties that arise from x64 "virtual syscalls" that make this project very hard.