---
description: Set up an active-passive pair of F5 NGINX Plus instances on premises, with keepalived moving a virtual IP address between them with VRRP.
title: High Availability Support for NGINX Plus in On-Premises Deployments
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "high availability, active-passive, keepalived, VRRP, virtual IP address, VIP, nginx-ha-keepalived, nginx-ha-setup, failover"
f5-summary: >
  This guide shows how to make two F5 NGINX Plus instances highly available with the nginx-ha-keepalived package.
  keepalived gives a virtual IP address to the primary instance, and moves it to the backup instance when NGINX Plus or the primary fails.
  It works where the operating system controls IP addresses, which isn't the case in most cloud environments.
f5-audience: operator
---

<span id="ha_support"></span><span id="high-availability-support-based-on-keepalived"></span>

## Overview

F5 NGINX Plus R6 and later support an active-passive high availability (HA) setup on premises, based on [keepalived](https://www.keepalived.org/). Two nodes share a virtual IP address (VIP). The primary node holds the VIP and handles the traffic. If the primary node fails, the backup node takes over the VIP.

{{< call-out class="note" >}} This solution works where the operating system controls IP addresses. Most cloud environments assign IP addresses through their own infrastructure, so it often doesn't work there. For cloud environments, see the [deployment guides]({{< ref "/nginx/deployment-guides/_index.md" >}}). {{< /call-out >}}

The solution has three parts:

- **The keepalived daemon**, which runs on both nodes.
- **The [Virtual Router Redundancy Protocol](https://datatracker.ietf.org/doc/html/rfc5798) (VRRP)**. The primary node sends VRRP advertisements to the backup node, every second in this guide. If the backup node gets no advertisement for three intervals and a short skew time, it takes over the VIP ([RFC 5798, section 6.1](https://datatracker.ietf.org/doc/html/rfc5798#section-6.1)).
- **Health-check scripts**. keepalived runs them regularly, and changes the priority of the node with their results. The node with the higher priority holds the VIP.

The **nginx-ha-keepalived** package of NGINX Plus sets up all three parts.

---

## Before you begin

Before you begin, make sure you have:

- **Two NGINX Plus nodes**: Both run NGINX Plus in the same subnet, with the same NGINX Plus configuration. To keep the configuration the same, see [Synchronizing NGINX Configuration in a Cluster]({{< ref "/nginx/admin-guide/high-availability/configuration-sharing.md" >}}).
- **A free IP address for the VIP**: An unused address in the subnet of the nodes. Clients connect to this address.
- **VRRP allowed between the nodes**: VRRP uses IP protocol 112. If a firewall blocks it, both nodes take the VIP, as described in [Both nodes hold the VIP](#both-nodes-hold-the-vip).
- **Root access**: You need `sudo` on both nodes.

---

<span id="ha_configure"></span><span id="configuring-high-availability"></span>

## Set up the active-passive pair

1. On both nodes, install the **nginx-ha-keepalived** package from the NGINX Plus repository, in addition to the **nginx-plus** package. The package is available for RHEL-based systems, Debian, Ubuntu, and SLES, as listed in the [technical specifications]({{< ref "/nginx/technical-specs.md" >}}):

    - RHEL and its derivatives:

        ```shell
        sudo dnf install nginx-ha-keepalived
        ```

    - Debian and Ubuntu:

        ```shell
        sudo apt install nginx-ha-keepalived
        ```

    - SLES:

        ```shell
        sudo zypper install nginx-ha-keepalived
        ```

1. On both nodes, run the `nginx-ha-setup` script:

    ```shell
    sudo nginx-ha-setup
    ```

    The script asks for:

    - The IP addresses of the local node and the remote node. One node becomes the primary node, and the other the backup node.
    - One free IP address for the VIP.

1. Check that the VIP is on the primary node, as described in [Show the state of a node](#show-the-state-of-a-node).

### The keepalived configuration

The script writes the configuration of keepalived to **/etc/keepalived/keepalived.conf**. It isn't an NGINX configuration file, so its syntax is different. For example, its directives don't end with semicolons. The script created this file on the primary node of a CentOS 7 pair:

```none
global_defs {
    vrrp_version 3
}

vrrp_script chk_manual_failover {
    script   "/usr/libexec/keepalived/nginx-ha-manual-failover"
    interval 10
    weight   50
}

vrrp_script chk_nginx_service {
    script   "/usr/libexec/keepalived/nginx-ha-check"
    interval 3
    weight   50
}

vrrp_instance VI_1 {
    interface                  eth0
    priority                   101
    virtual_router_id          51
    advert_int                 1
    accept
    garp_master_refresh        5
    garp_master_refresh_repeat 1
    unicast_src_ip             192.168.100.100

    unicast_peer {
        192.168.100.101
    }

    virtual_ipaddress {
        192.168.100.150
    }

    track_script {
        chk_nginx_service
        chk_manual_failover
    }

    notify "/usr/libexec/keepalived/nginx-ha-notify"
}
```

Each node has its own copy of the file. These directives differ between the nodes:

{{< table >}}

| Directive | Primary node | Backup node | Meaning |
|---|---|---|---|
| `priority` | `101` | `100` | The base priority of the node |
| `unicast_src_ip` | `192.168.100.100` | `192.168.100.101` | The address of the node |
| `unicast_peer` | `192.168.100.101` | `192.168.100.100` | The address of the other node |

{{< /table >}}

These directives are the same on both nodes:

- `virtual_router_id`: The ID of the VRRP instance. `51` is a sample value. Each VRRP instance on your network needs its own ID, and each `vrrp_instance` block on a node needs its own name, such as `VI_1`.
- `advert_int`: The interval of the advertisements, in seconds.
- `virtual_ipaddress`: The VIP.
- `track_script`: The health-check scripts that change the priority of the node.
- `notify`: The notification script of the package. keepalived runs it on each state change, and it logs the change to syslog.

For all directives, see the [keepalived configuration synopsis](https://keepalived.readthedocs.io/en/latest/configuration_synopsis.html).

<span id="ha_scripts"></span><span id="using-a-health-check-script-to-control-which-server-is-primary"></span>

### How the health checks choose the primary node

keepalived runs each `vrrp_script` every `interval` seconds. A script that exits with code `0` succeeds, and any other code is a failure. A script that runs longer than its timeout fails too. The timeout is the same as the interval, unless you set `timeout`. To act only on several results in a row, set `rise` and `fall`.

Both scripts have a `weight` of `50`. keepalived adds the weight of each script that succeeds to the base priority of the node:

- `chk_nginx_service` runs the **nginx-ha-check** script of the package every 3 seconds. It succeeds when NGINX Plus is up.
- `chk_manual_failover` runs every 10 seconds. It succeeds unless you start a [manual failover](#force-a-state-change).

When both nodes are healthy, the primary node has priority 201 (101 + 50 + 50), and the backup node has 200. The node with the higher priority holds the VIP:

{{< table >}}

| Situation | Primary node | Backup node | VIP on |
|---|---|---|---|
| Both nodes are healthy | 201 | 200 | Primary node |
| NGINX Plus fails on the primary node | 151 | 200 | Backup node |
| Manual failover on the primary node | 151 | 200 | Backup node |
| keepalived stops on the primary node | Sends priority 0 | 200 | Backup node |

{{< /table >}}

When the primary node recovers, its priority is higher again, and it takes the VIP back.

The **nginx-ha-check** script checks only that NGINX Plus is up. To check more, such as your applications, add scripts of your own.

keepalived has no fencing. If the nodes don't receive each other's advertisements, both nodes take the VIP. The health checks don't prevent this.

---

<span id="ha_status"></span><span id="displaying-node-state"></span>

## Show the state of a node

- To see which node holds the VIP, list the IPv4 addresses of the interface of the VRRP instance, **eth0** in this guide:

    ```shell
    ip -4 -brief address show eth0
    ```

    On the primary node, the output includes the VIP:

    ```none
    eth0             UP             192.168.100.100/24 192.168.100.150/32
    ```

    On the backup node, it doesn't:

    ```none
    eth0             UP             192.168.100.101/24
    ```

- The package records the state of each node in **/var/run/nginx-ha-keepalived.state**:

    ```shell
    cat /var/run/nginx-ha-keepalived.state
    ```

    The output is `STATE=MASTER` on the primary node, and `STATE=BACKUP` on the backup node.

- In version 1.1 and later of the **nginx-ha-keepalived** package, this command writes VRRP statistics to **/tmp/keepalived.stats**, and the configuration and state of keepalived to **/tmp/keepalived.data**:

    ```shell
    sudo service keepalived dump
    ```

    The command sends signals to keepalived. Only `root` can read the files:

    ```shell
    sudo cat /tmp/keepalived.stats
    ```

    The statistics start like this:

    ```none
    VRRP Instance: VI_1
      Advertisements:
        Received: 0
        Sent: 43
      Became master: 1
      Released master: 0
    ```

---

<span id="ha_state"></span><span id="forcing-a-state-change"></span>

## Force a state change

To move the VIP to the backup node, use one of these methods on the primary node:

- Stop keepalived:

    ```shell
    sudo systemctl stop keepalived
    ```

    As it stops, keepalived sends an advertisement with priority 0. The backup node then takes over the VIP within about a second, without waiting for the advertisements to time out. To take the VIP back, start keepalived again:

    ```shell
    sudo systemctl start keepalived
    ```

- In version 1.1 and later of the **nginx-ha-keepalived** package, create the manual failover file:

    ```shell
    sudo touch /var/run/keepalived-manual-failover
    ```

    The `chk_manual_failover` script then fails, and the priority of the primary node drops to 151, below the 200 of the backup node. The backup node discards advertisements with a lower priority, as if the primary node had stopped sending them. It takes over after three advertisement intervals. So the VIP moves within about 13 seconds: up to 10 seconds until the script runs, and about 3 more. To move the VIP back, delete the file:

    ```shell
    sudo rm /var/run/keepalived-manual-failover
    ```

    Delete the file when you no longer need it. Left on a node, it keeps the priority of that node low, as described in [The VIP doesn't move when NGINX Plus stops](#the-vip-doesnt-move-when-nginx-plus-stops).

---

<span id="ha_addresses"></span><span id="adding-more-virtual-ip-addresses"></span>

## Add more virtual IP addresses

The `nginx-ha-setup` script makes one IP address highly available. To add more:

1. On both nodes, add each address to the `virtual_ipaddress` block in **/etc/keepalived/keepalived.conf**:

    ```none
    virtual_ipaddress {
        192.168.100.150
        192.168.100.200
    }
    ```

    The lines take the syntax of the `ip address` command, so you can add options such as `dev eth1`.

1. Reload keepalived on both nodes:

    ```shell
    sudo systemctl reload keepalived
    ```

1. Check that the primary node holds all the VIPs:

    ```shell
    ip -4 -brief address show eth0
    ```

---

<span id="ha_addresses_ipv6"></span><span id="dual-stack-configuration-of-ipv4-and-ipv6"></span>

## Make IPv4 and IPv6 addresses highly available

A VRRP instance manages addresses of one family only ([RFC 5798, section 3](https://datatracker.ietf.org/doc/html/rfc5798#section-3)). Since keepalived 1.2.20 and version 1.1 of the **nginx-ha-keepalived** package, keepalived ignores addresses of the other family in a `virtual_ipaddress` block. It logs a message such as `address family must match VRRP instance [2001:db8:100::1] - ignoring`.

To make addresses of both families highly available, use one of these methods:

- Add the addresses of the other family to a `virtual_ipaddress_excluded` block:

    ```none
    vrrp_instance VI_1 {
        # ...
        virtual_ipaddress {
            192.168.100.150
        }

        virtual_ipaddress_excluded {
            2001:db8:100::1
        }
        # ...
    }
    ```

    The advertisements don't include these addresses, but keepalived still adds and removes them on each state change.

- Add a second VRRP instance for the IPv6 addresses. On the primary node, it's:

    ```none
    vrrp_instance VI_2 {
        interface         eth0
        priority          101
        virtual_router_id 51
        advert_int        1
        accept
        unicast_src_ip    2001:db8:100::3

        unicast_peer {
            2001:db8:100::2
        }

        virtual_ipaddress {
            2001:db8:100::1
        }

        track_script {
            chk_nginx_service
            chk_manual_failover
        }

        notify "/usr/libexec/keepalived/nginx-ha-notify"
    }
    ```

    On the backup node, swap the two addresses and use priority `100`. The IPv4 and IPv6 instances are independent of each other, so they can use the same `virtual_router_id`.

---

<span id="ha_misc"></span><span id="keeping-f5-nginx-plus-configuration-files-in-sync"></span>

## Keep the NGINX Plus configuration in sync

Both nodes must have the configuration of the services that you make highly available. For a way to keep the NGINX Plus configuration of the nodes the same, see [Synchronizing NGINX Configuration in a Cluster]({{< ref "/nginx/admin-guide/high-availability/configuration-sharing.md" >}}).

---

<span id="ha_troubleshoot"></span><span id="troubleshooting-keepalived-and-vrrp"></span>

## Troubleshooting

keepalived logs to syslog. To read its messages on a system with systemd:

```shell
sudo journalctl -u keepalived
```

Systems that write syslog to files have the messages in **/var/log/messages** (RHEL and SLES) or **/var/log/syslog** (Debian and Ubuntu).

In these messages of keepalived 2.3, a node starts as backup, adds the weights of both scripts, and becomes the primary node:

```none
Nov 23 08:57:15 node-1 Keepalived_vrrp[239]: (VI_1) Entering BACKUP STATE (init)
Nov 23 08:57:15 node-1 Keepalived_vrrp[239]: VRRP_Script(chk_manual_failover) succeeded
Nov 23 08:57:15 node-1 Keepalived_vrrp[239]: (VI_1) Changing effective priority from 101 to 151
Nov 23 08:57:15 node-1 Keepalived_vrrp[239]: VRRP_Script(chk_nginx_service) succeeded
Nov 23 08:57:15 node-1 Keepalived_vrrp[239]: (VI_1) Changing effective priority from 151 to 201
Nov 23 08:57:18 node-1 Keepalived_vrrp[239]: (VI_1) Receive advertisement timeout
Nov 23 08:57:18 node-1 Keepalived_vrrp[239]: (VI_1) Entering MASTER STATE
Nov 23 08:57:18 node-1 Keepalived_vrrp[239]: (VI_1) setting VIPs.
Nov 23 08:57:18 node-1 Keepalived_vrrp[239]: (VI_1) Sending/queueing gratuitous ARPs on eth0 for 192.168.100.150
```

Older versions of keepalived word some messages differently. The **nginx-ha-notify** script of the package also logs each transition, for example `Transition to state 'MASTER ' on VRRP instance 'VI_1 '.`

### Capture VRRP advertisements

If the log doesn't explain a problem, capture the VRRP advertisements with `tcpdump`. To see only the traffic with the other node, add its address from the `unicast_peer` block:

```shell
sudo tcpdump -vvv -ni eth0 vrrp and host 192.168.100.101
```

Use `vrrp` or `ip proto 112` as the filter. `vrrp` is itself a filter keyword, so `proto vrrp` fails with `can't parse filter expression: syntax error`.

On the primary node, the output shows its advertisements:

```none
tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
08:55:19.377184 IP (tos 0xc0, ttl 255, id 28, offset 0, flags [none], proto VRRP (112), length 32)
    192.168.100.100 > 192.168.100.101: VRRPv3, Advertisement, vrid 51, prio 201, intvl 100cs, length 12, addrs: 192.168.100.150
```

These fields help with debugging:

- `vrid`: The `virtual_router_id` of the instance.
- `prio`: The priority of the sender, with the weights of its scripts.
- `intvl`: The advertisement interval, `advert_int`, in centiseconds.
- `addrs`: The VIPs.

VRRP version 3 has no authentication, so the output has no authentication fields ([RFC 5798, section 9](https://datatracker.ietf.org/doc/html/rfc5798#section-9)).

---

### Both nodes hold the VIP

**Symptom**: `ip -4 -brief address show eth0` shows the VIP on both nodes, and the logs of both nodes have `Entering MASTER STATE`.

**Cause**: The nodes don't receive each other's advertisements. For example, a firewall drops IP protocol 112, or `unicast_peer` has a wrong address. keepalived has no fencing, so each node assumes that the other one failed.

**Fix**: Allow IP protocol 112 between the nodes, and check the `unicast_src_ip` and `unicast_peer` addresses on both nodes. To see which advertisements arrive, [capture them](#capture-vrrp-advertisements). When the nodes see each other again, the node with the lower priority becomes the backup node.

---

### The VIP doesn't move when NGINX Plus stops

**Symptom**: NGINX Plus stops on the primary node, but the VIP stays there.

**Cause**: The backup node doesn't have a higher priority. For example, a manual failover file left on the backup node lowers its priority to 150. That's below the 151 of the failed primary node.

**Fix**: On the backup node, delete **/var/run/keepalived-manual-failover**. Then check that the log of the primary node shows `VRRP_Script(chk_nginx_service) failed`.

---

### keepalived logs a SECURITY VIOLATION warning

**Symptom**: The log has the line `SECURITY VIOLATION - scripts are being executed but script_security not enabled.`

**Cause**: keepalived runs the health-check scripts as `root`, and the configuration doesn't turn on script security. The scripts still run.

**Fix**: No fix is needed. To run the scripts in secure mode, add `enable_script_security` and `script_user root` to the `global_defs` block on both nodes, and reload keepalived. keepalived then refuses scripts that users other than `root` can change, and logs `Disabling track script ... due to insecure` for them.

---

<span id="ha_examples"></span><span id="additional-configuration-examples"></span>

## References

The **nginx-ha-keepalived** package has more configuration examples in the **/usr/share/doc/nginx-ha-keepalived** directory.

For more information, see:

- [Configuring Active-Active High Availability and Additional Passive Nodes with keepalived]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived-nodes.md" >}})
- [Synchronizing NGINX Configuration in a Cluster]({{< ref "/nginx/admin-guide/high-availability/configuration-sharing.md" >}})
- [Keepalived configuration synopsis](https://keepalived.readthedocs.io/en/latest/configuration_synopsis.html)
- [RFC 5798: Virtual Router Redundancy Protocol (VRRP) Version 3](https://datatracker.ietf.org/doc/html/rfc5798)
