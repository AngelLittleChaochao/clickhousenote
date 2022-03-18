# ClickHouse debug environment setup in VS Code
VS code is very friendly for developers. I use VS Code to debug remote process in another linux machine.

1. Build from source code.

You can follow the steps to build. I only built clickhouse-server and clickhouse-client.

2. Prerequisites

You can follow the link [here](https://code.visualstudio.com/docs/cpp/config-linux) to install VS Code and C++ extension.

3. Ensure GCC is installed

Also reference link [here](https://code.visualstudio.com/docs/cpp/config-linux), the same link with last section.

```
gcc -v
sudo apt-get update
sudo apt-get install build-essential gdb
```

4. Attach clickhouse process.

Please start your clickhouse-server first, then find the process id.

Open the clickhouse folder in VS Code, then add configuration file. Here is my configuration file. You need to update "processId" and "program" to your own.


```
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) Attach",
            "type": "cppdbg",
            "request": "attach",
            "program": "${workspaceFolder}/build/programs/clickhouse-server",
            "processId": "11344",
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        },
    ]
}
```
5. Errors

    **1) Authentication**

    When encountered error like here when debugging:

    ```
    ==== AUTHENTICATING FOR org.freedesktop.policykit.exec ===
    Authentication is needed to run `/usr/bin/gdb' as the super user
    Authenticating as: hanbing
    Password: [1] + Stopped (tty output)       /usr/bin/pkexec "/usr/bin/gdb" --interpreter=mi --tty=${DbgTerm} 0<"/tmp/Microsoft-MIEngine-In-zmh829r2.jk4" 1>"/tmp/Microsoft-MIEngine-Out-jh3h8jnd.mmk"
    You have stopped jobs.
    ```

    Please follow the [link](https://github.com/Microsoft/MIEngine/wiki/Troubleshoot-attaching-to-processes-using-GDB) to update.  I used the first method works.

    Run the following command as super user: echo 0| sudo tee /proc/sys/kernel/yama/ptrace_scope


    **2) SourceRequest**

    When encountered the following error, it means it cannot find the right source file.

    ```
    Could not load source './build/./src/InterpretersexecuteQuery.cpp': 'SourceRequest' not supported..
    ```

    Using the suggestion from this [issue](https://github.com/microsoft/vscode-cpptools/issues/3831), I added 'sourceFileMap' field. For [sourceFileMap](https://code.visualstudio.com/docs/cpp/launch-json-reference)

    ```
    This allows mapping of the compile-time paths for source to local source locations. It is an object of key/value pairs and will resolve the first string-matched path. (example: "sourceFileMap": { "/mnt/c": "c:\\" } will map any path returned by the debugger that begins with /mnt/c and convert it to c:\\. You can have multiple mappings in the object but they will be handled in the order provided.)
    ```

    So here is my final launch.json file and it works.

    ```
    {
        // Use IntelliSense to learn about possible attributes.
        // Hover to view descriptions of existing attributes.
        // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
        "version": "0.2.0",
        "configurations": [
            {
                "name": "(gdb) Attach",
                "type": "cppdbg",
                "request": "attach",
                "program": "${workspaceFolder}/build/programs/clickhouse-server",
                "processId": "11344",
                "sourceFileMap": {"./build/./src": "/projects/ClickHouse/build/../src"},
                "MIMode": "gdb",
                "setupCommands": [
                    {
                        "description": "Enable pretty-printing for gdb",
                        "text": "-enable-pretty-printing",
                        "ignoreFailures": true
                    }
                ]
            },
        ]
    }
    ```
