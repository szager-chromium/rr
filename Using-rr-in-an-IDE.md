## Using rr in an IDE

rr implements the standard gdb server interface, and also can present the gdb command-line interface. This means it can be used in integrated development environments (IDEs) which support GUI debugging based on gdb/gdbserver.

Known to work:
* Visual Studio Code
* CLion
* QtCreator
* Eclipse
* emacs GUD/gdb-mi
* [gdbgui](https://gdbgui.com)

### Setting up [CLion](https://www.jetbrains.com/clion/)

Use at CLion version 2017.1 or greater, with rr version 5.0 or greater.

The buttons for the reverse-step gdb commands can be added to CLion by installing [UndoDB's plugin](https://plugins.jetbrains.com/clion/plugin/8620-undo-reversible-debugging-integration) for CLion.

1. After you install rr, run the following to save a copy of the gdb initialization script for rr to your home directory. You should also re-run this after installing a new rr release.

    ```bash
    $ rr gdbinit > ~/.rr_gdbinit
    ```
2. Add the following to the `.gdbinit` file in your home directory. Create a new one if it does not exist.

    ```gdb
    # get around CLion/QtCreator not supporting target extended-remote
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
4. Enter CLion.
5. Open `Run` -> `Edit Configurations`
6. Click the green `+` sign, and add a `GDB Remote Debug` configuration
7. Under `target remote args`, enter `:50505`, or another port of your choice
8. For the symbol file, point CLion to the executable that you are running (it is located in the build directory). You can also use the hard link in the rr trace directory, if it is there:
    ```bash
    ~/.local/share/rr/latest-trace/mmap_hardlink_3_executable_name
    ```
9. Invoke rr:
    ```bash
    $ rr replay -s 50505 -k
    ```
10. Start debugging in CLion by clicking the debug button. Make sure that the GDB remote configuration is selected.

### [QtCreator](http://doc.qt.io/qtcreator/)

Set up `.gdbinit` as for CLion, following steps 1-4 above. Then:

1. Invoke rr:
    ```bash
    $ rr replay -s 50505 -k
    ```
2. Enter QtCreator.
3. Open `Debug` -> `Start Debugging` -> `Attach to Running Debug Server...`
4. Set the field `Server Port` to `50505` and `Override sever address` to `localhost`.
5. In the field named `Local Executable`, select the executable that you are running (it is located in the build directory). You can also use the hard link in the rr trace directory, if it is there:
    ```bash
    ~/.local/share/rr/latest-trace/mmap_hardlink_3_executable_name
    ```
6. Start debugging by clicking the "Ok" button.

### Setting up [Eclipse](https://eclipse.org/)

1. Install the Eclipse CDT as usual.
2. Install rr master or >= 4.6.0.
3. Create a script somewhere like so, calling it e.g. `rrgdb`:
    ```bash
    #!/bin/bash
    exec rr replay -- "$@"
    ```
4. Record something using rr.
5. Create a debugging configuration specifying the debugger as `rrgdb`. Don't enable Eclipse's reverse debugging, it doesn't work with rr.
6. Launch the debugging configuration. It should work. You may need to manually set a breakpoint at `main` and then continue to it.
7. To reverse-execute, open the Debugger Console tab in the Console view and enter manual commands such as `reverse-continue` (`rc`), `reverse-step` (`rs`), etc

### Setting up emacs

See this [blog post](http://notes.secretsauce.net/notes/2017/02/24_interfacing-rr-to-gdb-in-gnu-emacs.html):
> you simply take the suggested command (`gdb --fullname` or `gdb -i=mi`), replace `gdb` with `rr replay`

### Setting up [Visual Studio Code](https://code.visualstudio.com/)

1. Install Visual Studio Code
2. Install the C/C++ extension (https://code.visualstudio.com/docs/languages/cpp)
3. Open the directory where your source code reside.
4. Create a debugging configuration from Debug view (Ctrl+Shift+D) and add the following configuration (this was for debugging firefox, adjust as per your requirements)
>         {
>             "name": "rr",
>             "type": "cppdbg",
>             "request": "launch",
>             "program": "${workspaceRoot}/../obj-ff-dbg/dist/bin/firefox",
>             "args": [],
>             "miDebuggerServerAddress": "localhost:50505",
>             "stopAtEntry": false,
>             "cwd": "${workspaceRoot}/../obj-ff-dbg/dist/bin",
>             "environment": [],
>             "externalConsole": true,
>             "linux": {
>               "MIMode": "gdb",
>               "setupCommands": [
>                 {
>                     "description": "Setup to resolve symbols",
>                     "text": "set sysroot /",
>                     "ignoreFailures": false
>                 }
>               ]
>             },
>             "osx": {
>               "MIMode": "gdb"
>             },
>             "windows": {
>               "MIMode": "gdb"
>             }
>         }
5. Install rr master or >= 4.6.0.
6. Record something using rr.
7. Start rr with ```rr replay -s 50505 -k```
8. Launch the rr debugging configuration in VS code
9. To reverse-execute, open the Debugger Console tab (Shift+Ctrl+Y) and enter manual commands such as `-exec reverse-continue` (`-exec rc`), `-exec reverse-step` (`-exec rs`), etc
10. When rewinding the code, the display isn't refreshed. Step-in in the code once will fix that.

You can set breakpoints like you would with a normal debugging session. To set hardware watchers, in the debug console enter the rr command preceded by ```-exec``` (like ```-exec watch -l mVar```)

### Setting up [gdbgui](https://gdbgui.com/)
1. Install gdbgui
2. Install rr master or >= 5.1.0 (not tested with earlier versions, but may work)
3. Record something using rr
4. Replay: `gdbgui --rr`. Optionally specify the directory of the recording: `gdbgui DIRECTORY --rr`

[View demo](https://www.youtube.com/watch?v=WijcPxOang0) on YouTube.
