## Step 1: Do the tests pass?

Bare minimum requirements

* rr test suite passes on x86 and x86-64 machines with updated kernels
* rr can record recent Firefox builds on those machines

Recommended testing

* multiple runs of rr test suite (tests can fail intermittently)
* LibreOffice and Firefox start up and shut down in virtual machines running various Linux releases (e.g. Ubuntu 14.04 32 and 64, Ubuntu 15.10 64).
The following `xvnc-runner.sh` script is helpful:
````
#!/bin/bash

CMD=$1
EXPECT=$2

rm -f ~/tmp/xvnc ~/tmp/xvnc-client ~/tmp/xvnc-wininfo ~/tmp/xvnc-client-replay

Xvnc :9 > ~/tmp/xvnc 2>&1 &
until grep -q "Listening" ~/tmp/xvnc; do
  sleep 1
done
DISPLAY=:9 ~/rr/obj/bin/rr $CMD > ~/tmp/xvnc-client 2>&1 &
DISPLAY=:9 xwininfo -tree -root > ~/tmp/xvnc-wininfo 2>&1
until grep -q "$EXPECT" ~/tmp/xvnc-wininfo; do
  sleep 1
  DISPLAY=:9 xwininfo -tree -root > ~/tmp/xvnc-wininfo 2>&1
done
kill %1
wait %2
~/rr/obj/bin/rr replay -a > ~/tmp/xvnc-client-replay 2>&1
if [[ $? != 0 ]]; then
  echo FAILED: replay failed
  exit 1
fi
diff ~/tmp/xvnc-client ~/tmp/xvnc-client-replay
if [[ $? != 0 ]]; then
  echo FAILED: replay differs
  exit 1
fi
echo PASSED: $CMD
exit 0
````
Run it like this:
````
(rm -rf ~/tmp/firefox-profile ; mkdir ~/tmp/firefox-profile ; ~/rr/vm/xvnc-runner.sh "firefox --profile $HOME/tmp/firefox-profile $HOME/rr/vm/test.html" "rr Test Page")
(rm -rf ~/.config/libreoffice ; ~/rr/vm/xvnc-runner.sh "libreoffice $HOME/rr/vm/rr-test-doc.odt" "rr-test-doc.odt")
````
* the following Firefox mochitest suites pass when recorded and replayed by rr
    * dom/tests/mochitest/ajax/jquery (short, simple, CPU intensive)
    * dom/browser-element (heavily multiprocess)
    * layout/generic
    * layout/base
    * content/media/test (exercises audio/video playback)
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

- [ ] Bump version: `cd $rr && ./src/script/tag-release.sh MAJOR MINOR PATCH`.
- [ ] `git fetch` and `merge` changes on the x86-32 build machine. `git checkout` the tag that was created.
- [ ] On both machines, `make && make check && make package`
- [ ] Copy `obj/dist/*` files from x86-32 build machine to `obj/dist` on x86-64 machine.
- [ ] Push changes to Github: `git push origin; git push --tags origin`
- [ ] [Create release and upload packages](https://github.com/mozilla/rr/releases)
- [ ] Update gh-pages: `./src/script/update-gh-pages.sh && git push origin`
- [ ] Update [News wiki page](https://github.com/mozilla/rr/wiki/News)
- [ ] Post to rr-dev mailing list.