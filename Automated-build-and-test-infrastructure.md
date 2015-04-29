rr uses [travis-ci](https://travis-ci.org/mozilla/rr) for automated builds.  The lifetime of a build/test run is as follows

0. A committer pushes commit _X_ to mozilla/rr
0. _X_ triggers a github service hook that notifies travis-ci
0. The travis-ci infrastructure schedules a build/test task for _X_.  The pending job shows up [here](https://travis-ci.org/mozilla/rr/builds).  The output shows up "live" once the task commences, so you can watch the results come in if you're extremely bored.
0. When the task completes, the results are posted [here](https://travis-ci.org/mozilla/rr)

Commits that break the build or tests will notify the [rr-builds](https://mail.mozilla.org/listinfo/rr-builds) mailing list, and the committer who broke the build.  Anyone can subscribe to the list.

The travis-ci configuration file is stored in `$rr/.travis.yml`.  It first invokes the script `src/script/setup_travis.sh` to configure the travis slave for rr, and then invokes the `make` and `make check` targets.  (It's really that simple!)

There is now [a Jenkins instance](http://45.55.219.138:8080/job/rr/) running builds and tests for rr commits. It's similar to the Travis setup described above, except it simply runs:

    ./configure && make -j`getconf _NPROCESSORS_ONLN` && make fastcheck

Jenkins is not currently configured to send email notifications.