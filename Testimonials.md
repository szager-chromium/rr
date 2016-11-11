> I'm an rr developer, but my real job involves a lot of Gecko debugging. I find rr a great improvement over bare gdb and use it for almost all my Linux debugging tasks. It's an amazing tool!

  -- Robert O'Callahan

> rr has taken the application I work on from borderline-impossible to use inside a debugger to comically easy. I've got all my coworkers hooked on it. If you're using gdb, you should probably be using rr.

  -- @kellerb

> I was ecstatic when gdb gained reverse debugging abilities, but quickly had to realize it didn't work for me in practice because I was trying to debug something too big for it (Firefox). I've recently used rr to do some debugging on Firefox, and it didn't fail to deliver.
> While things like reverse-continue were relatively slow, having to wait for those was totally worth it if you look at the pain you'd have had to go through if you hadn't been able to reverse-continue in the first place.
> My only complaint about rr is that it didn't exist earlier!

  -- @glandium

> The idea of record-and-replay is not new; where rr is different is that it’s very low overhead and capable of handling complex programs like QEMU and Mozilla. It’s a usable production quality debug tool, not just a research project. I can’t recommend rr highly enough — I think it deserves to become a standard part of the Linux C/C++ developer’s toolkit, as valgrind has done before it.

  -- @pm215

> Even though we started using rr very recently, it has already cut down what would have been weeks of painful debugging for a couple of really nasty bugs.  Simply put, rr with reverse debugging is mindblowing.  We are excited to use it more and more in the future!

  -- @kavindaw-optumsoft

> `rr` has quickly become the number one tool I reach for when debugging complicated C++ code. `rr` only runs on Linux and I don't even use Linux as my day-to-day operating system! But `rr` provides such a great debugging experience, and gives me such a huge productivity boost, that I will reboot into Fedora *just to use `rr`* for all but the most trivial bugs. Take note of that, Linux advocates.

  -- @fitzgen, [Back to the Futu-rr-e: Determinisitc Debugging with `rr`](http://fitzgeraldnick.com/weblog/64/)

> I've been using `rr` for some time now, but in the last 3 days I hit a situation where it really saved my bacon. I had a mysterious problem in a data structure delivered to my gcc plugin, and I'm really not all that familiar with the inner workings of gcc. First of all, `rr` was enormously helpful in getting a debugger onto the right subprocess without messing around with dummy shell scripts to intercept and hack things in. Then `rr` made it possible for me to go back and forth through the cryptic internal gcc processing, tracing the origins of data embedded in structures one step at a time and keeping my head straight about chronology via heavy use of the `when-ticks` command. Not only did I track down my bug, but I learned a huge amount about the gcc internals I was looking at. I can't imagine how I could have tracked my bug down without `rr`.

  -- @sfink (Steve Fink)

> `rr` is just the most awesome debugging tool I've ever used. It's been super-useful to diagnose all kind of strange, nondeterministic, or racy difficult to reproduce bugs in both Servo (where race conditions are unfortunately common and usually really hard to track down) and Gecko. It's simply fantastic.

  -- @emilio (Emilio Cobos Álvarez)

***
_If you find rr useful, please add your testimonial here!_