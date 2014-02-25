□ Bump version number in CMakeList.txt.  `git commit`.

□ `git tag` the new version.  `git push --tags`.

□ `git fetch` and `merge` changes on the build machine.

□ `git checkout` the tag that was created.

□ `mkdir obj && cd obj && cmake ../rr && make && make check && make package`

□ Copy `obj/dist/` files to machine hosting rr/gh-pages branch.

□ On gh-pages branch, `git pull`.

□ For now, update gh-pages/index.html with the new version number.  #753 will obsolete that.

□ For now, copy the built packages into gh-pages/release/.  #753 will obsolete that.

□ `git commit` the new files and index.html change.

□ Post to rr-dev mailing list.