---
description: Troubleshoot problems and find the cause of bugs in an NGINX or F5 NGINX Plus
  deployment, with the debugging binary, debug logging, and core dumps.
title: Debugging NGINX
toc: true
weight: 300
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "debug log, --with-debug, nginx-debug, debug_connection, error_log memory, core dump, worker_rlimit_core, working_directory, backtrace, gdb"
f5-summary: >
  This page explains how to get a debug-enabled NGINX binary, write the debugging log, collect core dumps and backtraces, and see the configuration that NGINX loaded.
  These are the details that a developer or the NGINX community needs to find the cause of a crash or of unexpected behavior.
  It applies to NGINX Open Source and NGINX Plus.
f5-audience: operator
---

## Overview

When NGINX Open Source or F5 NGINX Plus crashes or behaves unexpectedly, the error log at its usual levels often doesn't say enough. NGINX has these debugging tools:

- **The debugging log.** It records each step of request processing, such as the parsed request line and headers, location selection, upstream connections, and buffer use. You can write it to a file, to a memory buffer, or only for selected client addresses.
- **Core dumps.** If an NGINX process crashes, the operating system can save its memory to a core file. A debugger then shows where the crash happened.
- **The loaded configuration.** NGINX can show the configuration files it actually loaded, which can differ from the files on disk.

## Before you begin

