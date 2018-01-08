# Latency Injection

In this workshop latency will be injected into a network in order to simulate real world conditions.

We will deploy three virtual machines:

1. A client node
2. A router node
3. A server node

`tc` will be used to inject latency into the links of the router node.

## Overview

```

                              Management Network
   +-------------------------------------------------------------------+
         |                            |                          |
         |                            |                          |
         |                            |                          |
         |                            |                          |
   +------------+              +------------+             +------------+
   |            |              |            |             |            |
   |   Client   +--------------+   Router   +-------------+   Server   |
   |            |              |            |             |            |
   +------------+              +------------+             +------------+
                     Link1                       Link2


```

## Setup Virtual Resources

The virtual resources used in this workshop will be created in an OpenStack cloud.

### Create Networks and Subnets

Create three networks.

1. `li-mgmt` - A management network
2. `li-link1` - link1 network
3. `li-link2` - link2 network

*NOTE: The `openstack` command is abbreviated to `os`.*

*NOTE: For simplicity port security is being disabled.*

```
os network create li-mgmt --disable-port-security
os network create li-link1 --disable-port-security
os network create li-link2 --disable-port-security
```

Associate subnets with the networks.

*NOTE: A /29 netmask creates a smaller network than a /24.*

```
os subnet create --subnet-range=192.168.0.0/24 --network li-mgmt li-mgmt-subnet
os subnet create --subnet-range=172.16.0.0/29 --network li-link1 li-link1-subnet
os subnet create --subnet-range=172.16.0.8/29 --network li-link2 li-link2-subnet
```

os subnet create --subnet-range=172.16.0.0/29 --allocation-pool start=172.16.0.1,end=172.16.0.6 --network li-link1 li-link1-subnet
os subnet create --subnet-range=172.16.0.8/29 --allocation-pool start=172.16.0.9,end=172.16.0.14 --network li-link2 li-link2-subnet

Subnets:

```
$ os subnet list -c Name -c Subnet
+--------------------------+------------------+
| Name                     | Subnet           |
+--------------------------+------------------+
| li-link2-subnet          | 172.16.0.8/29    |
| li-mgmt-subnet           | 192.168.0.0/24   |
| li-link1-subnet          | 172.16.0.0/29    |
+--------------------------+------------------+
```

A router will NOT be setup between these networks. However, one will be required to assign a floating IP to the Client virtual machines management network so that it can be accessed remotely and act as a jump host to the other virtual machines.


### Create a OpenStack Neutron Router

A Neutron router is necessary to enable floating IPs.

TBD

### Create Ports

```
os port create --network li-mgmt --fixed-ip subnet=li-mgmt-subnet,ip-address=192.168.0.10 client-mgmt
os port create --network li-mgmt --fixed-ip subnet=li-mgmt-subnet,ip-address=192.168.0.20 router-mgmt
os port create --network li-mgmt --fixed-ip subnet=li-mgmt-subnet,ip-address=192.168.0.30 server-mgmt
os port create --network li-link1 --fixed-ip subnet=li-link1-subnet,ip-address=172.16.0.6 client-link1
os port create --network li-link1 --fixed-ip subnet=li-link1-subnet,ip-address=172.16.0.5 router-link1
os port create --network li-link2 --fixed-ip subnet=li-link2-subnet,ip-address=172.16.0.14 server-link2
os port create --network li-link2 --fixed-ip subnet=li-link2-subnet,ip-address=172.16.0.13 router-link2
```

### Build Virtual Machines

Now that the ports have been created the virtual machines can be built.

```
openstack server create \
	--image xenial \
	--flavor m1.small \
	--nic port-id=client-mgmt \
	--nic port-id=client-link1 \
	--key-name default \
	li-client

openstack server create \
	--image xenial \
	--flavor m1.small \
	--nic port-id=server-mgmt \
	--nic port-id=server-link2 \
	--key-name default \
	li-server

openstack server create \
	--image xenial \
	--flavor m1.small \
	--nic port-id=router-mgmt \
	--nic port-id=router-link1 \
	--nic port-id=router-link2 \
	--key-name default \
	li-router
```

VMs are now active and have the expected IP addresses.

```
os server list -c Name -c Status -c Networks
+----------------------+--------+------------------------------------------------------------------------------------+
| Name                 | Status | Networks                                                                           |
+----------------------+--------+------------------------------------------------------------------------------------+
| li-server            | ACTIVE | li-link2=172.16.0.14; li-mgmt=192.168.0.30                                         |
| li-router            | ACTIVE | li-link2=172.16.0.13; li-mgmt=192.168.0.20; li-link1=172.16.0.5                    |
| li-client            | ACTIVE | li-mgmt=192.168.0.10; li-link1=172.16.0.6                                          |
+----------------------+--------+------------------------------------------------------------------------------------+
```

