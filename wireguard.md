# OpenVPN Configuration

Here is a basic configuration for connecting two peers via [Wireguard](https://www.wireguard.com/) software. This example does not use Crypto Routing so each peer has a separateinterface.

## Installation

Assuming a Debian installation with a non-root, sudo user. First, install `wireguard` on both peers (we are assuming you are running a local peer and a friend has a remote peer):

```
$ sudo apt-get install wireguard
```

Create a private/public key pair:
```
privateKey=$(wg genkey)
publicKey=$(echo $privateKey | wg pubkey)
```

You can view these values using

```
echo $privateKey
echo $publicKey
```

Now we will create a config file on wg-cty0. Each new peer will increate the index (wg-cty1, wg-cty2 etc). This exmapls uses wg-cty0 but update as needed. These DO NOT need to match on the two nodes

```
$ sudo nano /etc/wireguard/wg-cty0.conf
```

Select one peer to be the server, and the other the client. Exchange value of `echo $oublicKey`. Decide on a network range between your nodes. IPv4 can use `/30` Cirds and IPv6 `/128` to save IP space. Server will also need to provide the IP address of their node and the ListenPort.

Use the following sample config on both peers and customize it with your info as outlined below:

* Replace `<PrivateKey>` with the contents of `echo $privateKey` 
* Replace `<Address>` with the IP address and `cidr` your wireguard peer to peer network will have. You can add multiple `Address =` entries if you want multiple IP addresses and/or IPv6 addresses.
* Replace `<PublicKey>` under `[Peer]` with the remote node's `echo $publicKey`
* **Server Node Only**: Comment out the Endpoint line under `[Peer]`
* **Server Node Only**: Replace `<ListenPort> to a free port on you node. 
* **Client Node Only**: Replace `<Endpoint>` with the public IP of the server node, and the port set under `ListenPort` on server node in the format 12.34.56.78:1234

* **Client Node Only**: Comment out `ListenPort` or set it to a free port on your node.

```
[Interface]
PrivateKey = <PrivateKey>
Address = <Address>
ListenPort = <ListenPort>
Table = off

[Peer]
PublicKey = <PublicKey>
AllowedIPs = 0.0.0.0/0
AllowedIPs = ::/0
Endpoint = <Endpoint> # Comment out for server
PersistentKeepalive = 20
```


On the server, before starting up, make sure that the port specified in `<ListenPort>` is opened in each machine's firewall and forwarded if necessary.

When done, start `wireguard` the new wireguard instance.

```
$ sudo systemctl start wg-quick@wg-cty0
```

Further, we can get it set up with systemd to run at boot.

```
$ sudo systemctl enable wg-quick@wg-cty0
```

When done, check the interfaces on each node and do a quick ping test if you'd like.

You can check the status of the wireguard instance by using the `wg` command. After having some data sent over the network you should see the transfer numbers increase. If one TX increases but RX does not, check the private and public keys are correct on both peers. Remember that the config has the local node's private key but the remote public key. `wg` however  will never show the local nodes private key, it will instead show the local nodes public key and the public keys of your peers.

When you update the config remember to restart the instance using

```
$ sudo systemctl restart wg-quick@wg-cty0
```

## Sample Testing

Make sure you ping from the client node, as the server node does not know where the client node is. The client nodes should automatically internal ping the server node every 20 seconds automatically to start the connection and keep the connection alive.

In this example the peers
Local peer: 192.168.254.1 and fe80:1::1/128
Remote Peer: 192.168.251.2 fe80:1::2/128

```
$  ip a show wg-cty0
239: wg-cty0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1412 qdisc pfifo_fast state UNKNOWN group default qlen 100
    link/none
    inet 192.168.254.1/30 scope global cty02
       valid_lft forever preferred_lft forever
    inet6 fe80:1::1/128 scope link
       valid_lft forever preferred_lft forever

$ ping -c4 -I wg-cty0 192.168.254.2
PING 192.168.254.2 (192.168.254.2) from 192.168.254.1 wg-cty0: 56(84) bytes of data.
64 bytes from 192.168.254.2: icmp_seq=1 ttl=64 time=91.4 ms
64 bytes from 192.168.254.2: icmp_seq=2 ttl=64 time=91.1 ms
64 bytes from 192.168.254.2: icmp_seq=3 ttl=64 time=91.2 ms
64 bytes from 192.168.254.2: icmp_seq=4 ttl=64 time=91.3 ms

--- 192.168.254.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 91.180/91.292/91.426/0.315 ms

$ ping6 -c4 -I wg-cty0 fe80:1::2
PING fe80:1::2(fe80:1::2) from fe80:1::1 wg-cty0: 56 data bytes
64 bytes from fe80:1::2: icmp_seq=1 ttl=64 time=91.1 ms
64 bytes from fe80:1::2: icmp_seq=2 ttl=64 time=91.2 ms
64 bytes from fe80:1::2: icmp_seq=3 ttl=64 time=91.9 ms
64 bytes from fe80:1::2: icmp_seq=4 ttl=64 time=90.7 ms

--- fe80:1::2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 90.768/91.264/91.901/0.408 ms
```

If pings go throught but larger things do not, try setting your local interfaces MTU to 1400

```
ifconfig eth0 MTU 1400
```
