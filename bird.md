# Bird Configuration

Here is a basic configuration creating a BGP via [Bird](http://bird.network.cz/) software. Large portions of this guide are inspired by or copied from [dn42's documentation](https://dn42.net/howto/Bird).

## Installation

Assuming a Debian installation with a non-root, sudo user. First, install `bird` on on one peer that you will designate as your router:

```
$ sudo apt-get install bird
```

Use the following sample config files and customize them with your info as outlined below:

* Replace `<AS>` with your [assigned router.city Autonomous System Number](README.md/#autonomous-system-numbers) (only the digits).
* Replace `<GATEWAY_IP>` with your gateway IP (the internal router.city IP address you use on the host).
* Replace `<SUBNET>` with your registered router.city subnet.
* Replace `<PEER_IP>` with the IP of your peer who is connected with you using OpenVPN. (This is provided by your remote peer)
* Replace `<PEER_AS>` with the Autonomous System number of your remote peer (only the digits). (This is provided by your remote peer)
* Replace `<PEER_NAME>` with a self chosen name for your remote peer.

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

template bgp dnpeers {
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
    fd00::/8{44,64} # ULA address space as per RFC 4193
  ];
}
```

```
# /etc/bird/peers6/<PEER_NAME>
protocol bgp <PEER_NAME> from dnpeers {
  neighbor <PEERING_IP> as <PEER_AS>;
  # if you use link-local ipv6 addresses for peering using the following
  # neighbor <PEERING_IP> % '<INTERFACE_NAME>' as <PEER_AS>;
};
```

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
# DN42
#######

template bgp dnpeers {
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
    172.20.0.0/14{21,29}, # dn42
    172.20.0.0/24{28,32}, # dn42 Anycast
    172.21.0.0/24{28,32}, # dn42 Anycast
    172.22.0.0/24{28,32}, # dn42 Anycast
    172.23.0.0/24{28,32}, # dn42 Anycast
    172.31.0.0/16+,       # ChaosVPN
    10.100.0.0/14+,       # ChaosVPN
    10.0.0.0/8{15,24}     # Freifunk.net
  ];
}
```

```
# /etc/bird/peers4/<PEER_NAME>
protocol bgp <PEER_NAME> from dnpeers {
  neighbor <PEERING_IP> as <PEER_AS>;
};
```


Run the following commands to enable IPv4 and IPv6 forwarding:

```
$ sudo sysctl net.ipv4.ip_forward=1
$ sudo sysctl net.ipv6.conf.all.forwarding=1
```

Now restart `bird`:

```
$ sudo systemctl restart bird
$ sudo systemctl restart bird6
```