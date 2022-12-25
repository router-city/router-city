# IPv6 Addressing

router.city will utilize both IPv4 and IPv6 addressing within the network.

IPv6 addresses will be assigned from the `2001:db8:dead:beef::/64` block, which is reserved space for documentation and source code examples. This range does not conflict with dn42, ChaosVPN, Friefunk, Yggdrasil, or cjdns.

Please add your allocation in lexicographical order.

## Current IPv6 Allocations

| CIDR                                    | Range                                                                        | User          |
| --------------------------------------- | ---------------------------------------------------------------------------- | ------------- |
| 2001:db8:dead:beef:4cbe::/80            | 2001:0db8:dead:beef:4cbe:0:0:0 - 2001:0db8:dead:beef:4cbe:ffff:ffff:ffff     | [Bandura Communications](https://byeob.de/)|
| 2001:db8:dead:beef:cafe:f00d:0::/112    | 2001:db8:dead:beef:cafe:f00d:0:0 - 2001:db8:dead:beef:cafe:f00d:0:ffff       | [Famicoman](https://github.com/Famicoman)|
| 2001:db8:dead:beef:cafe:babe:0::/112    | 2001:db8:dead:beef:cafe:babe:0:0 - 2001:db8:dead:beef:cafe:babe:0:ffff       | [mikenabhan](https://github.com/mikenabhan)|
| 2001:db8:dead:beef:cafe:be11:0::/112    | 2001:db8:dead:beef:cafe:be11:0:0 - 2001:db8:dead:beef:cafe:be11:0:ffff       | [darkdrgn2k](https://github.com/darkdrgn2k)|
| 2001:db8:dead:beef::/112                | 2001:db8:dead:beef:0:0:0:0 - 2001:db8:dead:beef:0:0:0:ffff                   | [brannondorsey](https://github.com/brannondorsey)|

These ranges may be changed at any time if they are found to conflict with another range or be unusable for any reason.
