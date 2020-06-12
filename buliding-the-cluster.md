---
title: "Building The Cluster: First Steps"
published: true
tags:
- kubernetes
- cluster
- homelab
- linux
---

> This post is #2 in a series which will demonstrate
> creating a Kubernetes cluster using an on-prem
> environment. If you're new to this series, please
> begin with
> [Part 1: The New Cluster](https://dev.to/mikeyglitz/the-new-custer-12cf)

# Installing the OS

I chose to use the [Debian](https://debian.org) distribution for this
cluster. I used the
[Bittorrent download](https://www.debian.org/CD/torrent-cd/)
to obtain the DVD image.

Make sure to compare the downloaded ISO file against a checksum.

I burned the install media using [Rufus](https://rufus.ie/) and a
USB pen-drive.

I chose the basic installation options for each node.
The only deviation I made from the default options was to not install
a Swap partition. [Kubernetes and Docker installations do not play
nice with Swap partitions](https://serverfault.com/questions/881517/why-disable-swap-on-kubernetes).

## Network Installation

There will be two networks in the cluster:

[![Cluster Layout](https://drive.google.com/thumbnail?id=1dHOeEqZzHaXSzrT4xgDbA1spGr9_5uTm)](https://app.diagrams.net/?lightbox=1&highlight=0000ff&layers=1&nav=1&title=Home%20Cluster#Uhttps%3A%2F%2Fdrive.google.com%2Fuc%3Fid%3D1dHOeEqZzHaXSzrT4xgDbA1spGr9_5uTm%26export%3Ddownload)

- 192.168.0.0/24 - Used for internal LAN
- 172.16.0.0/29 - Cluster network

### Interface Installation

The master node uses an intel wireless adapter for wifi.
This adapter uses the **non-free** Debian apt repository.
The **non-free** repository can be added by adding the following
line to `/etc/apt/sources.list`

```text
deb http://http.us.debian.org/debian stable main contrib non-free
```

The local apt cache needs to be refreshed

```bash
sudo apt update
sudo apt install firmware-iwlwifi
```

The network interface for the wireless adapter needs to be configured.
The wireless network is configured with WPA security. Debian has a
package for accessing WPA-secured networks, `wpasupplicant`.

```bash
sudo apt install wpasupplicant
```

`wpasupplicant` needs to be configured with the network SSID and
passkey

```bash
su -l -c "wpa_passphrase myssid my_very_secret_passphrase > /etc/wpa_supplicant/wpa_supplicant.conf"
```

The `wpa_passphrase` command will write the SSID and a passkey hash
to a file, `/etc/wpa_supplicant/wpa_supplicant.conf`.

```text
    ssid="myssid"
    #psk="my_very_secret_passphrase"
    psk="ccb290fd4fe6b22935cbae31449e050edd02ad44627b16ce0151668f5f53c01b"
```

The network interface configuration also needs to be updated at
`/etc/network/interface`.

```text
auto enx70886b81ddea
iface enx70886b81ddea inet static
        address 172.16.0.1
        netmask 255.255.255.248
        network 172.16.0.0
        broadcast 172.16.0.7

allow-hotplug wlp2s0
iface wlp2s0 inet static
        address 192.168.0.120
        netmask 255.255.255.0
        network 192.168.0.0
        gateway 192.168.0.1
        broadcast 192.168.0.255
        wpa-ssid myssid
        wpa-psk ccb290fd4fe6b22935cbae31449e050edd02ad44627b16ce0151668f5f53c01b
```

The interface `wpl2s0` configures wifi. The values at `wpa-ssid`
and `wpa-psk` are obtained from the values of `ssid` and `psk`
from `/etc/wpa_supplicant/wpa_supplicant.conf`.

The block beginning with `enx70886b81ddea` configures the interface
to the cluster network. The interface at `enx70886b81ddea` is a
USB to Ethernet adapter.

```text
iface enx70886b81ddea inet static
        address 172.16.0.1
        netmask 255.255.255.248
        network 172.16.0.0
        broadcast 172.16.0.7
```

Sets the `enx70886b81ddea` interface to listen on 172.16.0.1.
The network mask is `255.255.255.248` which is a 7-device address
space. The first and last addresses are reserved for the network
and broadcast addresses respectively.

### NAT Routing

The next part of the network set up is setting up network-address
translation (NAT) so that the cluster nodes can reach the Internet.

> ⚠ With Debian Buster, `iptables` has been changed to `nftables`

`iptables` and `iptables-persistent` need to be installed.

```bash
sudo apt install iptables iptables-persistent
sudo update-alternatives --set iptables=/usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables=/usr/sbin/ip6tables-legacy
```

NAT rules will need to be set using `iptables`.

```bash
# set up ip forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# NAT rules
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE --random
iptables -A FORWARD -i eth0 -o wlp2s0 -j ACCEPT
iptables -A FORWARD -i wlp2s0 -o eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -j DROP
```

The new rules will be cleared upon system reboot. It's important to
persist them.

```bash
iptables-save > /etc/iptables.rules
```

Create the file at `/etc/network/if-pre-up.d/firewall` if it doesn't exist

```bash
#!/bin/bash
/sbin/iptables-restore < /etc/iptables.rules
```

Add execute permissions to `/etc/network/if-pre-up.d/firewall`.

```bash
chmod +x /etc/network/if-pre-up.d/firewall
```

### DHCP


DHCP is a network protocol which is used to 
dynamically assign IP addresses to hosts that are
connected to the network. During the handshake phase
of establishing the network connection, the host
requests an IP address and is assigned one with a
DHCP lease.

[isc-dhcp-server](https://www.isc.org/dhcp/) is used
to perform DHCP services on the `172.16.0.0/29`
network. isc-dhcp-server is installed with the
following command:

```bash
sudo apt-get install -y isc-dhcp-server
```

`/etc/dhcp/dhcpd.conf` is used to configure the
isc-dhcp-server. The following configuration is used
to configure `/etc/dhcp/dhcpd.conf`:

```
ddns-update-style none;
default-lease-time 600;
max-lease-time 7200;
#ping true;
# option domain-name-servers 172.16.0.1;
# option domain-name "haus.net";
authorative;
log-facility local7;

subnet 172.16.0.0 netmask 255.255.255.248 {
        range 172.16.0.1 172.16.0.6;
        option subnet-mask 255.255.255.248;
        option domain-name-servers 172.16.0.1;
        option routers 172.16.0.1;
        get-lease-hostnames true;
        use-host-decl-names true;
        default-lease-time 600;
        max-lease-time 7200;
}
```

The interface isc-dhcp-server listens on needs to be
configured to listen on the `enx70886b81ddea`
interface. The `/etc/default/isc-dhcp-server` file
is where the interface can get set

```
INTERFACESv4="enx70886b81ddea"
```

Restart the services and the networks on the master
node will be configured.

```bash
systemctl restart isc-dhcp-server
ifup wsl2p0             # Starts the wifi network
ifup enx70886b81ddea    # Starts the Cluster network
```

### Compute Node Setup

Setup for the compute node is easier than the master
node. Edit `/etc/network/interfaces` with the
following information:

```text
allow-hotplug eth0
iface eth0 inet dhcp
```

This setup will configure each of the compute nodes
to use DHCP from the master node that was set up
earlier.

> ⚠ The ethernet device is not always going to be
> named `eth0`. Sometimes you'll have to check
> the device name with the `ip a` command

Hosts can be located on the 172.16.0.0/29 network by
using the `nmap` utility.

```bash
nmap -sn 172.16.0.0/29
```

## Cluster Setup

To set up the cluster, [K3S](https://k3s.io) is
going to be used instead of
[Kubernetes](https://kubernetes.io) due to the limited
resources that the cluster is using.
K3S is a lightweight flavor of Kubernetes developed
by [Rancher](https://rancher.io).

There is a convenience utility for K3S called
[k3sup](https://github.com/alexellis/k3sup).

> ⚠ I had issues setting up `k3sup`. The utility
> would not run the proper SSH commands due to
> my terminal not being connected to a TTY device.
> This can be fixed with
> [passwordless SSH](https://www.tecmint.com/ssh-passwordless-login-using-ssh-keygen-in-5-easy-steps/)
> and adding your user to sudoers without a password
> using the `visudo` command
>
> ```text
> <user>  ALL=(ALL) NOPASSWORD: ALL
> ```
>
> **I do not normally recommend doing this as
> passwordless sudo is a security vulnerability**

> ⚠ By default k3s will install containers using
> [containerd](https://containerd.io). Some of the
> operators I'll be using later will use
> [Docker Manifest](https://docs.docker.com/engine/reference/commandline/manifest/).
> Docker will have to be installed as a CNI.

### Installing Docker

Docker provides a convenience script which will
perform an express installation. Docker will need
to be installed on all cluster nodes.

```bash
curl -fsSL https://get.docker.com | sudo sh -
```

### Installing k3s

With Docker installed, it's time to set up the
Kubernetes cluster using k3s using `k3sup`.

On the master node, run:

```bash
k3sup install --user manager --ip <external-ip> --k3s-extra-args '--docker'
```

On the compute node, run:

```bash
k3sup join --ip <node-ip> --server-ip <external-ip> --user manager --k3s-extra-args '--docker'
```

With these commands, your Kubernetes cluster with
k3s should be active. You can begin to experiment with
Kubernetes in your home lab!

# Resources

- [How to use WiFi - Debian](https://wiki.debian.org/WiFi/HowToUse)
- [ISC-dhcp-server](https://help.ubuntu.com/community/isc-dhcp-server)
- [iptables -- how to set up a server as a router with NAT](https://serverfault.com/questions/564866/how-to-set-up-linux-server-as-a-router-with-nat)
- [How to save iptables rules in debian](https://websistent.com/how-to-save-iptables-rules-in-debian/)
- [k3s installation requirements](https://rancher.com/docs/k3s/latest/en/installation/installation-requirements/)
- [k3sup](https://github.com/alexellis/k3sup)
- [passwordless ssh](https://www.tecmint.com/ssh-passwordless-login-using-ssh-keygen-in-5-easy-steps/)
- [Passwordless Sudo](https://serverfault.com/questions/160581/how-to-setup-passwordless-sudo-on-linux)
- [Will it cluster? k3s on your raspberry pi](https://blog.alexellis.io/test-drive-k3s-on-raspberry-pi/)

> ℹ **Note:** Stay tuned for a guide on how to set up Kubernetes resources using Terraform
