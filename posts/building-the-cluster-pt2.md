---
title: 'Building The Cluster: Revised Edition'
description: ''
series: Homelab Installation
canonical_url: null
cover_image: ''
tags:
  - kubernetes
  - cluster
  - homelab
  - linux
published: true
id: 358064
date: '2020-06-18T15:10:40Z'
---

After putting in the work to
[set up my cluster in Debian initially](https://dev.to/mikeyglitz/building-the-cluster-first-steps-153o),
I came upon networking issues that I didn't know how to fix or why they were happening.
I ultimately ended up migrating my cluster to [Ubuntu](https://ubuntu.com).

The image that I'll be using for this guide is the
[Ubuntu Server 20.04](https://ubuntu.com/download/server)
image.

> ℹ 20.04 is a Long-Term Support (LTS) release which means that
> updates will and security fixes will be supported for the next
> 5 years.

As with before, I used [Rufus](https://rufus.ie) to burn the image to a USB drive.

Installation was created with the following partition map

```text
/dev/sda1
    vg-ubuntu
        lv-ubuntu-tmp  - /tmp - 2GB
        lv-ubuntu-home - /home - 2GB
        lv-ubuntu-root - / - Use all remaining space
```

# Network Configuration

Network configuration in Ubuntu was quite different than with Debian.
Debian uses `/etc/network/interfaces` to configure network interfaces.
Ubuntu uses a utility called `netplan` which manages network device configuration.
Ubuntu also includes the Intel WiFi divers as part of its distribution -- no need to
install `firmware-iwlwifi`.

`netplan` uses YAML-based configuration for network configuration.
The configuration can be found at `/etc/netplan/00-installer-config.yaml`.
The configuration used for the gateway node is below.

```yml
network:
  ethernets:
    eth0:
      addresses: [172.16.0.1/24]
      nameservers:
        addresses: [172.16.0.1]
  version: 2
  wifis:
    wlp2s0:
      access-points:
        "SSID":
           password: "wifi-password"
      addresses: [192.168.0.120/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [192.168.0.120]
      dhcp4: false
```

This configuration establishes 2 interfaces: wlp2s0 which will connect
to the external network (in this case, my home WiFi), and eth0 which
will connect to the cluster internal network.

The network configuration can be applied using `netplan apply`.

The compute node network configuration is similar

```yml
# This is the network config written by 'subiquity'
network:
  ethernets:
    eth0:
      dhcp4: true
  version: 2
```

This configuration uses the DHCP server which is set up on the gateway node
to receive an IP address lease.

## DHCP

Fortunately, I'm still able to use `isc-dhcp-server` with Ubuntu.
[I was able to follow the same instructions as before](https://dev.to/mikeyglitz/building-the-cluster-first-steps-153o#dhcp)

## NAT Routing

I learned some lessons with nat-routing from the previous cluster set up.

First set `iptables` to use `iptables-legacy`

```bash
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```

Enable ipv4 forwarding

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
sysctl net.ipv4.ip_forward=1
```

Set the NAT forwarding rules

```bash
# Enable masquerade on eth1 to rewrite the source address on outgoing packets. If you truly want symmetric NAT, you'll need the --random at the end:
iptables -t nat -A POSTROUTING -o wlp2s0 -j MASQUERADE --random
# Allow traffic from internal to external
iptables -A FORWARD -i eth0 -o wlp2s0 -j ACCEPT
# Allow returning traffic from external to internal
iptables -A FORWARD -i wlp2s0 -o eth0 -m conntrack --ctstate RELATED, ESTABLISHED -j ACCEPT
```

Install `iptables-persistent`

```bash
apt-get install iptables-persistent
```

Save the IP rules

```bash
iptables-save > /etc/iptables/rules.v4
```

# Cluster Setup

Setting up the cluster should be the
[same procedure as before](https://dev.to/mikeyglitz/building-the-cluster-first-steps-153o#cluster-setup).

# Network Share

The cluster has an external hard drive attached to it.
The external hard drive will have to be made available to the Kubernetes cluster as a network share.
The external hard drive can be exposed to the network by running a nfs server

Install the nfs server

```bash
sudo apt install -y nfs-kernel-server
```

Expose the external hard drive by updating `/etc/exports`

```text
/mnt/external 172.16.0.0/29(rw,sync,no_subtree_check,no_root_squash)
```

Restart the NFS server and update the shares

```bash
sudo exportfs -av
sudo systemctl restart nfs-kernel-server
```
