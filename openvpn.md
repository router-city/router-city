# OpenVPN Configuration

Here is a basic configuration for connecting two peers via [OpenVPN](https://openvpn.net/) software. Large portions of this guide are inspired by or copied from [dn42's documentation](https://dn42.net/howto/openvpn).

## Installation

Assuming a Debian installation with a non-root, sudo user. First, install `openvpn` on both peers (which I will call peer0 and peer1):

```
$ sudo apt-get install openvpn
```

Now we will create a config file on peer0:

```
$ sudo nano /etc/openvpn/router-city-peer0-peer1.config
```

Use the following sample config and customize it with your info as outlined below:

* Replace `<PEER_NAME>` with a self chosen name to identify this peer.
* Replace `<REMOTE_PEER_NAME>` with a self chosen name to identify the remote peer.
* Replace `<PROTO>` with either `udp` or `udp6`, depending if you reach your remote peer via clearnet with IPv4 or IPv6. (This is provided by your remote peer)
* Replace `<REMOTE_HOST>` with the public IP address of your remote peer. (This is provided by your remote peer)
* Replace `<REMOTE_PORT>` with the port number, where your remote peer's `openvpn` daemon listen for traffic. (This is provided by your remote peer)
* Replace `<LOCAL_HOST>` with the IP address on your local peer that you will bind to. This is usually a public IP for a server or a LAN address for a machine on a home network.
* Replace `<LOCAL_PORT>` with the port number, where your local peer's `openvpn` daemon listen for traffic.
* Replace `<INTERFACE_NAME>` with a self chosen name, this will be the name of your network interface (tun device) for this peering. You can use something cool like `cty00`.
* Replace `<LOCAL_GATEWAY_IP>` with your own router.city IPv4 address of your peer, picked from your [allocation](README.md/#current-ipv4-allocations).
* Replace `<REMOTE_GATEWAY_IP>` with the router.city IPv4 address of your remote peer. (This is provided by your remote peer)
* Replace `<LOCAL_GATEWAY_IPV6>` with your own router.city IPv6 address of your peer, picked from your [allocation](README.md/#current-ipv6-allocations). **NOTE:** Your IPv6 address and your peer's IPV6 address need to be on the same subnet. This should be handled already by our network block but may cause problems if this is changed in the future.
* Replace `<REMOTE_GATEWAY_IPV6>` with the router.city IPv6 address of your remote peer. (This is provided by your remote peer)


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
```

Now we need to create a secret key that is shared by both of the peers. This should only be done _ONCE_ and copied from one peer to the other:

```
$ sudo openvpn --genkey --secret /etc/openvpn/router-city-<PEER_NAME>-<REMOTE_PEER_NAME>.key
```

Now that we have peer0 set up, copy the key to peer1 and fill out the `/etc/openvpn/router-city-peer0-peer1.config` on peer1 using the proper info for this peer. Values like `<REMOTE_HOST>` and `<LOCAL_HOST>` or `<LOCAL_GATEWAY_IP>` and `<REMOTE_GATEWAY_IP>` will be flipped (as you would expect).

Before starting up, make sure that the port specified in `<LOCAL_PORT>` is opened in each machine's firewall and forwarded if necessary.

When done, start `openvpn` in the background on each peer, or the foreground if you want to verify things started up okay.

```
$ sudo nohup openvpn /etc/openvpn/router-city-<PEER_NAME>-<REMOTE_PEER_NAME>.conf &
```

When done, check the interfaces on each node and do a quick ping test if you'd like.

## Sample Testing

### peer0

```
$ ip a show cty00
218: cty00: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 100
    link/none
    inet 172.24.0.1 peer 172.24.0.2/32 scope global cty00
       valid_lft forever preferred_lft forever
    inet6 2001:db8:dead:beef:cafe:f00d:0:1/64 scope global
       valid_lft forever preferred_lft forever

$ ping -c4 172.24.0.2
PING 172.24.0.2 (172.24.0.2) 56(84) bytes of data.
64 bytes from 172.24.0.2: icmp_seq=1 ttl=64 time=78.6 ms
64 bytes from 172.24.0.2: icmp_seq=2 ttl=64 time=77.4 ms
64 bytes from 172.24.0.2: icmp_seq=3 ttl=64 time=79.0 ms
64 bytes from 172.24.0.2: icmp_seq=4 ttl=64 time=77.9 ms

--- 172.24.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 77.491/78.299/79.075/0.677 ms

$ ping6 -c4  2001:db8:dead:beef:cafe:f00d:0:2
PING 2001:db8:dead:beef:cafe:f00d:0:2(2001:db8:dead:beef:cafe:f00d:0:2) 56 data bytes
64 bytes from 2001:db8:dead:beef:cafe:f00d:0:2: icmp_seq=1 ttl=64 time=77.8 ms
64 bytes from 2001:db8:dead:beef:cafe:f00d:0:2: icmp_seq=2 ttl=64 time=78.1 ms
64 bytes from 2001:db8:dead:beef:cafe:f00d:0:2: icmp_seq=3 ttl=64 time=79.2 ms
64 bytes from 2001:db8:dead:beef:cafe:f00d:0:2: icmp_seq=4 ttl=64 time=77.6 ms

--- 2001:db8:dead:beef:cafe:f00d:0:2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 77.638/78.224/79.260/0.712 ms
```

### peer1

```
$ ip a show cty00
13: cty00: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 100
    link/none
    inet 172.24.0.2 peer 172.24.0.1/32 scope global cty00
       valid_lft forever preferred_lft forever
    inet6 2001:db8:dead:beef:cafe:f00d:0:2/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::20c7:7ddf:3e3d:7778/64 scope link flags 800
       valid_lft forever preferred_lft forever

$ ping -c4 172.24.0.1
PING 172.24.0.1 (172.24.0.1) 56(84) bytes of data.
64 bytes from 172.24.0.1: icmp_seq=1 ttl=64 time=77.1 ms
64 bytes from 172.24.0.1: icmp_seq=2 ttl=64 time=78.4 ms
64 bytes from 172.24.0.1: icmp_seq=3 ttl=64 time=79.2 ms
64 bytes from 172.24.0.1: icmp_seq=4 ttl=64 time=78.0 ms

--- 172.24.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 77.194/78.232/79.219/0.802 ms

$ ping6 -c4  2001:db8:dead:beef:cafe:f00d:0:1
PING 2001:db8:dead:beef:cafe:f00d:0:1(2001:db8:dead:beef:cafe:f00d:0:1) 56 data bytes
64 bytes from 2001:db8:dead:beef:cafe:f00d:0:1: icmp_seq=1 ttl=64 time=78.7 ms
64 bytes from 2001:db8:dead:beef:cafe:f00d:0:1: icmp_seq=2 ttl=64 time=77.5 ms
64 bytes from 2001:db8:dead:beef:cafe:f00d:0:1: icmp_seq=3 ttl=64 time=78.9 ms
64 bytes from 2001:db8:dead:beef:cafe:f00d:0:1: icmp_seq=4 ttl=64 time=77.1 ms

--- 2001:db8:dead:beef:cafe:f00d:0:1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 77.150/78.099/78.968/0.793 ms
```