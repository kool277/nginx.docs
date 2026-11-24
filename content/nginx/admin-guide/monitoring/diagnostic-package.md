---
description: Collect the data that F5 Technical Support needs to troubleshoot an NGINX Open Source or F5 NGINX Plus deployment, with the nginx-supportpkg.sh script.
title: NGINX Diagnostic Package
toc: true
weight: 400
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "diagnostic package, support package, nginx-supportpkg.sh, troubleshooting, F5 Technical Support, logs, configuration"
f5-summary: >
  This guide shows how to create an NGINX diagnostic package with the nginx-supportpkg.sh script.
  The package has the data that F5 Technical Support needs to troubleshoot an NGINX Open Source or F5 NGINX Plus deployment, such as configuration files, logs, and system information.
  You can leave parts out, and you should review the package before you share it.
f5-audience: operator
---

<span id="intro"></span>

## Overview

The NGINX diagnostic package has the information that [F5 Technical Support](https://account.f5.com/myf5) needs to troubleshoot your issue. The `nginx-supportpkg.sh` script creates it. You can [download the script](https://nginx.org/download/nginx-supportpkg.sh) from [nginx.org](https://nginx.org/download/).

The script collects:

- The output of host commands, such as `ps`, `lsof`, and `vmstat`
- NGINX configuration files
- NGINX log files
- NGINX service information
- NGINX process information
- NGINX versions and dynamically linked libraries
- The NGINX Plus API endpoints
- NGINX Plus license data: the JWT license payload, the license state file, and the subject, issuer, and dates of the repository certificate
- System logs and settings, such as the last 10,000 lines of the systemd journal, `dmesg` output, and the `iptables` rules
- NGINX Agent logs and configuration, if NGINX Agent is installed
- F5 WAF for NGINX logs and configuration, if F5 WAF for NGINX is installed

The script doesn't collect or create:

- njs scripts
- Lua scripts
- Core dumps

{{< call-out class="note" >}} Review the script and the package that it creates, and check that they meet the data sharing policies of your organization. {{< /call-out >}}

---

<span id="oses"></span><span id="supported-operating-systems"></span>

## Before you begin

The script runs on most of the [operating systems that NGINX supports]({{< ref "/nginx/technical-specs.md" >}}). It was tested on:

- AlmaLinux 9.1
- Amazon Linux 2
- CentOS 7
- Debian 11
- RHEL 9.1
- Rocky Linux 9.1
- SUSE Linux Enterprise Server 15
- Ubuntu 20.04

Before you begin, make sure you have:

- **Root access**: The script runs only as `root`.
- **Bash 4 or later**: The script stops with an error on older versions.
- **jq**: The script uses `jq` to add a JSON manifest that F5 Technical Support uses to read the package. Without `jq`, run the script with the `-xm` option, which leaves the manifest out.
- **Common tools**: `awk`, `cat`, `cp`, `egrep`, `file`, `find`, `grep`, `nginx`, `ps`, `sed`, `tar`, and `xargs`.
- **curl**: The script needs `curl` to collect the NGINX Plus API data. Without it, the script skips this data.

---

<span id="usage"></span>

## Create the diagnostic package

1. Download the `nginx-supportpkg.sh` script:

    ```shell
    wget https://nginx.org/download/nginx-supportpkg.sh
    ```

1. Make the script executable:

    ```shell
    chmod +x nginx-supportpkg.sh
    ```

1. Run the script as `root`. To change what it collects, add [arguments](#arguments):

    ```shell
    sudo ./nginx-supportpkg.sh
    ```

    The script creates the package in the current directory. The package is a `.tar.gz` archive named `support-pkg-<TIMESTAMP>.tar.gz`, where `<TIMESTAMP>` is the time in seconds since 1970. At the end, the script prints the path of the archive, in a line such as `Archive /home/user/support-pkg-1682457903.tar.gz ready`.

1. Extract the archive into a new directory, and review what it contains. The archive has no top-level directory, so extract it into a directory of its own:

    ```shell
    mkdir support-pkg
    tar -xzf support-pkg-1682457903.tar.gz -C support-pkg
    ```

    The archive holds the output of each command that the script ran, as text, so that you can review the collected data.

---

## Arguments

These arguments change what the script collects:

{{< table >}}

| Short | Long | Description | Example | Default |
|---|---|---|---|---|
| `-h` | `--help` | Prints information about the arguments. | `--help` | N/A |
| `-d` | `--debug` | Sets the debug flag of Bash. | `--debug` | N/A |
| `-o` | `--output_dir` | The directory for the archive. It must exist, because the script doesn't create it. End it with a slash (`/`), because the script adds the file name directly to it. | `-o ~/output/` | The current directory |
| `-n` | `--nginx_log_path` | The directory of the NGINX log files. The script uses it only if the main **nginx.conf** has no `access_log` directive. Otherwise, it collects the logs in the directories of those `access_log` directives. | `-n /var/log/nginx` | `/var/log/nginx` |
| `-xc` | | Leaves out all NGINX configuration files. Use the short form: the script accepts only `--exclude_nginx_conf` as the long form, not the `--exclude_nginx_configs` that its help shows. | `-xc` | N/A |
| `-xl` | `--exclude_nginx_logs` | Leaves out all NGINX log files. | `--exclude_nginx_logs` | N/A |
| `-ac` | `--exclude_agent_configs` | Leaves out all NGINX Agent configuration files. | `--exclude_agent_configs` | N/A |
| `-al` | `--exclude_agent_logs` | Leaves out all NGINX Agent logs. | `--exclude_agent_logs` | N/A |
| `-nc` | `--exclude_nap_configs` | Leaves out all F5 WAF for NGINX configuration files. | `--exclude_nap_configs` | N/A |
| `-nl` | `--exclude_nap_logs` | Leaves out all F5 WAF for NGINX log files. | `--exclude_nap_logs` | N/A |
| `-ea` | `--exclude_api_stats` | Leaves out the statistics of the NGINX Plus API. | `--exclude_api_stats` | N/A |
| `-xm` | `--exclude_manifest` | Leaves out the JSON manifest, so the script doesn't need `jq`. | `--exclude_manifest` | N/A |
| `-pi` | `--profile_interval` | The time in seconds that the script records `vmstat` output. | `-pi 20` | `15` |

{{< /table >}}

For example, to leave out the log files, and to save the archive in **/tmp**:

```shell
sudo ./nginx-supportpkg.sh -xl -o /tmp/
```

---

## Troubleshooting

### The script reports an unrecognized flag

**Symptom**: The script stops with `Unrecognized flag: --exclude_nginx_configs`.

**Cause**: The help of the script shows `--exclude_nginx_configs`, but the script accepts only `-xc` or `--exclude_nginx_conf`.

**Fix**: Use `-xc`.

---

### The script can't find jq

**Symptom**: The script stops with `Cannot find jq binary. Install jq or add it to the $PATH.`

**Cause**: `jq` isn't installed in a directory that the script searches. The script sets its own `PATH`: **/usr/sbin**, **/usr/bin**, **/sbin**, **/bin**, **/usr/local/sbin**, and **/usr/local/bin**.

**Fix**: Install `jq` with the package manager of your operating system, which puts it in one of these directories. Or run the script with `-xm` to leave out the JSON manifest.

---

### The archive has an unexpected name

**Symptom**: The script prints an archive path such as **/home/user/outputsupport-pkg-1682457903.tar.gz**, instead of a file in the **output** directory.

**Cause**: The `-o` directory doesn't end with a slash, so the script joins the directory and the file name without one.

**Fix**: End the directory with a slash, for example `-o ~/output/`.

---

### The archive is missing, although the script reports it ready

**Symptom**: The script prints `Archive ... ready`, but the file doesn't exist. `tar` reports an error just before, such as `Cannot open: No such file or directory`.

**Cause**: The `-o` directory doesn't exist. The script doesn't create it, and doesn't check the result of `tar`. It then deletes its temporary directory, so the collected data is lost.

**Fix**: Create the directory first, for example with `mkdir -p ~/output`, and run the script again.

---

## References

For more information, see:

- [The nginx-supportpkg.sh script](https://nginx.org/download/nginx-supportpkg.sh)
- [F5 Technical Support](https://account.f5.com/myf5)
- [NGINX Plus Technical Specifications]({{< ref "/nginx/technical-specs.md" >}})
