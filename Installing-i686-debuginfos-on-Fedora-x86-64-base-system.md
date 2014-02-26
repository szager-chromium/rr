Fedora (as of release 20) doesn't officially support installing debuginfos for secondary architectures.  Until then you can work around it by following the steps below.

**WARNING**: this is a giant hack.  **FIXME** find a better way.

If you're debugging and gdb prints a message like
<pre>
Missing separate debuginfos, use: debuginfo-install glibc-2.18-12.fc20.i686
</pre>
then obviously it can't find debuginfo for the given package(s).  Let's use "glibc-2.18-12.fc20.i686" as an example.  Here's how to get debuginfo installed for it

* Search online for "glibc-2.18-12.fc20.i686 debuginfo rpm" and download the link that comes up.
* <pre>
cd
mkdir debuginfos32
cd debuginfos32
mv ~/Downloads/\*-debuginfo-\*.i686.rpm ./
sudo rpm -ivh glibc-debuginfo-2.18-12.fc20.i686.rpm
</pre>
* Note that you'll have to update it manually when the underlying glibc package updates.