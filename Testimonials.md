https://twitter.com/chandlerc1024/status/879962014860193792

> I'm an rr developer, but my real job involves a lot of Gecko debugging. I find rr a great improvement over bare gdb and use it for almost all my Linux debugging tasks. It's an amazing tool!

  -- Robert O'Callahan

> I'm the author of OpenResty. I recently successfully used rr to quickly track down and fix a very obscure JIT stack overflow bug inside LuaJIT, with the help of LuaJIT's author. The patch is already merged to mainline: https://goo.gl/D5i47I The issue could only be randomly reproduced with a very large Lua script (1.8MB) in stress testing. `rr record` quickly recorded down a single run that hit this issue in stress testing on `x86_64`. The data breakpoints and reverse execution features in `rr replay` make debugging this nasty bug even enjoyable. Our [advanced gdb tools in Python](https://github.com/openresty/openresty-gdb-utils#readme) can also work flawlessly in `rr replay`. I really wish I had rr when I was tracking ~10 very deep LuaJIT bugs with LuaJIT's author a few years ago. At that time I could only analyze core dumps. Alas. rr is such an amazing tool!

  -- Yichun Zhang (@agentzh)

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

  -- @fitzgen, [Back to the Futu-rr-e: Deterministic Debugging with `rr`](http://fitzgeraldnick.com/weblog/64/)

> I've been using `rr` for some time now, but in the last 3 days I hit a situation where it really saved my bacon. I had a mysterious problem in a data structure delivered to my gcc plugin, and I'm really not all that familiar with the inner workings of gcc. First of all, `rr` was enormously helpful in getting a debugger onto the right subprocess without messing around with dummy shell scripts to intercept and hack things in. Then `rr` made it possible for me to go back and forth through the cryptic internal gcc processing, tracing the origins of data embedded in structures one step at a time and keeping my head straight about chronology via heavy use of the `when-ticks` command. Not only did I track down my bug, but I learned a huge amount about the gcc internals I was looking at. I can't imagine how I could have tracked my bug down without `rr`.

  -- @sfink (Steve Fink)

> `rr` is just the most awesome debugging tool I've ever used. It's been super-useful to diagnose all kind of strange, nondeterministic, or racy difficult to reproduce bugs in both Servo (where race conditions are unfortunately common and usually really hard to track down) and Gecko. It's simply fantastic.

  -- @emilio (Emilio Cobos Álvarez)

> `rr` is a fantastically useful debugging tool. It has made root cause analysis of `cargo-fuzz`-found panics quicker when fuzzing Rust. However, it is at its best when debugging a large codebase you can't possible know thoroughly, such as Gecko. For example, it helps in a situation where a problem doesn't occur on the first attempt, because there's an empty cache far away elsewhere in the codebase and the problem being debugged requires the cache to already have the relevant entry. As another example, it helped me greatly when debugging invariant violations arising from nested event loops by allowing a narrowing back-and-forth execution. Each time you continue or reverse-continue over the problem, you can move breakpoints closer to the problem until your breakpoints are close enough that you can see the problem between them. Having objects reside in the same memory addresses throughout the debugging session also helps greatly and makes watchpoints more useful than they would otherwise be.

  -- @hsivonen (Henri Sivonen)

***
_If you find rr useful, please add your testimonial here!_