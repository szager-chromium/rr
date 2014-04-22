This page describes the implementation plan for compressing rr traces as they're written to disk.  Trace compression intends to reduce disk-space usage, which would allow recording longer executions.

A back-of-the-envelope experiment, compressing a 1.1GB trace directory with `tar -cjvf`, reduced the size to 26MB.  That's a 42x compression ratio with a fully general compression algorithm (bz2).  An application-specific algorithm that's performant may or may not be able to do better.

The difficulty in compressing traces isn't the actual compression/decompression (which will be done by libraries initially), but rather refactoring rr to have multithreaded trace file read/write.  Once that's done, (de)compression is a relatively additional step at the read/write endpoint.

## Step 1: Transfer data ownership to/from trace write/read functions

The recorder produces data to be written to trace.  The trace-writing code then takes that data and writes it to file.  (And resp. for trace-reader / replayer.)  To make threading sane, we'll want an explicit transfer of ownership at the boundary between recorder/trace-writer.  C++11's `unique_ptr` is just the mechanism for this.

The model wrt the recorder is: it passes a `unique_ptr` to the appropriate `write()` helper, and the write helper assumes ownership.  The recorder doesn't have to worry about multithreading.  (And resp. read / replayer.)

(Note: the mythical "thread-safe object", with atomic refcounting etc., isn't the right way to do this.)

The work here is a straightforward modification of the TraceOfstream/Ifstream interfaces to accept `unique_ptr<T>&` inparams/outparams respetively.  And then of course updating the API consumers.  (We need to migrate away from the `operator <<()` shorthand.)

## Step 2: Read/write trace data on background threads

This will most likely make recording faster, but a speedup isn't required.  (A regression would be very surprising and should be investigated.) 

The next step is to add background writer threads to TraceOfstream (resp. reader / TraceIfstream).  The simplest model is producer/consumer, where the recorder produces data for the TraceOfstream consumer (resp. TraceIfstream / replayer).  The two threads synchronize over shared `queue<unique_ptr>`.  In the TraceOfstream `write()` helpers, the client's `unique_ptr` is given to the queue within a lock.  The write thread(s) then take that `unique_ptr` within the same lock.  (Condvar synchronization around the queue is as in vanilla producer/consumer.)

It's up for discussion whether there should be a single background writer thread or multiple.  It's probably easiest to use the following setup

* `data` / `data_header`: share their own background thread.  `data` is the most important file to write in the background.  It's not worth the hassle write `data_header` on a different thread than `data`.
* `events`: have its own background thread.  Almost certainly worth compressing.  It's awkward to design a shared queue that allows easily sharing a thread with `data`, and threads are cheap enough what the heck.
* `args_env`: only written once so don't bother with a thread.  (This doesn't have a persistent `fstream` anyway.)
* `mmaps`: not clear what's best yet.  Probably not worth compressing.  Easier to start without a mmaps thread.

These changes should be local to trace.{c,h} only.

## Step 3: Compress data

The project is to replace `write(bytes, len)` calls with calls to a compressed-write() helper from a compression library, and similarly for `read()`.  Which compression algorithm / library to use initially is an open question.

It's probably only worth compressing the files that are worth writing on background threads: `data` / `data_header` and `events`.