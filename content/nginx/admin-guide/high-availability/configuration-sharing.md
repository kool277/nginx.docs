---
description: Synchronize the configuration of a cluster of F5 NGINX Plus instances with nginx-sync. It's designed for high availability, but works for any cluster.
title: Synchronizing NGINX Configuration in a Cluster
toc: true
weight: 300
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "configuration synchronization, cluster, nginx-sync, nginx-sync.sh, high availability, rsync, SSH, primary node, peer"
f5-summary: >
  This guide shows how to push the configuration of one F5 NGINX Plus node, the primary node, to the other nodes of a cluster with the nginx-sync package.
  The nginx-sync.sh script checks the configuration, backs it up on each peer, copies it, and reloads NGINX Plus on the peers.
  If a step fails on a peer, the script restores the backup there.
f5-audience: operator
---

## Overview

F5 NGINX Plus often runs in a cluster of two or more nodes, for example for [high availability]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}}). The nodes need the same configuration. The **nginx-sync** package pushes the configuration from one node, the primary node, to the other nodes, its peers:

<img src="/nginx/images/nginx-plus-config-synchronization.png" alt="The primary node runs nginx-sync.sh, which copies its NGINX Plus configuration to each peer node" style="border:2px solid #666666; padding:2px; margin:2px;" />

The `nginx-sync.sh` script of the package:

- Checks the prerequisites of the system.
- Tests the configuration of the primary node with `nginx -t`, and stops if the test fails.
- Backs up the configuration on each peer.
- Copies the configuration to the peers with `rsync`, tests it on each peer with `nginx -t`, and then reloads NGINX Plus on the peer with `service nginx reload`.
- Restores the backup on a peer if a step fails there.

The script copies configuration files only. To share runtime state, such as session persistence data, see [Runtime State Sharing in a Cluster]({{< ref "/nginx/admin-guide/high-availability/zone_sync.md" >}}).

---

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus on all nodes**: On an operating system that supports the **nginx-sync** package: a RHEL-based system, Amazon Linux 2023, Debian, Ubuntu, or SLES. For details, see the [technical specifications]({{< ref "/nginx/technical-specs.md" >}}).
- **rsync on all nodes**: The script copies the files with `rsync` over SSH, which also runs `rsync` on each peer.
- **Root access**: You need `sudo` on all nodes.

---

<span id="instructions"></span><span id="in_detail1"></span><span id="installing-nginx-sync-on-the-primary-machine"></span>

## Install nginx-sync on the primary node

On the primary node, install the **nginx-sync** package from the NGINX Plus repository:

- RHEL and its derivatives, and Amazon Linux 2023:

    ```shell
    sudo dnf install nginx-sync
    ```

- Debian and Ubuntu:

    ```shell
    sudo apt install nginx-sync
    ```

- SLES:

    ```shell
    sudo zypper install nginx-sync
    ```

---

<span id="in_detail2"></span><span id="configuring-root-ssh-access-to-the-peers"></span>

## Set up root SSH access to the peers

The script runs as `root` on the primary node. It connects to each peer as `root` over SSH. There it copies files that `root` owns, such as the files in **/etc/nginx**, and runs commands such as `service nginx reload`.

<span id="faq1"></span><span id="why-do-i-need-to-grant-ssh-access-to-root"></span>

{{< call-out class="caution" >}} After these steps, anyone with `root` access on the primary node also has `root` access on the peers. Any process that can change the NGINX Plus configuration of a node and reload NGINX Plus can gain `root` access to that node anyway. Protect the primary node accordingly. {{< /call-out >}}

1. On the primary node, create an SSH key pair for `root`, and show its public key. Accept the default file name, and leave the passphrase empty, so that the script can use the key without a prompt:

    ```shell
    sudo ssh-keygen -t rsa -b 2048
    sudo cat /root/.ssh/id_rsa.pub
    ```

    The output is the public key:

    ```none
    ssh-rsa AAAAB3Nz4rFgt...vgaD root@node1
    ```

