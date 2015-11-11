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