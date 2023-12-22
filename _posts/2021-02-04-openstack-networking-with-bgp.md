---
layout: post
title: "OpenStack Networking With BGP"
date: "2021-02-04"
categories: BGP
---

With the increment of virtualized/containerized workloads it is becoming more and more common to use pure layer-3 Spine and Leaf network deployments at datacenters. There are several benefits of this, such as reduced complexity at scale, reduced failures domains, limiting broadcast traffic, among others.

The goal of this blogpost is to present the ongoing effort to develop a new agent in charge of exposing OpenStack VMs through BGP on an OVN based deployment. The connectivity between the OpenStack controller/compute nodes happens through the Spine and Leaf network. More specifically the agent is in charge of :

- Advertising IPv4/IPv6 host routes on provider networks
- Advertising IPv4/IPv6 tenant networks via Neutron Gateway router
- Advertising Floating IPs (IPv4) host routes directly to compute nodes (DVR)

## OVN-BGP Agent

The [OVN-BGP Agent](https://github.com/luis5tb/bgp-agent) is a Python based daemon that runs on each node (OpenStack controller/compute node), assuming they are all connected to the BGP peers (leafs). It monitors the OVN Southbound database to detect when a VM boots/shuts down, as well as when a FIP gets associated/disassociated to it. When this happens, leveraging kernel networking capabilities, it makes the needed reconfiguration so that the IP address associated with that VM is advertised via BGP, as well as performs all the required actions to route the external traffic to the OVN overlay.

The OVN-BGP Agent operates under the following assumptions:

- Some **dynamic routing solution** (eg. FRR) is deployed and advertises/withdraws routes added/deleted to/from certain local interface(s), including the ones associated to VRFs (used to avoid interfering with local routing).
- The relevant provider **OVS bridges** are created, configured with a loopback IP address (eg. 1.1.1.1/32 for IPv4) and **proxy ARP/NDP** is enabled on their kernel interface. In the case of OpenStack, this shall be done by TripleO.

## How routing works

The actions performed by the OVN-BGP Agent to ensure VMs are reachable through BGP as expected can be divided in two groups:

- **_Traffic between nodes or BGP Advertisement_**: These are the actions needed to expose the BGP routes and make sure all the nodes know how to reach the VM IP on the nodes.
- **_Traffic within a node or redirecting traffic to/from OVN overlay_**: These are the actions needed to redirect the traffic to/from a VM on the OVN neutron networks, once they have reached the node where the VM is or in their way out of the node.

## BGP Advertisement

The agent is in charge of triggering [FRR](https://frrouting.org/) (ip routing protocol suite for Linux which includes protocol daemons for BGP, OSPF, RIP, among others) to advertise/withdraw **directly connected routes** via BGP. To do that, when the agent starts it ensures that:

- There is a **VRF** created, by default with name “ovn-bgp-vrf”
- There is a **dummy interface** type (by default with name “ovn”) associated with the above created VRF device.

Then, to expose the routes the agent just needs to **_add/remove the VM IP (or FIP) into/from the dummy interface_**. This triggers FRR to advertise (or withdraw in the case of deletion) that IP if it is configured to advertise directly connected routes. Note, as we also want to be able to expose VM connected to tenant networks, there is a need to expose the Neutron router gateway port (CR-LRP on OVN) so that the traffic to VMs on tenant networks is injected into OVN overlay through the node that is hosting that port.

## Redirecting traffic to/from OVN

With the BGP advertisement process we ensured we know the node where the VM is and how to send the traffic to that node (or where the CR-LRP port is, for the VMs on the tenant networks). However, once the traffic reaches the required node, the agent needs to make sure the traffic is injected into the OVN overlay networks. And similarly for the traffic out from the OVN overlay networks.

### Traffic to OVN

For the traffic that reaches the node and that needs to be redirected to the OVN overlay (i.e., provider OVS bridge), the OVN-BGP Agent relies on kernel networking, in this case **_rules_** and **_routes_**, to redirect the traffic. In order to keep the default routing table of the servers where OVN-BGP Agent is running as clean as possible, the agent relies on the existence of extra routing tables for each one of the OVS provider bridges:

- If the provider OVS bridge name is br-ex, we should have something like the next:

```
$ cat /etc/iproute2/rt_tables
#
# reserved values
#
255     local
254     main
253     default
0       unspec
#
# local
200 br-ex
```

Then, depending on the events, and whether they are related to the local hypervisor running the agent, the next IP rules needs to be created by the agent to redirect the traffic to the associated routing table when the traffic is destined to OVN networks:

- IP rules for either VM IP on the provider network, VM FIP, or neutron router gateway port (cr-lrp):

```
$ ip rule
0:      from all lookup local
1000:   from all lookup [l3mdev-table]
32000:  from all to 172.24.4.146 lookup br-ex
32000:  from all to 172.24.6.225 lookup br-ex
32766:  from all lookup main
32767:  from all lookup default
```

- IP rules for the tenant networks CIDR:

```
$ ip rule
0:      from all lookup local
1000:   from all lookup [l3mdev-table]
32000:  from all to 10.0.0.65/26 lookup br-ex
32766:  from all lookup main
32767:  from all lookup default
```

Once the rules are in place, the traffic is redirected to the br-ex routing table. The agent is in charge of populating it with the default route, as well as some extra rules for tenant network traffic redirection:

- Default route:

```
$ ip route show table br-ex
default dev br-ex scope link
```

- And the routes to ensure the traffic to the network is redirected through the CR-LRP port to the ovs provider bridge (br-ex in this case):

```
$ ip route show table br-ex
default dev br-ex scope link
10.0.0.0/26 via 172.24.4.146 dev br-ex
172.24.4.146 dev br-ex scope link
```

And finally, a **static ARP entries** for the Router gateway ports (172.24.4.146 in the above example) must be added so that traffic is steered to OVN via br-int (this is because OVN will not reply to ARP requests outside its L2 network).

```
$ ip nei
172.24.4.146 dev br-ex lladdr fa:16:3e:b5:a6:77 PERMANENT
```

### Traffic from OVN

On the other hand, for the traffic that must be sent out of OVN overlay, the agent needs to configure the OVS provider bridges to **rewrite the destination MAC address to that of the bridge kernel interface** (br-ex in this case). This will allow ECMP routing at kernel level through the physical interfaces as well as performing **proxy ARP/NDP**.

Note that for this to work, the next kernel flags need to be enable to ensure needed configuration for ARP/NDP:

```
(ipv4) sudo sysctl -w net.ipv4.conf.all.rp_filter=0
(ipv4) sudo sysctl -w net.ipv4.conf.br-ex.proxy_arp=1
(ipv4) sudo sysctl -w net.ipv4.ip_forward=1
(ipv6) sudo sysctl -w net.ipv6.conf.br-ex.proxy_ndp=1
(ipv6) sudo sysctl -w net.ipv6.conf.all.forwarding=1
```

And the bridge kernel interface must have an IP, e.g.:

```
$ sudo ip a a 1.1.1.1/32 dev br-ex
$ sudo ip a a fd53:d91e:0400:7f17::1/128 dev br-ex
```

## Events

The OVN-BGP Agent watches the OVN Southbound Database, and the above mentioned actions are triggered based on the events detected. The agent is reacting to the next events, all of them by watching the **Port\_Binding** OVN table:

- **PortBindingChassisCreatedEvent** and **PortBindingChassisDeletedEvent**: Detects when a port of type **“”** or **“chassisredirect”** gets attached to an OVN chassis. This is the case for VM ports on the provider networks, VM ports on tenant networks which have a FIP associated, and neutron gateway router ports (CR-LRPs). In this case the ip rules are created for the specific IP of the port as well as (BGP) exposed through the ovn dummy interface. For the CR-LRP case, extra rules for its associated subnets are created, as well as the extra routes for the ovs provider bridges routing table (e.g., br-ex).
- **FIPSetEvent** and **FIPUnsetEvent**: Detects when a patch port gets its nat\_addresses field updated (e.g., action related to FIPs NATing). If that so, and the associated VM port is on the local chassis the event is processed by the agent and the required ip rule gets created and also the IP is (BGP) exposed through the ovn dummy interface.
- **SubnetRouterAttachedEvent** and **SubnetRouterDetachedEvent**: Detects when a “LRP” patch port gets created or deleted. This means a subnet is attached to a router. If the chassis is the one having the CR-LRP port for that router where the port is getting created, then the event is processed by the agent and the ip rules for exposing the network are created as well as the related routes in the ovs provider bridge routing table (e.g., br-ex).
- **TenantPortCreatedEvent** and **TenantPortDeletedEvent**: Detects when a port of type **“”** gets updated or deleted. If that port is not on a provider network and the chassis where the event is detected has the LRP for the network where that port is located (meaning is the node with the CR-LRP for the router where the port’s network is connected to), then the event is processed and the port IP is (BGP) exposed.

For the list of steps to create a testing environment you can check the follow-up blog posts: [OVN-BGP Agent: Testing Setup](ovn-bgp-agent-testing-setup). And for a more in-depth description of the traffic flows in this environment you can check the follow-up blog post: [OVN BGP Agent: In-depth traffic flow inspection](ovn-bgp-agent-in-depth-traffic-flow-inspection)