1. Find the IP address of the primary node, for example `192.168.1.2`:

    ```shell
    ip -4 -brief address
    ```

1. On each peer, add the public key to the **authorized_keys** file of `root`. The `from="192.168.1.2"` option accepts the key only from the address of the primary node:

    ```shell
    sudo mkdir -p -m 700 /root/.ssh
    echo 'from="192.168.1.2" ssh-rsa AAAAB3Nz4rFgt...vgaD root@node1' | sudo tee -a /root/.ssh/authorized_keys
    ```

    Use `sudo tee -a` to append to the file. With `sudo echo ... >> /root/.ssh/authorized_keys`, your own shell opens the file before `sudo` runs, and fails with `Permission denied`.

1. On each peer, check that the SSH server lets `root` log in with a key:

    ```shell
    sudo sshd -T | grep -i permitrootlogin
    ```

    The value must be `yes`, `prohibit-password`, or `without-password`, an older name of `prohibit-password`. `prohibit-password` is the default of OpenSSH. If the value is `no`, change the `PermitRootLogin` line in **/etc/ssh/sshd_config**, or in a file in **/etc/ssh/sshd_config.d**, to:

    ```none
    PermitRootLogin prohibit-password
    ```

    The SSH server uses the first value that it reads for a keyword. So change the existing line, instead of adding one at the end of the file. Then restart the SSH server on the peer:

    - RHEL and its derivatives, and SLES:

        ```shell
        sudo systemctl restart sshd
        ```

    - Debian and Ubuntu:

        ```shell
        sudo systemctl restart ssh
        ```

1. On the primary node, check that `root` can connect to each peer without a password. The `hostname` command runs on the peer, and prints its name:

    ```shell
    sudo ssh root@node2.example.com hostname
    ```

    On the first connection, `ssh` asks you to confirm the host key of the peer. Answer `yes`, so that the script doesn't stop at this prompt later.

---

<span id="in_detail3"></span><span id="creating-the-nginx-syncconf-configuration-file-on-the-primary-node"></span>

## Create the nginx-sync configuration file

On the primary node, create the file **/etc/nginx-sync.conf** with these contents:

```none
NODES="node2.example.com node3.example.com node4.example.com"
CONFPATHS="/etc/nginx/nginx.conf /etc/nginx/conf.d"
EXCLUDE="default.conf"
```

With this file, the script copies **/etc/nginx/nginx.conf** and the directory **/etc/nginx/conf.d** to the three peers, without the files named **default.conf**.

### Common parameters

Separate the items of each list with spaces or new lines.

{{< table >}}

| Parameter | Description |
|---|---|
| `NODES` | The peers that receive the configuration of the primary node. |
| `CONFPATHS` | The files and directories to copy from the primary node to the peers. |
| `EXCLUDE` | Optional. The configuration files of the primary node not to copy to the peers. |

{{< /table >}}

### Advanced parameters

{{< table >}}

| Parameter | Description | Default |
|---|---|---|
| `BACKUPDIR` | The location of the backup on each peer. | **/var/lib/nginx-sync** |
| `DIFF` | The location of the `diff` binary. | **/usr/bin/diff** |
| `LOCKFILE` | The location of the lock file, which lets only one `nginx-sync` operation run at a time. | **/tmp/nginx-sync.lock** |
| `NGINX` | The location of the **nginx-plus** binary. | **/usr/sbin/nginx** |
| `POSTSYNC` | A space-separated list of substitutions to make in files on each peer, in the format `'<filename>\|<sed-expression>'`. The script applies each one in place, with `sed -i' ' <sed-expression> <filename>`. | None |
| `RSYNC` | The location of the `rsync` binary. | **/usr/bin/rsync** |
| `SSH` | The location of the `ssh` binary. | **/usr/bin/ssh** |

{{< /table >}}

The script applies the same substitutions on every peer. So use `POSTSYNC` only for changes that all peers need, not for values that differ between the nodes, such as the addresses in **keepalived.conf**. For example, to turn off write access to the NGINX Plus API on the peers:

