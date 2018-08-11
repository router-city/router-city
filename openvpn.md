# OpenVPN Configuration

Here is a basic configuration for connecting two peers via [OpenVPN](https://openvpn.net/) software. Large portions of this guide are inspired by or copied from [dn42's documentation](https://dn42.net/howto/openvpn).

## Installation

Assuming a Debian installation with a non-root, sudo user. First, install `openvpn` on both peers (we are assuming you are running a local peer and a friend has a remote peer):

```
$ sudo apt-get install openvpn
```

Now we will create a config file on peer0:

```
$ sudo nano /etc/openvpn/router-city-peer0-peer1.config
```

Use the following sample config and customize it with your info as outlined below:

* Replace `<PEER_NAME>` with a self chosen name to identify this local peer.
* Replace `<REMOTE_PEER_NAME>` with a self chosen name to identify the remote peer.
* Replace `<PROTO>` with either `udp` or `udp6`, depending if you reach your remote peer via clearnet with IPv4 or IPv6. (This is provided by your remote peer)
* Replace `<REMOTE_HOST>` with the public IP address of your remote peer. (This is provided by your remote peer)
* Replace `<REMOTE_PORT>` with the port number, where your remote peer's `openvpn` daemon listen for traffic. (This is provided by your remote peer)
* Replace `<LOCAL_HOST>` with the IP address on your local peer that you will bind to. This is usually a public IP for a server or a LAN address for a machine on a home network (behind NAT).
* Replace `<LOCAL_PORT>` with the port number, where your local peer's `openvpn` daemon listen for traffic.
* Replace `<INTERFACE_NAME>` with a self chosen name, this will be the name of your network interface (tun device) for this peering. You can use something cool like `cty00` or the classic `tun0`.
* Replace `<LOCAL_GATEWAY_IP>` with a private IPv4 address of your peer, agreed upon by remote peer (like `192.168.254.1`)
* Replace `<REMOTE_GATEWAY_IP>` with the private IPv4 address of your remote peer. (This is provided by your remote peer, something in the same subnet as your `<LOCAL_GATEWAY_IP>` like `192.168.254.2`)
* Replace `<LOCAL_GATEWAY_IPV6>` with a private IPv6 address of your peer, agreed upon by your remote peer. The link-local address space works great here (and include the cidr, like `fe80:1::1/64`)
* Replace `<REMOTE_GATEWAY_IPV6>` with the private IPv6 address of your remote peer. (This is provided by your remote peer, something like `fe80:1::2`, no cidr here)


```
#/etc/openvpn/router-city-<PEER_NAME>-<REMOTE_PEER_NAME>.conf
proto       <PROTO>
mode        p2p
remote      <REMOTE_HOST>
rport       <REMOTE_PORT>
local       <LOCAL_HOST>
lport       <LOCAL_PORT>
dev-type    tun
tun-ipv6
resolv-retry infinite
dev         <INTERFACE_NAME>
comp-lzo
persist-key
persist-tun
cipher aes-256-cbc
ifconfig-ipv6 <LOCAL_GATEWAY_IPV6> <REMOTE_GATEWAY_IPV6>
ifconfig <LOCAL_GATEWAY_IP>  <REMOTE_GATEWAY_IP>
secret /etc/openvpn/router-city-<PEER_NAME>-<REMOTE_PEER_NAME>.key

# The secret can also be included inline with the config by
#   wrapping it in <secret></secret> tags.
# <secret>
# ... Key File contents go here ...
# </secret>

```

Now we need to create a secret key that is shared by both of the peers. This should only be done _ONCE_ and copied from one peer to the other:

```
$ sudo openvpn --genkey --secret /etc/openvpn/router-city-<PEER_NAME>-<REMOTE_PEER_NAME>.key
```

Now that we have our local peer set up, copy the key to the remote peer and fill out the `/etc/openvpn/router-city-<PEER_NAME>-<REMOTE_PEER_NAME>.config` on the remote peer using the proper info for this peer. Values like `<REMOTE_HOST>` and `<LOCAL_HOST>` or `<LOCAL_GATEWAY_IP>` and `<REMOTE_GATEWAY_IP>` will be flipped (as you would expect).

Before starting up, make sure that the port specified in `<LOCAL_PORT>` is opened in each machine's firewall and forwarded if necessary.

When done, start `openvpn` in the background on each peer, or the foreground if you want to verify things started up okay.

```
$ sudo nohup openvpn /etc/openvpn/router-city-<PEER_NAME>-<REMOTE_PEER_NAME>.conf &
```

When done, check the interfaces on each node and do a quick ping test if you'd like.

## Sample Testing

```
$  ip a show cty02
239: cty02: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 100
    link/none
    inet 192.168.254.1 peer 192.168.254.2/32 scope global cty02
       valid_lft forever preferred_lft forever
    inet6 fe80:1::1/64 scope link
       valid_lft forever preferred_lft forever

$ ping -c4 -I cty02 192.168.254.2
PING 192.168.254.2 (192.168.254.2) from 192.168.254.1 cty02: 56(84) bytes of data.
64 bytes from 192.168.254.2: icmp_seq=1 ttl=64 time=91.4 ms
64 bytes from 192.168.254.2: icmp_seq=2 ttl=64 time=91.1 ms
64 bytes from 192.168.254.2: icmp_seq=3 ttl=64 time=91.2 ms
64 bytes from 192.168.254.2: icmp_seq=4 ttl=64 time=91.3 ms

--- 192.168.254.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 91.180/91.292/91.426/0.315 ms

$ ping6 -c4 -I cty02 fe80:1::2
PING fe80:1::2(fe80:1::2) from fe80:1::1 cty02: 56 data bytes
64 bytes from fe80:1::2: icmp_seq=1 ttl=64 time=91.1 ms
64 bytes from fe80:1::2: icmp_seq=2 ttl=64 time=91.2 ms
64 bytes from fe80:1::2: icmp_seq=3 ttl=64 time=91.9 ms
64 bytes from fe80:1::2: icmp_seq=4 ttl=64 time=90.7 ms

--- fe80:1::2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 90.768/91.264/91.901/0.408 ms
```