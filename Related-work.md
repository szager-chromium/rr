This page is in a somewhat disorganized state, please bear with us.

== Userspace recording ==

No target recompilation or VM hypervisor required.

[http://code.google.com/p/chronomancer/ Chronomancer]/[http://code.google.com/p/chronicle-recorder/ Chronicle]

[http://sourceware.org/gdb/wiki/ReverseDebug gdb reverse debugging] ([http://sourceware.org/gdb/wiki/ProcessRecord "process recorder"])
<blockquote>
Process record and replay works by logging the execution of each machine instruction in the child process (the program being debugged), together with each corresponding change in machine state (the values of memory and registers).
</blockquote>
* unclear how modification of user memory during syscalls is recorded
* unclear how process-shared memory is dealt with
* very very high overhead
* good approach for efficient replaying reverse-step et al.

== Hypervisor recording ==

[http://www.cs.uiuc.edu/homes/kingst/Research_files/dunlap02.pdf ReVirt]

[http://blogs.vmware.com/workstation/2008/04/enhanced-execut.html VMWare Record & Replay]

== (Not yet categorized) ==

[http://www.cs.columbia.edu/~nieh/pubs/sigmetrics2010_scribe.pdf‎ Scribe]

Thus spake roc:
<blockquote>
There are a few major differences between Scribe and rr:
* Scribe doesn't serialize all threads. Instead they do a bunch of work to make sure all threads can run simultaneously. This reduces overhead in some places and adds overhead in others.
* They say their approach doesn't require "changing, relinking or recompiling the kernel" but their approach has to track internal kernel state like inodes and VFS path traversal, and it's not really clear how they do that. They also say "Scribe records by intercepting all interactions of processes with their environment, capturing all nondeterminism in events that are stored in log queues inside the kernel" so my guess is they're using a kernel module. That's a pretty big negative in my view.
* Scribe doesn't use performance counters to record asynchronous events. Instead they defer signal delivery until the next time the process enters the kernel. If the process doesn't enter the kernel for a long time, they basically take a snapshot of the entire state, force the process into the kernel and restart recording --- extremely heavyweight. For some bugs, it's essential to allow async signal delivery at any program point, so I don't like Scribe's approach there.
</blockquote>

[http://cseweb.ucsd.edu/~calder/papers/thesis-satish.pdf iDNA]

[http://home.gna.org/jockey/ Jockey]

[http://dl.acm.org/citation.cfm?id=1772954.1772958 Pinplay]

[http://web.eecs.umich.edu/~nsatish/papers/ASPLOS-10-Respec.pdf Respec]

[http://www.mnis.fr/fr/services/virtualisation/pdf/ekatrinaitskova.pdf Echo]

[http://web.eecs.umich.edu/virtual/papers/king03.pdf OS Support]

[http://css.csail.mit.edu/6.858/2012/readings/backtracking.pdf BackTracker]

[http://static.usenix.org/event/usenix05/tech/general/king/king.pdf Time-Traveling Virtual Machines]

[https://web.eecs.umich.edu/~pmchen/papers/lucchetti05.pdf ExtraVirt]

[http://web.eecs.umich.edu/~pmchen/papers/king06.pdf SubVirt]

[https://cs.uwaterloo.ca/~brecht/courses/702/Possible-Readings/debugging/execution-replay-for-mp-vms-vee-2008.pdf SMP-ReVirt]

[http://notrump.eecs.umich.edu/papers/asplos23-nightingale.pdf Speck]

[http://www-personal.umich.edu/~jouyang/veeraraghavan11.pdf DoublePlay]

See [http://read.seas.harvard.edu/cs261/2011/doubleplay.html this page].

[http://www-mount.ece.umn.edu/~jjyi/MoBS/2007/program/01C-Xu.pdf ReTrace]

[http://www.jeffhuang.tk/academic/clap.pdf CLAP]

[http://iacoma.cs.uiuc.edu/iacoma-papers/asplos09.pdf Capo]

[http://iacoma.cs.uiuc.edu/iacoma-papers/isca13_1.pdf QuickRec / Capo3]

[https://www.usenix.org/legacy/event/usenix04/tech/general/full_papers/srinivasan/srinivasan_html/paper.html FlashBack]

[http://dl.acm.org/citation.cfm?id=1772958 PinPlay]