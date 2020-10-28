## Step 1: Do the tests pass?

Build with `cmake -DCMAKE_BUILD_TYPE=RELEASE -Dstaticlibs=TRUE -Dstrip=TRUE`! You may need to build and install the latest stable version of `capnproto` to get static libraries you can link into rr.

Bare minimum requirements

* rr test suite passes on x86 and x86-64 machines with updated kernels
* rr can record recent Firefox builds on those machines

Recommended testing

* multiple runs of rr test suite (tests can fail intermittently)
* LibreOffice and Firefox start up and shut down in virtual machines running various Linux releases (e.g. Ubuntu 14.04 32 and 64, Ubuntu 15.10 64). Scripts in https://github.com/rocallahan/rr-vm-testing help automate this.
* the following Firefox mochitest suites pass when recorded and replayed by rr
    * dom/tests/mochitest/ajax/jquery (short, simple, CPU intensive)
    * dom/browser-element (heavily multiprocess)
    * layout/generic
    * layout/base
    * dom/media/test (exercises audio/video playback)
    * dom/media/tests/mochitest (exercises WebRTC)
    * dom/workers/test (heavily multithreaded)
    * dom/indexedDB/test
* the following Firefox reftest-like suites pass when recorded and replayed by rr
    * crashtest
    * crashtest-ipc
    * reftest (**Warning**: this can generate a large amount of trace data)
    * reftest-ipc (**Warning**: this can generate a large amount of trace data)
* recording overhead has not regressed (**TODO**)
    * see [this blog post](http://robert.ocallahan.org/2014/03/introducing-rr.html)

Running the test schedule above on both debug and release builds of Firefox is recommended.

For major releases, multiple runs of the test schedule above are recommended.

We of course want to automate this process as soon as possible/practical.

## Step 2: Building and shipping

- [ ] Bump version: `cd $rr && ./scripts/tag-release.sh MAJOR MINOR PATCH`.
- [ ] On the build machine (running an old Linux distro, e.g. Ubuntu 16.06), `make && make check && make package`
- [ ] Push changes to Github: `git push origin; git push --tags origin`
- [ ] [Create release and upload packages](https://github.com/mozilla/rr/releases)
- [ ] Update gh-pages: `./scripts/update-gh-pages.sh && git push origin`
- [ ] Update [News wiki page](https://github.com/mozilla/rr/wiki/News)
- [ ] Post to rr-dev mailing list.