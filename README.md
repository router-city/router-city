# router.city

router.city is a darknet project making use of BGP to experiment with routing. BGP (or Border Gateway Protocol) is a dominant protocol on the Internet, used for connecting multiple networks together.

Several projects like this have existed before such as [dn42](https://dn42.net/Home) and [AnoNet](http://wiki.ucis.nl/Anonet).

Chat with us on Matrix! - [#bgp:phillymesh.net](https://matrix.to/#/#bgp:phillymesh.net)

## Addressing

router.city will utilize both IPv4 and IPv6 addressing within the network.

IPv6 addresses will be assigned from the `2001:db8:dead:beef:/64` block, which is reserved space for documentation and source code examples. This range does not conflict with dn42, ChaosVPN, Friefunk, Yggdrasil, or cjdns.

IPv4 addresses will be assigned from the `172.24.0.0/7` block, which is reserved for private network space. This range does not conflict with dn42, ChaosVPN, or Friefunk.

### Current IPv4 Allocations

| CIDR          | Range                      | User          |
| ------------- | -------------------------- | ------------- |
| 172.24.0.0/24 | 172.24.0.0 - 172.24.0.255  | [Famicoman](https://github.com/Famicoman)|
| 172.24.1.0/24 | 172.24.1.0 - 172.24.1.255  | [mikenabhan](https://github.com/mikenabhan)|

### Current IPv6 Allocations

| CIDR                                 | Range                                                                  | User          |
| ------------------------------------ | ---------------------------------------------------------------------- | ------------- |
| 2001:db8:dead:beef:cafe:f00d:0::/112 | 2001:db8:dead:beef:cafe:f00d:0:0 - 2001:db8:dead:beef:cafe:f00d:0:ffff | [Famicoman](https://github.com/Famicoman)|
| 2001:db8:dead:beef:cafe:babe:0::/112 | 2001:db8:dead:beef:cafe:babe:0:0 - 2001:db8:dead:beef:cafe:babe:0:ffff | [mikenabhan](https://github.com/mikenabhan)|

These ranges may be changed at any time if they are found to conflict with another range or be unusable for any reason.

## Autonomous System Numbers

Each organization on the router.city network with control of an address block will need a unique Autonomous System Number (ASN) to identify the organization's network.

AS numbers will be assigned from the `64496 - 64511` range, which is reserved space for documentation and source code examples.

| ASN   | User          |
| ----- | ------------- |
| 64496 | [Famicoman](https://github.com/Famicoman)|
| 64497 | [mikenabhan](https://github.com/mikenabhan)|

This range may change at any time if it is found to conflict with another range or be unusable for any reason.

## Software

Participants in the network will need to use some sort of VPN software to facilitate connections to one another with their an address in their allocation. Users can choose to use any VPN client like WireGuard or tinc, though OpenVPN will be considered the base case as it arguably has the most compatibility with different operating systems and hosting environments.

Any BGP daemon can be used to create a router, though the base case will showcase bird. Other daemons include Quagga and BGPd.

### OpenVPN

Sample configuration is available here, [OpenVPN Configuration](openvpn.md).

### Bird

Sample configuration is available here, [Bird Configuration](bird.md) (untested).
