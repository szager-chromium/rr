Please make sure you go through this list before submitting a patch.  The rules aren't hard and fast, but mostly adhering to them will make for quicker mergings.

&#9633; Does your PR add support for a new kernel API?  For example, supporting a new syscall.  If so, your patch should include at least one new test for the API.  This is usually pretty easy.  See `$rr/src/test` for examples.

&#9633; Did you run the rr test suite (including your new tests, if any), and pass all the tests?  `make -C $objdir check`.  Unfortunately, rr doesn't have automated infrastructure that can run the tests yet, so developers have to run them locally.

**Note**: the `alarm` test fails sporadically for a known (NYI) reason, < 1/10 runs, with an error that looks like
<pre>
TODO: support multiple pending signals; received SIGSYS (code: %d) at $ip:%p while trying to deliver SIGSTKFLT (code: %d)
</pre>
The tests that interact with gdb will also fail if the rr process is unluckily assigned a pid < 1000.  There will be a message that says something to the effect of "can't bind to port X", where X < 1000.

&#9633; Does your PR apply cleanly on top of upstream/master HEAD?  It's dangerous to have someone else sort out your merge conflicts, so just don't do it.  Best of all is to have a PR *rebased* on top of upstream/master HEAD, so that the merge is simply a fast-forward.

&#9633; If your PR includes multiple changesets, do they all (i) build cleanly in sequence; (ii) pass all tests in sequence?  This is important for bisecting over commit history.

&#9633; If your PR is a very large-scale change (for example, a rewrite in Rust to use the visitor pattern), did you discuss the proposed changes in an issue or the mailing list?  It's hard to review large patches that just fall in ones lap.  It's much easier to discuss the important changes at a high level and then approach the patch knowing what's important and what's not.

&#9633; If your PR is large or includes many changesets, would it have been possible to break the changes into a series of smaller PRs?  For example, it's hard to review a big patch that, say, fixes whitespace errors in a file along with a one-line, important, bug fix.  It's much easier to review one PR that fixes whitespace (which can just be skimmed), and then review another PR that makes the one-line bug fix (which would be scrutinized more).  This approach is also better for the patch author in that it usually allows the work to land faster.

&#9633; Whitespace is an eternal annoyance.  For historical reasons, rr uses hard tabs with an 8-char offset (like the linux kernel).  It's just easier for everyone if you make sure your editor uses that mode.