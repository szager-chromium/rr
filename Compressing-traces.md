This page describes the implementation plan for compressing rr traces as they're written to disk.  Trace compression intends to reduce disk-space usage, which would allow recording longer executions.

A back-of-the-envelope experiment, compressing a 1.1GB trace directory with `tar -cjvf`, reduced the size to 26MB.  That's a 42x compression ratio with a fully general compression algorithm (bz2).  An application-specific algorithm that's performant may or may not be able to do better.

The difficulty in compressing traces isn't the actual compression/decompression (which will be done by libraries initially), but rather refactoring rr to have multithreaded trace file read/write.  Once that's done, (de)compression is a relatively additional step at the read/write endpoint.

## Step 1: Transfer data ownership to/from trace write/read functions