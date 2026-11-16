---
description: Capture detailed information about errors and request processing in log
  files, either locally or via syslog.
title: Configuring Logging
toc: true
weight: 200
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "error_log, access_log, log_format, syslog, conditional logging, upstream timing"
f5-summary: >
  This page explains how to configure the error log and the access log, define log formats, log conditionally, and send logs to syslog.
  Logs are the main tool for troubleshooting NGINX and understanding its traffic.
  It applies to NGINX Open Source and NGINX Plus, and marks the features that need NGINX Plus.
f5-audience: operator
---

## Overview

NGINX Open Source and F5 NGINX Plus write two kinds of logs:

- The **error log** records problems and other events, at several severity levels.
- The **access log** records each request after NGINX has processed it.

Use this guide to set where the logs go, what they contain, and which requests NGINX logs.

---

## Before you begin

Before you begin, make sure you have:

- **NGINX Open Source or NGINX Plus installed and running**.
- **Permission to edit the configuration and reload NGINX**: On most systems, this means root or `sudo` access.

---

## Set up the error log {#error_log}

The [error_log](https://nginx.org/en/docs/ngx_core_module.html#error_log) directive sets where NGINX writes the error log and the minimum severity level of the messages. The log can go to a file, to `stderr`, or to syslog. Builds with `--with-debug` can also log to a memory buffer; see [Debugging NGINX]({{< ref "/nginx/admin-guide/monitoring/debugging.md" >}}).

### Default path

If no `error_log` directive in the main context writes to a file, NGINX also writes errors to the path set at build time, by default **logs/error.log** inside the installation prefix. The `-e` command-line option overrides this path. To find the build-time path, look for `--error-log-path=` in the output of `nginx -V`:

```shell
nginx -V 2>&1 | sed -n 's/.*--error-log-path=\([^ ]*\).*/\1/p'
```

The output depends on the package, for example:

- `/var/log/nginx/error.log` for the packages from nginx.org and for NGINX Plus.
- `stderr` for some distribution packages, such as those in Debian and Ubuntu. NGINX then writes the errors to its standard error, for example to the systemd journal, until the configuration sets a file with `error_log`.

If the command prints nothing, NGINX was built without `--error-log-path`, and the path is **logs/error.log** inside the `--prefix=` directory. To find the default access log path, use the same command with `--http-log-path`. In the official Docker images, `/var/log/nginx/error.log` is a link to `/dev/stderr`, so the errors appear in the container logs.

### Severity level

The severity levels follow the syslog classification. From the lowest to the highest, they are `debug`, `info`, `notice`, `warn`, `error`, `crit`, `alert`, and `emerg`. NGINX logs messages of the level you set and all higher levels. The default level is `error`.

This example lowers the level from `error` to `warn`:

```nginx
error_log logs/error.log warn;
```

With this setting, NGINX logs messages of the `warn`, `error`, `crit`, `alert`, and `emerg` levels.

For debug logging, see [Debugging NGINX]({{< ref "/nginx/admin-guide/monitoring/debugging.md" >}}).

### Error log formats

NGINX writes the error log in its standard text format. NGINX Plus can also write the error log in JSON with the [json](https://nginx.org/en/docs/ngx_core_module.html#error_log_json) parameter:

```nginx
error_log /var/log/nginx/error.json error json;
```

This is an example of a JSON error log entry:

```json
{
  "level": "error",
  "timestamp": "2026-05-04T10:30:15.042+00:00",
  "pid": 12345,  "tid": 12345,  "cnum": 3,
  "msg": "connect() failed",
  "client": "192.168.1.10",  "server": "example.com",
  "request": "GET /api HTTP/1.1",
  "upstream": "http://127.0.0.1:8080/api",
  "errno": 111,
  "errtext": "Connection refused"
}
```

### Error log inheritance

An `error_log` directive in the main context applies to the whole configuration. You can also use `error_log` in the [http](https://nginx.org/en/docs/http/ngx_http_core_module.html#http), [stream](https://nginx.org/en/docs/stream/ngx_stream_core_module.html#stream), [mail](https://nginx.org/en/docs/mail/ngx_mail_core_module.html#mail), `server`, and [location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location) contexts. A directive at a lower level replaces the one inherited from a higher level.

NGINX writes each message once, to the error log that applies where the error occurred. If one level has several `error_log` directives, NGINX writes the message to all of them.

## Set up the access log {#access_log}

NGINX writes an entry to the access log after it finishes processing a request. The [access_log](https://nginx.org/en/docs/http/ngx_http_log_module.html#access_log) directive sets the location and the format of the log. By default, the access log is **logs/access.log** inside the installation prefix, unless NGINX was built with another `--http-log-path`. The default format is the predefined **combined** format.

To define another format, use the [log_format](https://nginx.org/en/docs/http/ngx_http_log_module.html#log_format) directive in the `http` context. A format is a string with variables.

This example extends the **combined** format with the compression ratio of the response, in the `$gzip_ratio` variable. The `gzip on` directive turns on compression for the virtual server:

```nginx
http {
    log_format compression '$remote_addr - $remote_user [$time_local] '
                           '"$request" $status $body_bytes_sent '
                           '"$http_referer" "$http_user_agent" "$gzip_ratio"';

    server {
        listen 80;
        gzip on;
        access_log /var/log/nginx/nginx-access.log compression;
    }
}
```

### Log upstream timing

To find out where a slow request spends its time, log these time variables:

- [$upstream_connect_time](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#var_upstream_connect_time) – Time to set up the connection to the upstream server.
- [$upstream_header_time](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#var_upstream_header_time) – Time from the start of connecting to the upstream server until NGINX received the response header.
- [$upstream_response_time](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#var_upstream_response_time) – Time from the start of connecting to the upstream server until NGINX received the whole response.
- [$request_time](https://nginx.org/en/docs/http/ngx_http_log_module.html#var_request_time) – Total time that NGINX spent on the request, from the first bytes it read from the client.

All times are in seconds with millisecond resolution.

```nginx
http {
    log_format upstream_time '$remote_addr - $remote_user [$time_local] '
                             '"$request" $status $body_bytes_sent '
                             '"$http_referer" "$http_user_agent" '
                             'rt="$request_time" uct="$upstream_connect_time" '
                             'uht="$upstream_header_time" urt="$upstream_response_time"';

    server {
        listen 80;
        access_log /var/log/nginx/nginx-access.log upstream_time;

        location / {
            proxy_pass http://127.0.0.1:8000;
        }
    }
}
```

When you read the upstream time values, keep in mind:

- If NGINX tried several servers for a request, the variable has one value per server, separated by a comma and a space, for example `urt="-, 0.004"`.
- If an internal redirect passed the request to an upstream group again, the same group or another one, for example with `error_page` or `X-Accel-Redirect`, the values of each pass are separated by ` : `, for example `urt="0.001 : 0.003"`.
- If NGINX didn't measure a time, the value is `-` (hyphen). For example, `$upstream_header_time` is `-` if NGINX didn't receive the response header, and `$upstream_connect_time` is `-` if NGINX couldn't connect.

### Buffer the access log

To write the access log less often, use the `buffer` parameter of [access_log](https://nginx.org/en/docs/http/ngx_http_log_module.html#access_log). NGINX keeps entries in memory and writes them when the buffer is full, or after the time set by the `flush` parameter:

```nginx
access_log /var/log/nginx/access.log combined buffer=32k flush=5s;
```

If the path of a log file contains variables, the [open_log_file_cache](https://nginx.org/en/docs/http/ngx_http_log_module.html#open_log_file_cache) directive keeps the file descriptors of frequently used files open.

### Access log inheritance

As with `error_log`, an `access_log` directive at a lower level replaces the ones inherited from higher levels. NGINX writes the entry for a request to the logs that apply where processing ends. If one level has several `access_log` directives, NGINX writes the entry to all of them. To turn off the access log in a context, use `access_log off`.

## Enable conditional logging {#conditional}

To leave unimportant entries out of the access log, use the `if` parameter of [access_log](https://nginx.org/en/docs/http/ngx_http_log_module.html#access_log). NGINX logs a request only if the condition isn't empty and isn't `0`.

This example doesn't log requests with `2xx` and `3xx` status codes. The `map` directive belongs in the `http` context:

```nginx
map $status $loggable {
    ~^[23]  0;
    default 1;
}

access_log /var/log/nginx/access.log combined if=$loggable;
```

## Use case: sampling TLS parameters {#tls_sample}

This use case needs NGINX Plus, for the key-value store.

Many clients still use TLS versions older than TLS 1.3, and some use ciphers that are no longer considered secure. If you turn off weak protocols and ciphers, you protect clients from downgrade attacks, but legacy clients can't connect anymore. Logging the TLS parameters of your clients shows how many clients a change would affect.

The following configuration logs the TLS protocol, the cipher, and the `User-Agent` header once for each combination of client IP address and `User-Agent`:

1. Define a log format, for example `sslparams`, with the TLS protocol ([$ssl_protocol](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#var_ssl_protocol)), the cipher ([$ssl_cipher](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#var_ssl_cipher)), the client IP address ([$remote_addr](https://nginx.org/en/docs/http/ngx_http_core_module.html#var_remote_addr)), and the `User-Agent` header (`$http_user_agent`):

    ```nginx
    log_format sslparams '$ssl_protocol $ssl_cipher '
                         '$remote_addr "$http_user_agent"';
    ```

2. Define a key-value store, for example `clients`, that keeps the client IP addresses and `User-Agent` headers:

    ```nginx
    keyval_zone zone=clients:80m timeout=3600s;
    ```

3. Create a variable, for example `$seen`, for each combination of `$remote_addr` and the `User-Agent` header, and log a request only the first time its combination is seen:

    ```nginx
    keyval $remote_addr:$http_user_agent $seen zone=clients;

    server {
        listen 443 ssl;

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers   HIGH:!aNULL:!MD5;

        # ssl_certificate and ssl_certificate_key are required here

        set $logme 0;

        if ($seen = "") {
            set $seen  1;
            set $logme 1;
        }
        access_log  /tmp/sslparams.log sslparams if=$logme;

        # ...
    }
    ```

    The `set $logme 0` line gives the variable a value for clients that were seen before. Without it, NGINX logs a warning about an uninitialized variable for each of their requests.

4. View the log file:

    ```none
    TLSv1.2 AES128-SHA 1.1.1.1 "Mozilla/5.0 (X11; Linux x86_64; rv:45.0) Gecko/20100101 Firefox/45.0"
    TLSv1.2 ECDHE-RSA-AES128-GCM-SHA256 2.2.2.2 "Mozilla/5.0 (iPhone; CPU iPhone OS 9_1 like Mac OS X) AppleWebKit/601.1.46 (KHTML, like Gecko) Version/9.0 Mobile/13B143 Safari/601.1"
    TLSv1.2 ECDHE-RSA-AES128-GCM-SHA256 3.3.3.3 "Mozilla/5.0 (Windows NT 6.1; WOW64; rv:58.0) Gecko/20100101 Firefox/58.0"
    TLSv1.2 ECDHE-RSA-AES128-GCM-SHA256 4.4.4.4 "Mozilla/5.0 (Android 4.4.2; Tablet; rv:65.0) Gecko/65.0 Firefox/65.0"
    TLSv1 AES128-SHA 5.5.5.5 "Mozilla/5.0 (Android 4.4.2; Tablet; rv:65.0) Gecko/65.0 Firefox/65.0"
    TLSv1.2 ECDHE-RSA-CHACHA20-POLY1305 6.6.6.6 "Mozilla/5.0 (Linux; U; Android 5.0.2; en-US; XT1068 Build/LXB22.46-28) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/57.0.2987.108 UCBrowser/12.10.2.1164 Mobile Safari/537.36"
    ```

5. Count the clients per cipher. With a larger log, the output looks like this example:

    ```shell
    cut -d ' ' -f 2 /tmp/sslparams.log | sort | uniq -c | sort -rn | perl -ane 'printf "%30s %s\n", $F[1], "="x$F[0];'
    ```

    The output shows which ciphers few clients use, here `AES128-SHA` and others:

    ```none
    ECDHE-RSA-AES128-GCM-SHA256 =========================
    ECDHE-RSA-AES256-GCM-SHA384 ========
                     AES128-SHA ====
    ECDHE-RSA-CHACHA20-POLY1305 ==
        ECDHE-RSA-AES256-SHA384 ==
    ```

    Check the log for the clients that use these ciphers, and then decide whether to remove the ciphers from the NGINX Plus configuration.

For more about sampling requests with conditional logging, see the [blog post](https://www.nginx.com/blog/sampling-requests-with-nginx-conditional-logging/).

## Logging to syslog {#syslog}

Syslog is a standard for collecting log messages from many devices on one server. To send logs to syslog, use the `syslog:` prefix in the [error_log](https://nginx.org/en/docs/ngx_core_module.html#error_log) and [access_log](https://nginx.org/en/docs/http/ngx_http_log_module.html#access_log) directives.

The `server=` parameter sets the syslog server: a domain name, an IP address, or a UNIX-domain socket path after the `unix:` prefix. A domain name or IP address can include a port; the default port is UDP `514`.

```nginx
error_log  syslog:server=unix:/var/log/nginx.sock debug;
access_log syslog:server=[2001:db8::1]:1234,facility=local7,tag=nginx,severity=info;
```

In this example, NGINX sends error log messages of the `debug` level and higher to a UNIX-domain socket. Debug messages exist only in builds with `--with-debug`. It sends the access log to a syslog server at an IPv6 address on port `1234`.

Other parameters:

- `facility=` – The type of program that logs the message. The default is `local7`. Other values are `kern`, `user`, `mail`, `daemon`, `auth`, `intern`, `lpr`, `news`, `uucp`, `clock`, `authpriv`, `ftp`, `ntp`, `audit`, `alert`, `cron`, and `local0` to `local6`.
- `tag=` – The tag of the messages, up to 32 letters, digits, and underscores. The default is `nginx`.
- `severity=` – The severity level that NGINX gives to access log messages: `debug`, `info`, `notice`, `warn`, `error`, `crit`, `alert`, or `emerg`. The default is `info`. It sets the label of the messages; it doesn't filter them. For the error log, NGINX uses the level of each message.
- `nohostname` – Leaves the host name out of the messages.

The `server=` parameter is required. NGINX resolves a domain name once, when it loads the configuration, and uses the first address.

## Live activity monitoring {#monitoring}

NGINX Plus provides a live activity monitoring interface that shows load and performance metrics of your [HTTP]({{< ref "/nginx/admin-guide/load-balancer/http-load-balancer.md" >}}) and [TCP]({{< ref "/nginx/admin-guide/load-balancer/tcp-udp-load-balancer.md" >}}) upstream servers. For more information, see [Live Activity Monitoring]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}}).

---

## Example

This complete configuration writes errors of the `warn` level and higher to a file, logs requests with upstream timing, and leaves requests to **/health** out of the access log file. It also sends the log of all requests, including **/health**, to a syslog server.

It's a whole **nginx.conf**, so back up your existing file first. It needs an application server on `127.0.0.1:8000` and a syslog server that accepts UDP messages on `127.0.0.1:514`. Without the syslog server, NGINX logs `send() failed` errors.

```nginx
error_log /var/log/nginx/error.log warn;

events {}

http {
    log_format timed '$remote_addr - $remote_user [$time_local] '
                     '"$request" $status $body_bytes_sent '
                     'rt=$request_time urt=$upstream_response_time';

    map $uri $loggable {
        /health  0;
        default  1;
    }

    server {
        listen 80;

        access_log /var/log/nginx/access.log timed if=$loggable;
        access_log syslog:server=127.0.0.1:514,tag=nginx timed;

        location / {
            proxy_pass http://127.0.0.1:8000;
        }

        location = /health {
            return 200 "ok\n";
        }
    }
}
```

---

## Verify the configuration

1. Test the configuration, and start NGINX or reload it if it's running:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

2. Send a request, and check the last access log entry:

    ```shell
    curl -s -o /dev/null http://localhost/
    sudo tail -1 /var/log/nginx/access.log
    ```

    The entry ends with the request time and the upstream response time, for example:

    ```none
    127.0.0.1 - - [04/May/2026:10:30:15 +0000] "GET / HTTP/1.1" 200 1024 rt=0.004 urt=0.004
    ```

3. Request **/health** and check that NGINX didn't log it:

    ```shell
    curl -s -o /dev/null http://localhost/health
    sudo tail -1 /var/log/nginx/access.log
    ```

    The last entry is still the one from step 2.

---

## Troubleshooting

### Nothing is written to the log file

**Symptom**: The log file stays empty or doesn't exist.

**Cause**: A lower configuration level has its own `access_log` or `error_log` that replaces yours, `access_log off` is set, an `if=` condition is `0`, or the worker processes can't write to the file.

**Fix**: Run `sudo nginx -T | grep -E 'access_log|error_log'` to find all log directives. Check the error log for `open() ... failed` messages, and the permissions of the log directory.

### Expected messages are missing from the error log

**Symptom**: Messages such as `upstream server temporarily disabled` don't appear.

**Cause**: The message has a lower severity than the configured level. The default level is `error`.

**Fix**: Lower the level, for example to `warn` or `info`, in the `error_log` directive that applies.

### Upstream time variables show "-"

**Symptom**: `$upstream_header_time` or `$upstream_connect_time` is `-`.

**Cause**: NGINX couldn't connect to the upstream server or didn't receive the response header, or the response came from the cache.

**Fix**: Check the error log for the failed upstream connection or timeout.

---

## Compatibility note {#compat}

- Several `error_log` directives on the same level are supported since NGINX Open Source [1.5.2](https://nginx.org/en/CHANGES).
- JSON error logs are supported since NGINX Plus [PLS.37.0.0.1 LTS]({{< ref "/nginx/releases.md#r37.0" >}}).

---

## References

For more information, see:

- [Debugging NGINX]({{< ref "/nginx/admin-guide/monitoring/debugging.md" >}})
- [ngx_http_log_module reference](https://nginx.org/en/docs/http/ngx_http_log_module.html) on nginx.org
- [error_log directive](https://nginx.org/en/docs/ngx_core_module.html#error_log) on nginx.org
- [Logging to syslog](https://nginx.org/en/docs/syslog.html) on nginx.org