### Assign a Floating IP

TBD

## Latency Injection Lab

### Client Node Configuration

Now that the virtual resources have been created the latency injection lab can begin.

Start by ssh-ing into the client node using the floating IP that was attached to the client node.

*NOTE: `SNIP!` means extra output was removed for brevity and is not shown.*

```
$ ssh ubuntu@<floating IP>
SNIP!
ubuntu@li-client:~$
```

#### Setup /etc/hosts

Paste the below into /etc/hosts on the client node.

```
127.0.0.1 localhost li-client

192.168.0.10 client-mgmt
192.168.0.20 router-mgmt
192.168.0.30 server-mgmt
172.16.0.6 client-link1
172.16.0.5 router-link1
172.16.0.14 server-link2
172.16.0.13 router-link2


# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```

#### Validate Access to Server and Router Nodes

From the client node ensure ssh access works to the server and router nodes.

```
ubuntu@li-client:~$ ssh router-mgmt hostname
li-router
ubuntu@li-client:~$ ssh server-mgmt hostname
li-server
```

Check if the IP address on port server-link2 is available (it shouldn't be):

```
ubuntu@li-client:~$ ping -c 3 -W 5 server-link2
PING 172.16.0.14 (172.16.0.14) 56(84) bytes of data.
From 198.55.49.251 icmp_seq=1 Destination Net Unreachable
From 198.55.49.251 icmp_seq=2 Destination Net Unreachable
From 198.55.49.251 icmp_seq=3 Destination Net Unreachable

--- 172.16.0.14 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 2003ms
```

When the lab is complete, the above ping should work.

### Setup Routes

Given this is an artificial lab, all of the nodes have an interface on the management network. But this interface will only be used to ssh into the nodes and configure them for the lab. The actual routing component and access from client to server for the purposes of latency inject will be done via the link1 and link2 networks (not over the managment network).

On the client node:

```
ubuntu@li-client:~$ sudo ip route add 172.16.0.8/29 via 172.16.0.5 dev ens4
ubuntu@li-client:~$ ip ro sh
default via 192.168.0.1 dev ens3
169.254.169.254 via 192.168.0.1 dev ens3
172.16.0.0/29 dev ens4  proto kernel  scope link  src 172.16.0.6
172.16.0.8/29 via 172.16.0.5 dev ens4
192.168.0.0/24 dev ens3  proto kernel  scope link  src 192.168.0.10
```

On the server node:

```
ubuntu@li-server:~$  sudo ip route add 172.16.0.0/29 via 172.16.0.13 dev ens4
ubuntu@li-server:~$ ip ro sh
default via 192.168.0.1 dev ens3
169.254.169.254 via 192.168.0.1 dev ens3
172.16.0.0/29 via 172.16.0.13 dev ens4
172.16.0.8/29 dev ens4  proto kernel  scope link  src 172.16.0.14
192.168.0.0/24 dev ens3  proto kernel  scope link  src 192.168.0.30
```

### Enable IP Forwarding in Router

On the router:

```
ubuntu@li-router:~$ sudo sysctl -w net.ipv4.ip_forward=1
```

### Ping server-link2 from client

With the routes setup and IP forwarding enabled on the router, the below ping from the client to the server-link2 interface should work.

```
ubuntu@li-client:~$ ping -c 3 -W 5 server-link2
PING server-link2 (172.16.0.14) 56(84) bytes of data.
64 bytes from server-link2 (172.16.0.14): icmp_seq=1 ttl=63 time=1.16 ms
64 bytes from server-link2 (172.16.0.14): icmp_seq=2 ttl=63 time=1.24 ms
64 bytes from server-link2 (172.16.0.14): icmp_seq=3 ttl=63 time=1.31 ms

--- server-link2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 1.160/1.239/1.312/0.074 ms
```

Note that the latency is ~1.2ms.

### Add Latency with tc

Now using tc we can add latency on the router node.

Let's add 25ms of latency on the ens4/link1 interface.

```
ubuntu@li-router:~$ sudo tc qdisc add dev ens4 root netem delay 25ms
```

Rerun the ping from the client to server interface.

```
ubuntu@li-client:~$ ping -c 3 -W 5 server-link2
PING server-link2 (172.16.0.14) 56(84) bytes of data.
64 bytes from server-link2 (172.16.0.14): icmp_seq=1 ttl=63 time=27.8 ms
64 bytes from server-link2 (172.16.0.14): icmp_seq=2 ttl=63 time=26.3 ms
64 bytes from server-link2 (172.16.0.14): icmp_seq=3 ttl=63 time=26.5 ms

--- server-link2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 26.396/26.933/27.865/0.674 ms
```

Latency is now ~26ms.

Add 25ms to the link2/ens5 interface on the router.

```
ubuntu@li-router:~$ sudo tc qdisc add dev ens5 root netem delay 25ms
```

Ping again from the client node to the server link2 interface.

```
ubuntu@li-client:~$ ping -c 3 -W 5 server-link2
PING server-link2 (172.16.0.14) 56(84) bytes of data.
64 bytes from server-link2 (172.16.0.14): icmp_seq=1 ttl=63 time=52.5 ms
64 bytes from server-link2 (172.16.0.14): icmp_seq=2 ttl=63 time=51.6 ms
64 bytes from server-link2 (172.16.0.14): icmp_seq=3 ttl=63 time=51.5 ms

--- server-link2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 51.501/51.903/52.546/0.529 ms
```

Latency is now ~52ms. This is something like the distance between Vancouver and Toronto in terms of latency.

`tc` can show all configurations.

```
ubuntu@li-router:~$ sudo tc -s qdisc
qdisc noqueue 0: dev lo root refcnt 2
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0
qdisc pfifo_fast 0: dev ens3 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
 Sent 1109887 bytes 13480 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0
qdisc netem 8001: dev ens4 root refcnt 2 limit 1000 delay 25.0ms
 Sent 26242 bytes 170 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 676b 6p requeues 0
qdisc netem 8002: dev ens5 root refcnt 2 limit 1000 delay 25.0ms
 Sent 714 bytes 9 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0
```

### Validate with tcpdump

If desired, `tcpdump` can be run on a router interface to ensure ICMP traffic is indeed being routed via that interface.

While `tcpdump` is running, from another terminal/console ping the server-link2 interface from the client node.

```
ubuntu@li-router:~$ sudo tcpdump -n -e -ttt -i ens5 icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens5, link-type EN10MB (Ethernet), capture size 262144 bytes
 00:00:00.000000 fa:16:3e:1f:8d:d3 > fa:16:3e:fd:0e:b3, ethertype IPv4 (0x0800), length 98: 172.16.0.6 > 172.16.0.14: ICMP echo request, id 1813, seq 1, length 64
 00:00:00.002041 fa:16:3e:fd:0e:b3 > fa:16:3e:1f:8d:d3, ethertype IPv4 (0x0800), length 98: 172.16.0.14 > 172.16.0.6: ICMP echo reply, id 1813, seq 1, length 64
 00:00:00.998769 fa:16:3e:1f:8d:d3 > fa:16:3e:fd:0e:b3, ethertype IPv4 (0x0800), length 98: 172.16.0.6 > 172.16.0.14: ICMP echo request, id 1813, seq 2, length 64
 00:00:00.000710 fa:16:3e:fd:0e:b3 > fa:16:3e:1f:8d:d3, ethertype IPv4 (0x0800), length 98: 172.16.0.14 > 172.16.0.6: ICMP echo reply, id 1813, seq 2, length 64
 00:00:01.001181 fa:16:3e:1f:8d:d3 > fa:16:3e:fd:0e:b3, ethertype IPv4 (0x0800), length 98: 172.16.0.6 > 172.16.0.14: ICMP echo request, id 1813, seq 3, length 64
 00:00:00.000642 fa:16:3e:fd:0e:b3 > fa:16:3e:1f:8d:d3, ethertype IPv4 (0x0800), length 98: 172.16.0.14 > 172.16.0.6: ICMP echo reply, id 1813, seq 3, length 64
```

## Lab Completed

The lab is now complete!

Now if we would like we could simulate long geographic distances and perhaps test things like MySQL Galera clusters over these simulated high latency links. Or setup more routers and clients and servers, and configure different latency between them.

## Teardown Lab

Delete VMs:

```
os server delete li-client li-server li-router
```

Delete ports:

```
export PORTS="client-mgmt
server-mgmt
router-mgmt
client-link1
router-link1
server-link2
router-link2"

for p in $PORTS; do
  os port delete $p
done
```

Delete subnets:

```
export SUBNETS="li-mgmt-subnet
li-link1-subnet
li-link2-subnet"

for s in $SUBNETS; do
  os subnet delete $s
done
```

Delete networks:

```
export NETS="li-mgmt
li-link1
li-link2"

for n in $NETS; do
  os network delete $n
done
```
