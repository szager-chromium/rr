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

The next step is to add background writer threads to TraceOfstream (resp. reader / TraceIfstream).  The simplest model is producer/consumer, where the recorder produces data for the TraceOfstream consumer (resp. TraceIfstream / replayer).  The two threads synchronize over shared `queue<unique_ptr>`.  In the TraceOfstream `write()` helpers, the client's `unique_ptr` is given to the queue within a lock on the recorder thread.  The write thread(s) then take that `unique_ptr` within the same lock on the background thread.  (Condvar synchronization around the queue is as in vanilla producer/consumer.)

It's up for discussion whether there should be a single background writer thread or multiple.  It's probably easiest to use the following setup

* `data` / `data_header`: share their own background thread.  `data` is the most important file to write in the background.  It's not worth the hassle write `data_header` on a different thread than `data`.
* `events`: have its own background thread.  Almost certainly worth compressing.  It's awkward to design a shared queue that allows easily sharing a thread with the `data` writer, and threads are cheap enough that don't bother trying.  Possible perf advantage too.
* `args_env`: only written once so don't bother with a thread.  (This doesn't have a persistent `fstream` in TraceOfstream anyway.)
* `mmaps`: not clear what's best yet.  Probably not worth compressing.  Easier to start without a mmaps thread.

These changes should be local to trace.{c,h} only.

## Step 3: Compress data

The project is to replace `write(bytes, len)` calls with calls to a compressed-write() helper from a compression library, and similarly for `read()`.  Which compression algorithm / library to use initially is an open question.

It's probably only worth compressing the files that are worth writing on background threads per above: `data` / `data_header` and `events`.

## Compression algorithms

I did trial compression of a short Firefox run. The results are for compressing the `data` file. The uncompressed size of `data` was 1079MB. By comparison, the next biggest file was `events` at 151MB.

| Algorithm     | Elapsed Time (s) | Compressed File Size |
| ------------- | ----------------:| --------------------:|
| `bzip2` (1.0.6) | 144 | 48 |
| `xz` (5.1.2alpha) | 157 | 16 |
| `xz -T 0` (5.1.2alpha) | 36 | 16 |
| `gzip` (5.1.2alpha) | 10 | 45 |
| `lz4` (r119) | 1 | 103 |
| `lzo` (2.08) | 2 | 285 |

`xz` uses LZMA. `xz -T 0` uses all available cores (4 cores, 8 hyperthreads in my case). All other algorithms used only a single core. All algorithms used default compression options.

Based on these results I think `gzip` is a reasonable choice. It's efficient and compresses well. I don't see any need to compress further at the risk of recording overhead.

## CLOEXEC

When making these changes, move away from using C++ streams so we can set CLOEXEC on trace file descriptors. This will avoid the current situation where trace file descriptors leak into the fd space of every tracee.