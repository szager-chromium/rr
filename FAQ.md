* How does one pronounce 'rr' ?

  As the letter 'r', twice.

* Does rr work in docker on an arbitrary host system?

  rr will work inside docker if it works outside docker, though you may need to run your container with `--privileged` (and you should be aware of the security implications!)

* Will rr record automatically start recording forked processes?

  Yes.

* How do you record / replay a multi process session?

  rr records process trees, so if all of the processes you care about are part of the same process tree, you're already set.  Alternatively, you can make two separate recordings for two processes.  During replay, you can launch multiple replays simultaneously, selecting the processes you care about with `-p`.  The `when` command inside the gdb rr launches can be used to orient the different replays in time.

* Are there any plans to support BSD / MacOS?

  [No](https://joneschrisg.wordpress.com/2015/01/29/rr-on-os-x-can-it-be-ported/).

* Can rr be used to (effectively) debug java/jvm programs? webdriver tests?

  rr can record and replay programs written in higher level languages, but because it presents you with an enhanced gdb interface during replay, you may find it challenging to debug languages that are difficult to debug from gdb.