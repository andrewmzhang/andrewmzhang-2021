---
title: Mosh + Tmux + Copy Paste
layout: post
date: "2024-04-17 19:47:39 -0500"
description: Oracle Cloud Docker No Route To Host
comments: true
author: Andrew M. Zhang
---

TLDR; Oracle cloud is pre-loaded with a bunch of iptable rules. You'll have to hack at them to get Docker networking
to behave in a typical fashion.

# The Setup

We're using an Oracle Cloud Ubuntu VM as our host. Assume a machine with LAN address `10.0.0.100` on the `etho0` interface.
We're interested in running a Linuxserver.io Swag container as a reverse proxy on a user-defined bridge Docker network
`lsio` that shows up in `ifconfig` as `br-xyz`. In the Swag container's compose file, we port forward `10.0.0.100:80:80`
and `10.0.0.100:443:443`.

```bash
# This is your typical linuxserver.io swag reverse proxy setup.
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Networks}}"
CONTAINER ID   NAMES         IMAGE                         PORTS                                          NETWORKS
91b38a3164f2   swag          lscr.io/linuxserver/swag      10.0.0.100:80->80/tcp, 10.0.0.100:443->443/tcp lsio
```

If we've set this up correctly, we expect to be able to reach a webpage at `http://10.0.0.100:80`.

```bash
ubuntu@myhost:~$ curl http://10.0.0.100:80
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

# The Problem

If we exec into the `swag` container, we expect the following ip and port to be reachable. Unfortunately, it is not.

```bash
ubuntu@myhost:~$ docker exec -it swag /bin/bash
root@133d975caa2f:/# curl 10.0.0.100
curl: (7) Failed to connect to 10.0.0.100 port 80 after 2 ms: Couldn't connect to serve
```

The reason for this is that Oracle VMs are pre-configured with a bunch of iptable rules and with UFW disabled. I believe
the reasoning for these rules is to ensure VMs are sufficiently hardened and also to manager their boot volume traffic,
which is mounted via iSCSI(?). They document their odd setup [here](https://blogs.oracle.com/developers/post/enabling-network-traffic-to-ubuntu-images-in-oracle-cloud-infrastructure).
Note that Oracle states that you need to explicitly open port 80 for web traffic to flow. However, we did not do this
with our Ubuntu VM, so why is traffic available at `10.0.0.100:80`? This behavior is driven by how Docker port forwards
work. Documented [here](https://docs.docker.com/network/packet-filtering-firewalls/#docker-and-ufw):

> When you publish a container's ports using Docker, traffic to and from that container gets diverted before it goes
> through the ufw firewall settings. Docker routes container traffic in the nat table, which means that packets are
> diverted before it reaches the INPUT and OUTPUT chains that ufw uses. Packets are routed before the firewall rules can
> be applied, effectively ignoring your firewall configuration.

If we take a look at the NAT table, we see that traffic is DNAT'd through to our docker container. When the packet hits
the DNAT rule, the next chain that will be executed is the FORWARD chain where it hits out `swag` reverse proxy.

```bash
ubuntu@myhost:~$ sudo iptables -L -v -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
 2846  181K DOCKER     all  --  any    any     anywhere             anywhere             ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
  382 55570 DOCKER     all  --  any    any     anywhere            !localhost/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 [... TRUNCATED ... ]

Chain DOCKER (2 references)
 pkts bytes target     prot opt in      out     source               destination
    0     0 RETURN     all  --  docker0 any     anywhere             anywhere
    0     0 RETURN     all  --  br-xyz  any     anywhere             anywhere
    0     0 DNAT       tcp  --  !br-xyz any     anywhere             myhost.mysubnet.myvcn.oraclevcn.com  tcp dpt:https to:172.20.0.2:443
   20  1200 DNAT       tcp  --  !br-xyz any     anywhere             myhost.mysubnet.myvcn.oraclevcn.com  tcp dpt:http to:172.20.0.2:80
```

This means you don't actually need to open up port 80 and 443 for the reverse proxy to work. Note however that traffic
originating from `br-xyz` network will not get DNAT-ed. Instead the `DOCKER` chain returns and packets end up in the
`INPUT` chain. Given Oracle's restrictive iptable settings, our packet gets rejected as it only matches the last rule.

```bash
ubuntu@myhost:~$ sudo iptables -L -v
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
 144K   20M ts-input   all  --  any    any     anywhere             anywhere             # This is because I installed tailscale w/ accept-routes
69380   12M ACCEPT     all  --  any    any     anywhere             anywhere             state RELATED,ESTABLISHED
    2   168 ACCEPT     icmp --  any    any     anywhere             anywhere
31629 2614K ACCEPT     all  --  lo     any     anywhere             anywhere
    0     0 ACCEPT     udp  --  any    any     anywhere             anywhere             udp spt:ntp
  501 29392 ACCEPT     tcp  --  any    any     anywhere             anywhere             state NEW tcp dpt:ssh
   32  4715 REJECT     all  --  any    any     anywhere             anywhere             reject-with icmp-host-prohibited
```

# The Solution

For the typical reverse-proxy application, you probably won't need to access the `swag` reverse-proxy from inside the
user-defined bridge network through ip port `10.0.0.100:80`. If the reverse-proxy redirects traffic from some public
ip or domain, you should be able to get the same effect by going through `http://mydomain.example.com:80`. However,
sometimes my reverse proxy is hosted on a Tailscale IP and it might need to "talk to itself" through its own Tailscale
IP. An example would be if you ran an uptime-kuma service that needs to check if services are up behind your reverse proxy
hosted on your host's tailscale ip.

The solution is to allow traffic headed to port 80 and 443 in the INPUT chain.

```bash
# Add the following to your iptable, where br-xyz is the network your container is hosted
-A INPUT -i br-xyz -d 10.0.0.100 -p tcp -m state --state NEW -m multiport --dports 80,443 -j ACCEPT
```

I'm not aware of any way to automatically represent the IP addr of `eth0` instead of hardcoding it like I did above. You
could make the ip subnet range by replacing it with something like `10.0.0.100/24`.

## IPv6

Oracle doesn't bork ip6tables, so you shouldn't need to do anything extra to get this to work. You will need to configure
docker to work with ipv6.