```none
POSTSYNC="/etc/nginx/conf.d/api.conf|'s/api write=on/api write=off/'"
```

---

<span id="in_detail4"></span><span id="testing-the-configuration"></span>

## Synchronize the configuration

Back up the configuration before you run the script for the first time. Run the script as `root` on the primary node, because it uses the SSH key of `root`:

- To copy the configuration and reload NGINX Plus on the peers:

    ```shell
    sudo nginx-sync.sh
    ```

- To show the usage information:

    ```shell
    sudo nginx-sync.sh -h
    ```

- To compare the configuration of the primary node with one peer:

    ```shell
    sudo nginx-sync.sh -c <PEER_NODE>
    ```

- To compare the configuration of the primary node with all peers:

    ```shell
    sudo nginx-sync.sh -C
    ```

---

<span id="faq"></span><span id="frequently-asked-questions"></span>

## Handle failed nodes

<span id="faq2"></span><span id="how-do-i-synchronize-configuration-if-the-primary-fails"></span>

### If the primary node fails

If the primary node fails, and won't return to service soon, make a peer the new primary node. On that peer:

1. [Install nginx-sync](#in_detail1).
1. [Set up root SSH access](#in_detail2) to the remaining peers.
1. [Create the nginx-sync configuration file](#in_detail3).

You can prepare several nodes to work as the primary node, but only one node may act as the primary node at a time.

<span id="faq3"></span><span id="what-happens-if-a-peer-node-fails"></span>

### If a peer node fails

A peer that is down doesn't receive configuration changes. The script reports an error for that peer, and still copies the configuration to the other peers.

When the peer is up again, its configuration is out of date. To show the differences, run `nginx-sync.sh -c` with the `-d` option on the primary node:

```shell
sudo nginx-sync.sh -c node2.example.com -d
```

The output shows the differences as a diff:

```diff
diff -ru /tmp/localconf.1XrIqP7f/etc/nginx/conf.d/responder.conf /tmp/remoteconf.Xq5LWGKU/etc/nginx/conf.d/responder.conf
--- /tmp/localconf.1XrIqP7f/etc/nginx/conf.d/responder.conf    2020-09-25 10:29:36.988064021 -0800
+++ /tmp/remoteconf.Xq5LWGKU/etc/nginx/conf.d/responder.conf    2020-09-25 10:28:39.764066539 -0800
@@ -4,6 +4,6 @@
    listen 80;

    location / {
-        return 200 "Received request on $server_addr on host $hostname blue\n";
+        return 200 "Received request on $server_addr on host $hostname red\n";
    }
}

* Synchronization ended at Fri Sep 25 18:30:49 UTC 2020
```

The next run of `nginx-sync.sh` updates the peer with the configuration of the primary node.

---

## Troubleshooting

### ssh asks for a password or reports Permission denied

**Symptom**: `sudo ssh root@node2.example.com hostname` asks for a password, or fails with a message such as `Permission denied (publickey)`.

**Cause**: The peer doesn't accept the key of `root` on the primary node. For example, `PermitRootLogin` is `no`, or the key isn't in **/root/.ssh/authorized_keys**. Or the `from` option has another address than the primary node.

**Fix**: On the peer, check the output of `sudo sshd -T | grep -i permitrootlogin`. Also check the line in **/root/.ssh/authorized_keys**. Then repeat the check from the primary node.

---

### nginx-sync.sh stops at a host key prompt

**Symptom**: The script stops with a question such as `Are you sure you want to continue connecting (yes/no/[fingerprint])?`.

**Cause**: `root` on the primary node hasn't connected to that peer before, so `ssh` doesn't know its host key.

**Fix**: Connect once with `sudo ssh root@<PEER_NODE> hostname`, confirm the host key, and run the script again.

---

## References

For more information, see:

- [High Availability Support for NGINX Plus in On-Premises Deployments]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}})
- [Runtime State Sharing in a Cluster]({{< ref "/nginx/admin-guide/high-availability/zone_sync.md" >}})
- [The sshd_config manual page](https://man.openbsd.org/sshd_config)
