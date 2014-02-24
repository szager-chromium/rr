It's almost essential for rr dev to have your distro's patched libc source handy, and sometimes also useful to have your own libc build.  Here's approximately how to do it on f20

First, follow [these instructions](http://fedoraproject.org/wiki/Building_a_custom_kernel) for installing the right build tools.  Then,
<pre>
# glibc build needs this too.
sudo yum install libstdc++-static
# Download glibc source.
cd tmp/
yumdownloader --source glibc
rpm -Uvh glibc-*.src.rpm
# Create a build dir for glibc.
cd $home/rpmbuild/SPECS/
rpmbuild -bp --target=$(uname -m) glibc.spec
# Configure and build it.
cd ../BUILD/glibc-*
mkdir obj/
cd obj/
LD_LIBRARY_PATH="" ../configure --prefix="$PWD"
make -s
</pre>
It takes a while to complete the build.

Run programs with your custom glibc with
<pre>
LIBC_OBJDIR="$HOME/rpmbuild/BUILD/glibc-2.18/obj"
LD_PRELOAD="$LIBC_OBJDIR/libc.so:$LIBC_OBJDIR/nptl/libpthread.so" \
    program --args
</pre>
(Obviously, it's convenient to do this using a helper script.)