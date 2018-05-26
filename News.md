**May 26, 2018**: [rr 5.2.0 released](https://github.com/mozilla/rr/releases/tag/5.2.0). This is a minor maintenance release with some improvements to trace portability: https://robert.ocallahan.org/2018/04/cpuid-features-xsave-and-rr-trace.html

**December 14, 2017**: [rr 5.1.0 released](https://github.com/mozilla/rr/releases/tag/5.1.0). This is a minor maintenance release.

**September 7, 2017**: [rr 5.0.0 released](https://github.com/mozilla/rr/releases/tag/5.0.0). This is a pretty big release that introduces trace portability and a stable trace format; see release notes.

**February 4, 2017**: [rr 4.5.0 released](https://github.com/mozilla/rr/releases/tag/4.5.0). This is a maintenance release with lots of small improvements; see release notes.

**October 1, 2016**: [rr 4.4.0 released](https://github.com/mozilla/rr/releases/tag/4.4.0). This is a maintenance release with lots of small improvements; see release notes.

**July 1, 2016**: [rr 4.3.0 released](https://github.com/mozilla/rr/releases/tag/4.3.0). This is a maintenance release with lots of small improvements; see release notes.

**June 2, 2016**: Sean Heelan published [a nice blog post](https://sean.heelan.io/2016/05/31/tracking-down-heap-overflows-with-rr/) about tracking down exploitable heap overflows with rr.

**March 17, 2016**: [rr 4.2.0 released](https://github.com/mozilla/rr/releases/tag/4.2.0). The main feature of this release is chaos mode. There are many other improvements; see release notes.

**February 10, 2016**: [rr chaos mode landed](http://robert.ocallahan.org/).

**February 3, 2016**: [rr 4.1.0 released](https://github.com/mozilla/rr/releases/tag/4.1.0). The main feature of this release is major replay performance improvements: replaying buffered syscalls avoids trapping to rr. Replay is generally faster than recording now; before, it was generally slower. This release also implements support for gdb's `find` command. Various bugs have been fixed. Intel Skylake is supported. A kernel bug that caused machines to lock up during rr tests has been worked around.

**December 5, 2015**: CppCast did [a podcast interview](http://cppcast.com/2015/12/robert-ocallahan) about rr with Robert O'Callahan.

**December 2, 2015**: [Replay performance](http://robert.ocallahan.org/2015/11/even-more-rr-replay-performance.html) has been considerably improved on master.

**November 23, 2015**: [rr 4.0.3 released](https://github.com/mozilla/rr/releases/tag/4.0.3). This was another minor release fixing bugs and adding the `real-tid` command.

**November 12, 2015**: Jeff Muizelaar [blogged](http://muizelaar.blogspot.co.nz/2015/11/debugging-reftests-with-rr.html) about his [rr-dataflow](https://github.com/jrmuizel/rr-dataflow/) tool that extends rr with an `origin` command that provides cool data-flow tracking capabilities. 

**November 11, 2015**: rr 4.0.2 released. This was another minor release fixing bugs and supporting a few more syscalls.

**November 3, 2015**: Another nice blog post about rr:
* [Back to the Futu-rr-e](http://fitzgeraldnick.com/weblog/64/)

**November 2, 2015**: rr 4.0.1 released. This makes some minor fixes and adds the `rr replay -d <debugger'` option.

**October 28, 2015**: Some nice blog posts about rr:
* [Rreverrse Debugging](http://huonw.github.io/blog/2015/10/rreverse-debugging/)
* [Tricks for debugging QEMU - rr](http://www.linaro.org/blog/core-dump/tricks-for-debugging-qemu-rr/)

**October 23, 2015**: rr 4.0.0 released. This is the first stable release with reverse execution enabled. [More here](http://robert.ocallahan.org/2015/10/rr-40-released-with-reverse-execution.html).

**June 6, 2015**: A user-visible behavior change [documented here](http://robert.ocallahan.org/2015/06/small-change-to-rr-behavior.html) makes rr send fake SIGKILL events to gdb just before the debuggee process exits, giving users a chance to reverse-execute from that point.

**April 29, 2015**: rr 3.2.0 released. This fixes a serious regression in rr 3.1.0.

**April 23, 2015**: rr 3.1.0 released. See [announcement](http://robert.ocallahan.org/2015/04/rr-31-released.html). This release contains the nascent reverse-execution support, which has not yet been declared stable.

**December 12, 2014**: rr 3.0.0 released. See [announcement](http://robert.ocallahan.org/2014/12/rr-30-released-with-x86-64-support.html). The major feature is x86-64 support.

**September 9, 2014**: rr 2.0.0 released. See [announcement](http://robert.ocallahan.org/2014/09/rr-20-released.html). The major feature is the ability to run debuggee functions from gdb during replay.

**June 24, 2014**: rr 1.5.0 released.

**June 16, 2014**: rr 1.4.0 released.

**April 15, 2014**: rr 1.2.1 released.

**April 15, 2014**: rr 1.2.0 released.

**April 2, 2014**: rr 1.1.0 released.

**March 26, 2014**: Blog post: [Introducing rr](http://robert.ocallahan.org/2014/03/introducing-rr.html)

**March 26, 2014**: rr 1.0 released.