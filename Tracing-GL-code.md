`rr` does not support tracing DRM-related ioctl. rr tries to block access to `/dev/dri/*`, which is normally enough to avoid DRI and trigger fallback to llvmpipe.

If this does not happen automatically for you, you can check if you have llvmpipe installed and forcing it manually:

```
$ LIBGL_ALWAYS_SOFTWARE=1 glxinfo | grep OpenGL
OpenGL vendor string: VMware, Inc.
OpenGL renderer string: Gallium 0.4 on llvmpipe (LLVM 3.7, 256 bits)
OpenGL core profile version string: 3.3 (Core Profile) Mesa 11.0.4
OpenGL core profile shading language version string: 3.30
OpenGL core profile context flags: (none)
OpenGL core profile profile mask: core profile
OpenGL core profile extensions:
OpenGL version string: 3.0 Mesa 11.0.4
OpenGL shading language version string: 1.30
OpenGL context flags: (none)
OpenGL extensions:
OpenGL ES profile version string: OpenGL ES 3.0 Mesa 11.0.4
OpenGL ES profile shading language version string: OpenGL ES GLSL ES 3.00
OpenGL ES profile extensions:
$ LIBGL_ALWAYS_SOFTWARE=1 glxinfo32 | grep OpenGL
OpenGL vendor string: VMware, Inc.
OpenGL renderer string: Gallium 0.4 on llvmpipe (LLVM 3.7, 256 bits)
OpenGL core profile version string: 3.3 (Core Profile) Mesa 11.0.4
OpenGL core profile shading language version string: 3.30
OpenGL core profile context flags: (none)
OpenGL core profile profile mask: core profile
OpenGL core profile extensions:
OpenGL version string: 3.0 Mesa 11.0.4
OpenGL shading language version string: 1.30
OpenGL context flags: (none)
OpenGL extensions:
OpenGL ES profile version string: OpenGL ES 3.0 Mesa 11.0.4
OpenGL ES profile shading language version string: OpenGL ES GLSL ES 3.00
OpenGL ES profile extensions:
```

```
LIBGL_ALWAYS_SOFTWARE=1 rr ./your-app
```

There is more discussion about underlying issues and possible solutions in [#1578](https://github.com/mozilla/rr/issues/1578) and [#1579](https://github.com/mozilla/rr/pull/1579).