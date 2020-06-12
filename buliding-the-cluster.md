---
title: "Building The Cluster: First Steps"
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
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
iptables -A INPUT -i eth0 -j ACCEPT
iptables -A INPUT -i eth1 -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -j ACCEPT
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