- You need `sudo` access on the NGINX host.
- For core dumps, backtraces, and the GDB scripts on this page, install the GNU Debugger (GDB), for example with `sudo apt install gdb` or `sudo dnf install gdb`. GDB also needs an NGINX binary with [debugging symbols](#compile_symbols).
- To read the loaded configuration through the Control API, install `jq`, a command-line JSON processor.
- Debug logging writes a lot of data. On a busy server, prefer the [memory buffer](#error_log_memory) or [selected client addresses](#error_log_ip) over a log file at the `debug` level for all traffic.
- Core files and debug logs can contain sensitive data, such as passwords, cookies, and private keys. Store them securely, and remove them when you're done.

## Get an NGINX binary with debug support {#enable}

The debugging log works only in an NGINX binary built with debug support. A binary without it has no code for debug messages. It accepts the `debug` level of `error_log`, but writes no `[debug]` lines.

To check whether a binary has debug support, run:

```shell
nginx -V 2>&1 | grep -o -- '--with-debug'
```

If the output is `--with-debug`, the binary has debug support. If it's empty, switch to the debug binary (NGINX Plus) or rebuild NGINX (NGINX Open Source).

### Switch to the NGINX Plus debug binary

Starting from [Release 8]({{< ref "/nginx/releases.md#r8" >}}), NGINX Plus ships the `nginx-debug` binary together with the standard binary. To use it, stop NGINX Plus and start the debug binary:

```shell
sudo service nginx stop && sudo service nginx-debug start
```

Then [enable the debugging log](#error_log) in the configuration.

While the debug binary runs, use `nginx-debug` instead of `nginx` in the commands on this page. For example, run `sudo nginx-debug -t`, `sudo nginx-debug -s reload`, and `nginx-debug -V`. The standard binary has no debug support, so `nginx -t` fails for a configuration with `error_log memory:`.

### Build NGINX Open Source with debug support {#enable_oss}

To add debug support to NGINX Open Source, rebuild it with the `--with-debug` option of the `configure` script:

1. Download and unpack the NGINX source files, and go to the directory with the source files. See [Download the sources]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-open-source.md#sources_download" >}}).

2. Get the `configure` arguments of the current binary:

    ```shell
    nginx -V 2>&1 | grep 'configure arguments'
    ```

3. Run `configure` with `--with-debug` added to those arguments:

    ```shell
    ./configure --with-debug <OTHER_CONFIGURE_ARGUMENTS>
    ```

    Replace `<OTHER_CONFIGURE_ARGUMENTS>` with the arguments from the previous step.

4. Build and install NGINX:

    ```shell
    make
    sudo make install
    ```

5. Restart NGINX.

### Debugging symbols {#compile_symbols}

Debugging symbols map machine code back to functions, variables, source files, and line numbers. A debugger needs them to show a readable [backtrace](#backtrace).

When you build NGINX from source with GCC or Clang, `configure` adds the `-g` option, so the `objs/nginx` binary has debugging symbols. It does this only if the `CFLAGS` environment variable isn't set. Some packages remove the symbols from the installed binary. If GDB reports "No symbol table info available", the binary has no symbols. Rebuild NGINX from source, and use the binary that the build creates.

NGINX is built with the `-O` optimization level by default. Optimization can make a backtrace harder to read, because some variables show as `<optimized out>`. To turn it off, add `-O0` to the `--with-cc-opt` option. The compiler gets this option after the default `-O`, so `-O0` takes effect.

`configure` uses only the last `--with-cc-opt` option. If the arguments from `nginx -V` already have one, add `-O0 -g` at the end of its value instead of adding a second option. For example, change `--with-cc-opt='-g -O2'` to `--with-cc-opt='-g -O2 -O0 -g'`. If there's none, add it:

```shell
./configure --with-debug --with-cc-opt='-O0 -g' <OTHER_CONFIGURE_ARGUMENTS>
```

## Enable the debugging log {#error_log}

To turn on the debugging log, set the `debug` level in the [`error_log`](https://nginx.org/en/docs/ngx_core_module.html#error_log) directive. You can write the debugging log to a file, to a memory buffer, to `stderr`, or to syslog.

Set the debugging log in the `main` context, outside any block. An `error_log` directive in the `http`, `server`, or `location` context replaces the one it inherits. A debug log set there misses the messages of the other contexts.

### Write the debugging log to a file {#error_log_file}

A debugging log in a file can slow NGINX down under high load, and the file can grow quickly. To limit the effect, use the [memory buffer](#error_log_memory) or [selected client addresses](#error_log_ip) instead.

1. Check that the binary has [debug support](#enable).

2. Open the NGINX configuration file, usually `/etc/nginx/nginx.conf`:

    ```shell
    sudo vi /etc/nginx/nginx.conf
    ```

3. In the `main` context, set the level of the `error_log` directive to `debug`. If you want, change the path to the log file:

    ```nginx
    error_log /var/log/nginx/error.log debug;
    ```

4. Save the file, then check the configuration and reload NGINX:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

### Write the debugging log to memory {#error_log_memory}

NGINX can write the debugging log to a cyclic buffer in memory. Writing to memory has much less impact on performance than writing to a file.

The buffer works like this:

- Each worker process has its own buffer. To read the log of a request, read the buffer of the worker process that handled it. To find that worker, add the `$pid` variable to the access log format. Or set [`worker_processes 1`](https://nginx.org/en/docs/ngx_core_module.html#worker_processes) while you debug, so that one worker handles all requests.
- When the buffer is full, new messages overwrite the oldest ones. The newest messages can be in the middle of the buffer, followed by older ones.
- The buffer starts with a `MEMLOG` line, which has the buffer size and the place in the configuration where it's defined.

To write the debugging log to memory:

1. Check that the binary has [debug support](#enable). Without it, `nginx -t` fails with "nginx was built without debug support".

2. In the `main` context, set the `error_log` directive with the `memory:` prefix and a buffer size:

    ```nginx
    error_log memory:32m debug;
    ```

3. Check the configuration and reload NGINX:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

#### Extract the debugging log from memory

Read the buffer with a GDB script. The script needs an NGINX binary with [debugging symbols](#compile_symbols). While GDB is attached, the worker process stops and doesn't serve requests. Run the script quickly.

1. Find the PIDs of the worker processes:

    ```shell
    ps -o pid,cmd -C nginx,nginx-debug
    ```

    Worker processes show as `nginx: worker process`.

2. Attach GDB to a worker process:

    ```shell
    sudo gdb -p <WORKER_PID>
    ```

    Replace `<WORKER_PID>` with the PID of a worker process from the previous step.

3. Paste this script into GDB and press **Enter**. It saves the buffer to the `debug_log.txt` file in the current directory:

    ```none
    set $log = ngx_cycle->log
    while $log->writer != ngx_log_memory_writer
        set $log = $log->next
    end
    set $buf = (ngx_log_memory_buf_t *) $log->wdata
    dump binary memory debug_log.txt $buf->start $buf->end
    ```

4. Quit GDB with **Ctrl+D**, and confirm that you want to detach. The worker process continues.

5. Open the file. It has the size of the buffer, and unused space is filled with spaces:

    ```shell
    less debug_log.txt
    ```

### Write the debugging log for selected clients {#error_log_ip}

To write the debugging log only for connections from some clients, list their addresses with the [`debug_connection`](https://nginx.org/en/docs/ngx_core_module.html#debug_connection) directive in the [`events`](https://nginx.org/en/docs/ngx_core_module.html#events) block. NGINX logs connections from these addresses at the `debug` level, and other connections at the level that `error_log` sets. This works well on a production server, because only a few connections write debug messages.

The directive takes an IPv4 or IPv6 address, a CIDR range, a hostname, or `unix:` for connections over UNIX-domain sockets. NGINX resolves a hostname once, when it reads the configuration. You can repeat the directive:

```nginx
error_log /var/log/nginx/error.log;

events {
    debug_connection 192.168.1.1;
    debug_connection 192.168.10.0/24;
}
```

Without debug support in the binary, NGINX ignores `debug_connection` and logs a warning: `"debug_connection" is ignored, you need to rebuild nginx using --with-debug option to enable it`.

### Set the debugging log for one virtual server {#error_log_server}

An `error_log` directive in a `server` block replaces the one from the `main` context for that server. Use it to turn the debugging log on or off for one virtual server.

Until NGINX knows which virtual server a request is for, it logs to the log of the default server for the `listen` address. This matters for a name-based virtual server that isn't the default one. The start of each of its connections and requests, up to the `Host` header, goes to the default server's log.

To write the debugging log for one virtual server only, set a separate log file with the `debug` level in its `server` block:

```nginx
error_log /var/log/nginx/error.log;

http {
    server {
        listen 80;
        server_name debug.example.com;
        error_log /var/log/nginx/debug.example.com.log debug;
    }
}
```

To turn the debugging log off for one virtual server, set `error_log` in its `server` block without a level. The default level is `error`:

```nginx
error_log /var/log/nginx/error.log debug;

http {
    server {
        listen 80;
        server_name quiet.example.com;
        error_log /var/log/nginx/error.log;
    }
}
```

## Enable core dumps {#coredump}

A core file shows where an NGINX process crashed. To get one, the operating system must allow core dumps. NGINX also needs a directory to write them to, and a large enough size limit.

Core files can be as large as the memory of the process. Make sure the file system has enough free space.

### Prepare the operating system {#coredump_os}

1. Create a directory for core files, for example `/tmp/cores`, and let every user write to it. The worker processes don't run as `root`:

    ```shell
    sudo mkdir /tmp/cores
    sudo chown root:root /tmp/cores
    sudo chmod 1777 /tmp/cores
    ```

2. Check where Linux writes core files:

    ```shell
    cat /proc/sys/kernel/core_pattern
    ```

    - A pattern that starts with `|`, such as `|/usr/share/apport/apport ...` or `|/usr/lib/systemd/systemd-coredump ...`, passes core files to that program. Get the core file with the program's tools, for example `coredumpctl` for `systemd-coredump`, or set a file pattern as in the next step.
    - A pattern that doesn't start with `/`, such as `core`, is a file name in the current directory of the process. NGINX sets that directory with the [`working_directory`](#coredump_nginx) directive.

3. Optional: write core files to `/tmp/cores` with the program name and PID in the file name. This setting lasts until the next reboot:

    ```shell
    echo '/tmp/cores/core.%e.%p' | sudo tee /proc/sys/kernel/core_pattern
    ```

On Linux, worker processes allow core dumps themselves after they switch from `root` to the [`user`](https://nginx.org/en/docs/ngx_core_module.html#user) set in the configuration, so `fs.suid_dumpable` doesn't need to change. On FreeBSD, allow core dumps of processes that changed their user, and set where core files go:

```shell
sudo sysctl kern.sugid_coredump=1
sudo sysctl kern.corefile=/tmp/cores/%N.core.%P
```

### Configure NGINX {#coredump_nginx}

1. Open the NGINX configuration file, usually `/etc/nginx/nginx.conf`:

    ```shell
    sudo vi /etc/nginx/nginx.conf
    ```

2. In the `main` context, set the directory for core files with the [`working_directory`](https://nginx.org/en/docs/ngx_core_module.html#working_directory) directive. Worker processes change to this directory when they start. They must be able to write to it:

    ```nginx
    working_directory /tmp/cores/;
    ```

    If the directory doesn't exist, `nginx -t` fails and NGINX doesn't start. Some systems empty `/tmp` at boot. Create the directory again after a reboot, or remove `working_directory` when you're done debugging.

3. In the `main` context, set the maximum size of a core file with the [`worker_rlimit_core`](https://nginx.org/en/docs/ngx_core_module.html#worker_rlimit_core) directive. Choose a value larger than the memory of a worker process:

    ```nginx
    worker_rlimit_core 500M;
    ```

    Each worker process sets this limit for itself while it still runs as `root`. You don't need to raise the core file limit of NGINX with `ulimit` or `prlimit`.

4. Check the configuration and reload NGINX:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

The settings apply to worker processes, which handle the requests. They don't change the core file limit or the directory of the master process.

## Get a backtrace from a core file {#backtrace}

A backtrace lists the function calls that led to the crash.

1. Open the core file in GDB with the NGINX binary that crashed:

    ```shell
    sudo gdb <NGINX_BINARY> <CORE_FILE>
    ```

    Replace `<NGINX_BINARY>` with the path to the NGINX binary, for example `/usr/sbin/nginx`, and `<CORE_FILE>` with the path to the core file. Use exactly the binary that crashed: a backtrace from a different build is wrong.

2. Get the backtrace:

    ```none
    (gdb) backtrace full
    ```

    `backtrace full` also shows the local variables of each function. If NGINX uses [thread pools](https://nginx.org/en/docs/ngx_core_module.html#thread_pool), run `thread apply all backtrace full` to see every thread.

3. Quit GDB with **Ctrl+D**.

If GDB reports "No symbol table info available", the binary has no debugging symbols. See [Debugging symbols](#compile_symbols).

## See the configuration that NGINX loaded {#configdump}

The configuration that NGINX runs with can differ from the files on disk, for example if someone changed a file after the last reload, or deleted it. You might need the loaded configuration to check what NGINX actually runs, or to restore lost files.

- `nginx -T` tests the files **on disk** and prints them. It doesn't show what's loaded.
- The [Control API]({{< ref "/nginx/admin-guide/basic-functionality/runtime-control.md#control-api" >}}) returns the loaded configuration. It's available in NGINX Open Source 1.31.5 and later. NGINX must be built with `--with-control-api` and started with the `-l` option. The response is a list of JSON objects with the `name` and `content` of each file:

    ```shell
    sudo curl -s --unix-socket /run/nginx-control.sock http://localhost/1/control/config \
        | jq -r '.[] | "# configuration file \(.name):\n\(.content)"'
    ```

- Without the Control API, get the loaded configuration from a running process with GDB, as the next steps show. NGINX keeps a copy of the configuration in memory only if the binary has [debug support](#enable) or the Control API is enabled.

To get the configuration with GDB:

1. Check that the binary has [debug support](#enable) and [debugging symbols](#compile_symbols).

2. Find the PID of the master process:

    ```shell
    ps -o pid,cmd -C nginx,nginx-debug
    ```

    The master process shows as `nginx: master process`. A worker process also has the configuration.

3. Attach GDB to the process:

    ```shell
    sudo gdb -p <NGINX_PID>
    ```

    Replace `<NGINX_PID>` with the PID from the previous step.

4. Paste this script into GDB and press **Enter**. It appends each configuration file to `nginx_conf.txt` in the current directory. Delete an old `nginx_conf.txt` first:

    ```none
    set $cd = ngx_cycle->config_dump
    set $nelts = $cd.nelts
    set $elts = (ngx_conf_dump_t*)($cd.elts)
    while ($nelts-- > 0)
    set $name = $elts[$nelts]->name.data
    printf "Dumping %s to nginx_conf.txt\n", $name
    append memory nginx_conf.txt \
          $elts[$nelts]->buffer.start $elts[$nelts]->buffer.end
    end
    ```

    The script writes the files in the reverse order of how NGINX read them. It doesn't add file names or separators to `nginx_conf.txt`. The names appear only in the "Dumping" lines that GDB prints.

5. Quit GDB with **Ctrl+D**, and confirm that you want to detach.

6. Open the file:

    ```shell
    less nginx_conf.txt
    ```

## Example

This configuration writes the debugging log for one client address, keeps the normal error log at the `error` level for everyone else, and prepares worker processes for core dumps:

```nginx
worker_processes   auto;
error_log          /var/log/nginx/error.log;
working_directory  /tmp/cores/;
worker_rlimit_core 500M;

events {
    debug_connection 192.0.2.10;
}

http {
    server {
        listen 80;
        root   /usr/share/nginx/html;
    }
}
```

Requests from `192.0.2.10` write `[debug]` lines to `/var/log/nginx/error.log`. If a worker process crashes, its core file goes to `/tmp/cores/`. This works if the core pattern is a file name, as described in [Prepare the operating system](#coredump_os).

## Verify the configuration

1. Check that the binary has debug support. For the NGINX Plus debug binary, run `nginx-debug -V` instead:

    ```shell
    nginx -V 2>&1 | grep -o -- '--with-debug'
    ```

    The output is `--with-debug`.

2. Send a request from the address in `debug_connection`, then look for debug lines:

    ```shell
    grep -c '\[debug\]' /var/log/nginx/error.log
    ```

    The count is larger than `0`. Lines of a request look like this:

    ```none
    2026/11/19 10:59:53 [debug] 991660#991660: *1 http request line: "GET / HTTP/1.1"
    ```

3. Check the core file limit and the current directory of a worker process:

    ```shell
    WORKER=$(pgrep -f 'nginx: worker process' | head -1)
    grep 'core file' /proc/$WORKER/limits
    sudo readlink /proc/$WORKER/cwd
    ```

    The core file limit is `524288000` bytes in both the `Soft Limit` and `Hard Limit` columns. In NGINX, `M` means 1024 × 1024 bytes. The directory is `/tmp/cores`.

## Troubleshooting

### There are no debug lines in the log

**Cause:** The binary doesn't have debug support, or a more specific `error_log` directive replaces the one with the `debug` level.

**Fix:** Run `nginx -V 2>&1 | grep -o -- '--with-debug'`, or `nginx-debug -V` for the NGINX Plus debug binary. If the output is empty, [get a binary with debug support](#enable). Then search the configuration for `error_log` directives in `http`, `server`, and `location` blocks, with `sudo nginx -T | grep error_log`. For a name-based virtual server, the start of each request goes to the [default server's log](#error_log_server).

### `nginx -t` fails with "nginx was built without debug support"

**Cause:** `error_log memory:` needs a binary with debug support.

**Fix:** [Get a binary with debug support](#enable), or write the log to a file.

### The warning says "debug_connection" is ignored

**Cause:** The binary doesn't have debug support.

**Fix:** [Get a binary with debug support](#enable).

### GDB can't attach to the process

**Cause:** Attaching to a process that GDB didn't start needs `root` privileges on most Linux systems.

**Fix:** Run GDB with `sudo`.

### No core file appears after a crash

**Cause:** The core pattern sends core files to another program, the directory isn't writable, or the size limit is too small.

**Fix:**

- Check the core pattern with `cat /proc/sys/kernel/core_pattern`, as described in [Prepare the operating system](#coredump_os).
- Check that the worker processes can write to the `working_directory` directory.
- Check the limit with `grep 'core file' /proc/<WORKER_PID>/limits`, and raise `worker_rlimit_core` if it's too small.

### GDB reports "No symbol table info available"

**Cause:** The binary has no debugging symbols.

**Fix:** See [Debugging symbols](#compile_symbols).

## Ask for help

When you ask for help with a problem, include:

1. The NGINX version, the compiler version, and the `configure` arguments, from the output of `nginx -V`.
2. The full NGINX configuration. See [See the configuration that NGINX loaded](#configdump).
3. The debugging log. See [Enable the debugging log](#error_log).
4. The backtrace, if NGINX crashed. See [Enable core dumps](#coredump) and [Get a backtrace from a core file](#backtrace).

## References

- [A debugging log](https://nginx.org/en/docs/debugging_log.html) on nginx.org
- [`error_log`](https://nginx.org/en/docs/ngx_core_module.html#error_log), [`debug_connection`](https://nginx.org/en/docs/ngx_core_module.html#debug_connection), [`working_directory`](https://nginx.org/en/docs/ngx_core_module.html#working_directory), and [`worker_rlimit_core`](https://nginx.org/en/docs/ngx_core_module.html#worker_rlimit_core) directives
- [Configuring Logging]({{< ref "/nginx/admin-guide/monitoring/logging.md" >}})
- [Control NGINX with Control API]({{< ref "/nginx/admin-guide/basic-functionality/runtime-control.md#control-api" >}})
