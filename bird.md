# Bird Configuration

Here is a basic configuration creating a BGP via [Bird](http://bird.network.cz/) software. Large portions of this guide are inspired by or copied from [dn42's documentation](https://dn42.net/howto/Bird).

## Installation

Assuming a Debian installation with a non-root, sudo user. First, install `bird` on on one peer that you will designate as your router:

```
$ sudo apt-get install bird
```

Use the following sample config files and customize them with your info as outlined below:

* Replace `<AS>` with your [assigned router.city Autonomous System Number](README.md/#autonomous-system-numbers) (only the digits).
* Replace `<GATEWAY_IP>` with your gateway IP (the internal private IP address you use on the host to connect to your remote peer).
* Replace `<SUBNET>` with your registered router.city subnet.
* Replace `<PEER_IP>` with the private IP address of your peer who is connected with you using OpenVPN or whatever other VPN software you use (NOT their public IP address). (This is provided by your remote peer)
* Replace `<PEER_AS>` with the Autonomous System number of your remote peer (only the digits). (This is provided by your remote peer)
* Replace `<PEER_NAME>` with a self chosen name for your remote peer.
* Replace `<INTERFACE_NAME>`, if needed, with a self the interface you connect to your remote peer with (like `tun0` or `cty00`).

### IPv4

```
# /etc/bird/bird.conf
# Device status
protocol device {
  scan time 10; # recheck every 10 seconds
}

protocol static {
  # Static routes to announce your own range(s) in dn42
  route <SUBNET> reject;
  import all;
  export none;
};

# local configuration
######################

# keeping router specific in a seperate file, 
# so this configuration can be reused on multiple routers in your network
include "/etc/bird/local4.conf";

# filter helpers
#################

##include "/etc/bird/filter4.conf";

# Kernel routing tables
########################

/*
    krt_prefsrc defines the source address for outgoing connections.
    On Linux, this causes the "src" attribute of a route to be set.
    
    Without this option outgoing connections would use the peering IP which
    would cause packet loss if some peering disconnects but the interface
    is still available. (The route would still exist and thus route through
    the TUN/TAP interface but the VPN daemon would simply drop the packet.)
*/
protocol kernel {
  scan time 20;
  import none;
  export filter {    
    if source = RTS_STATIC then reject;
    krt_prefsrc = OWNIP;
    accept;
  };
};

# router.city peers
####################

template bgp rcpeers {
  local as OWNAS;
  # metric is the number of hops between us and the peer
  path metric 1;
  # this lines allows debugging filter rules
  # filtered routes can be looked up in birdc using the "show route filtered" command
  import keep filtered;
  import filter {
    # accept every subnet, except our own advertised subnet
    # filtering is important, because some guys try to advertise routes like 0.0.0.0
    if is_valid_network() && !is_self_net() then {
      accept;
    }
    reject;
  };
  export filter {
    # here we export the whole net
    if is_valid_network() then {
      accept;
    }
    reject;
  };
  import limit 1000 action block;
  #source address OWNIP;
};

include "/etc/bird/peers4/*";
```

```
#/etc/bird/local4.conf
# should be a unique identifier, <GATEWAY_IP> is what most people use.
router id <GATEWAY_IP>;

define OWNAS =  <AS>;
define OWNIP = <GATEWAY_IP>;

function is_self_net() {
  return net ~ [<SUBNET>+];
}

function is_valid_network() {
  return net ~ [
    172.24.0.0/16+       # router.city
#    172.20.0.0/14{21,29}, # dn42
#    172.20.0.0/24{28,32}, # dn42 Anycast
#    172.21.0.0/24{28,32}, # dn42 Anycast
#    172.22.0.0/24{28,32}, # dn42 Anycast
#    172.23.0.0/24{28,32}, # dn42 Anycast
#    172.31.0.0/16+,       # ChaosVPN
#    10.100.0.0/14+,       # ChaosVPN
#    10.0.0.0/8{15,24}     # Freifunk.net
  ];
}
```

```
# /etc/bird/peers4/<PEER_NAME>
protocol bgp <PEER_NAME> from rcpeers {
  neighbor <PEERING_IP> as <PEER_AS>;
};
```


### IPv6 

```
#/etc/bird/bird6.conf
protocol device {
  scan time 10;
}

# local configuration
######################

include "/etc/bird/local6.conf";

# filter helpers
#################

##include "/etc/bird/filter6.conf";

# Kernel routing tables
########################


/*
    krt_prefsrc defines the source address for outgoing connections.
    On Linux, this causes the "src" attribute of a route to be set.
    
    Without this option outgoing connections would use the peering IP which
    would cause packet loss if some peering disconnects but the interface
    is still available. (The route would still exist and thus route through
    the TUN/TAP interface but the VPN daemon would simply drop the packet.)
*/
protocol kernel {
  scan time 20;
  import none;
  export filter {
    if source = RTS_STATIC then reject;
    krt_prefsrc = OWNIP;
    accept;
  };
}

# static routes
################

protocol static {
  route <SUBNET> reject;
  import all;
  export none;
}

# router.city peers
####################

template bgp rcpeers {
  local as OWNAS;
  path metric 1;
  import keep filtered;
  import filter {
    if is_valid_network() && !is_self_net() then {
      accept;
    }
    reject;
  };
  export filter {
    if is_valid_network() then {
      accept;
    }
    reject;
  };
  import limit 1000 action block;
}

include "/etc/bird/peers6/*";
```

```
# /etc/bird/local6.conf
# should be a unique identifier, use same id as for ipv4
router id <GATEWAY_IP>;

define OWNAS =  <AS>;
define OWNIP = <GATEWAY_IP>;

function is_self_net() {
  return net ~ [<SUBNET>+];
}

function is_valid_network() {
  return net ~ [
  2001:db8:dead:beef::/64+  # router.city
#  fd00::/8{44,64} # ULA address space as per RFC 4193
  ];
}
```

```
# /etc/bird/peers6/<PEER_NAME>
protocol bgp <PEER_NAME> from rcpeers {
  # if you don't use link-local ipv6 addresses for peering, use the following
  # neighbor <PEERING_IP> as <PEER_AS>;
  # if you use link-local ipv6 addresses for peering using the following
  neighbor <PEERING_IP>%<INTERFACE_NAME> as <PEER_AS>;
};
```

## Running bird

Now enable and restart `bird` to have it run at boot and apply our new config:

```
$ sudo systemctl enable bird
$ sudo systemctl restart bird
$ sudo systemctl enable bird6
$ sudo systemctl restart bird6
```

### Testing bird

You can use the bird console `birdc` and `birdc6` to get some info about your configuration to make sure it is working.

If you ever need to rehash your config, use `birdc conf check` to check the config for errors, and `birdc conf` to apply the config without taking down bird. This will prevent flapping. The same commands can be used with `bird6`, `birdc6 conf check` and `birdc6 conf`.

```
$ birdc show route
BIRD 1.4.5 ready.
172.24.0.0/24      unreachable [static1 2018-08-10] * (200)
172.24.2.0/24      via 192.168.254.2 on cty02 [darkdrgn2k 02:10:49] * (100) [AS64498i]

$ birdc show protocols
BIRD 1.4.5 ready.
name     proto    table    state  since       info
device1  Device   master   up     2018-08-10
static1  Static   master   up     2018-08-10
kernel1  Kernel   master   up     2018-08-10
darkdrgn2k BGP      master   up     01:35:36    Established

$ ip route | grep bird
172.24.2.0/24 via 192.168.254.2 dev cty02  proto bird  src 192.168.254.1
```

```
$ birdc6 show route
BIRD 1.4.5 ready.
2001:db8:dead:beef:cafe:f00d::/112 unreachable [static1 2018-08-10] * (200)
2001:db8:dead:beef:cafe:be11::/112 via fe80:1::2 on cty02 [darkdrgn2k 01:35:02] * (100) [AS64498i]

$ birdc6 show protocols
BIRD 1.4.5 ready.
name     proto    table    state  since       info
device1  Device   master   up     2018-08-10
kernel1  Kernel   master   up     2018-08-10
static1  Static   master   up     2018-08-10
darkdrgn2k BGP      master   up     01:35:03    Established

$ ip -6 route | grep bird
2001:db8:dead:beef:cafe:be11::/112 via fe80:1::2 dev cty02  proto bird  src fe80:1::1  metric 1024
```

## Adding a local router.city address

Now that BGP is working via bird, we can attach router.city addresses in our allocations to an existing interface.

Assuming a debian system with a sudo user, open up `/etc/network/interfaces` and add the following to the bottom of the file where `<IFACE>` is an existing network interface on your system (like `eth0`), `<RC_ADDRESS_IPV6>` is a router.city address in your [IPv6 allocation](allocations/ipv6.md) and `<RC_ADDRESS_IPV4>` is a router.city address in your [IPv4 allocation](allocations/ipv4.md).

```
iface <IFACE> inet6 static
        <RC_ADDRESS_IPV6>
        netmask 112

iface <IFACE> inet static
        address <RC_ADDRESS_IPV4>
```

Now, restart networking:

```
$ sudo systemctl restart networking
```