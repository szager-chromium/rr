## Using rr in an IDE

Since rr has implemented a standard gdb server interface, this means it can be used in integrated development environments (IDEs) which support GUI debugging based on gdb. The instructions below are written for CLion, but they should apply to other IDEs with gdb-based debugging too. Here is a list of some popular ones:

  * [CLion](https://www.jetbrains.com/clion/)  
For use with rr, use at CLion version 2017.1 or greater, with rr version (strictly) greater than 4.5.0 (until 4.6.0 is released, you will have to [build](Building-And-Installing) rr from the latest source)

* [Qt Creator](https://www.qt.io/ide)
* [Emacs](https://www.gnu.org/software/emacs/)
* [KDevelop](https://www.kdevelop.org)
* [Code::Blocks](http://www.codeblocks.org/)

### Setting up CLion to use rr

1. After you install rr, run the following to save a copy of the gdb initialization script for rr to your home directory. You should also re-run this after installing a new rr release.

    ```bash
    $ rr gdbinit > ~/.rr_gdbinit
    ```
2. Add the following to the `.gdbinit` file in your home directory. Create a new one if it does not exist.

    ```gdb
    # get around CLion not supporting target extended-remote
    define target remote
    target extended-remote $arg0
    end

    define target hook-extended-remote
    source ~/.rr_gdbinit
    end

    # optional: prevent gdb asking for confirmation
    # when invoking the run command in gdb
    set confirm off

    set remotetimeout 100000

    ```
3. Record an rr trace from command line [as usual](Usage). (You could perhaps add a Run configuration in CLion if you are doing this often.)
4. In CLion:

  1. Open `Run` -> `Edit Configurations`
  2. Click the green `+` sign, and add a `GDB Remote Debug` configuration
  3. Under `target remote args`, enter `:50505`, or another port of your choice
  4. For the symbol file, point CLion to the executable that you are running (it is located in the build directory). You can also use the hard link in the rr trace directory, if it is there:

        ```bash
      ~/.local/share/rr/latest-trace/mmap_hardlink_3_executable_name
        ```
  5. Invoke rr:

        ```bash
      $ rr replay -s 50505 -k
        ```
  6. Start debugging in CLion by clicking the debug button. Make sure that the GDB remote configuration is selected.

The buttons for the reverse-step gdb commands can be added to CLion by installing [UndoDB's plugin](https://plugins.jetbrains.com/clion/plugin/8620-undo-reversible-debugging-integration) for CLion.