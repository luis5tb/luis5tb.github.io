---
layout: post
title: "OVN BGP AGENT: In-depth traffic flow inspection"
date: "2021-02-04"
categories: BGP
---

In the [initial\_blog post](openstack-networking-with-bgp) we explained how the OVN-BGP agent works, and in the [follow up blog post](ovn-bgp-agent-testing-setup) we explained how to set up a testing environment. Now we make use of that environment to check that the connectivity works and how the traffic gets redirected with the several rules/routes that the OVN-BGP Agent creates.

By default, devstack already creates a public shared provider network, as well as a private one. So, let's reuse the public one for creating the VM on the provider network. But first let’s prepare the environment for it by creating a neutron security group that allows connectivity (tcp and icmp), and enable the dhcp on the provider network so that we can create VMs on them:

```
$ vagrant ssh rack-1-host-1  # Let’s use the controller for triggering the OpenStack commands
$ source ~/devstack/openrc admin demo
$ openstack security group create -c id -f value sg-testing
$ openstack security group rule create --protocol tcp sg-testing
$ openstack security group rule create --protocol icmp sg-testing
$ openstack subnet set --dhcp public-subnet
```

## VMs on Provider Networks

To create a VM on the provider network, we first create a port and then a VM using that port. Note, for helping the debugging process we are going to also force the VM to be allocated on specific nodes. For this VM we will use rack-1-host-2:

```
$ vagrant ssh rack-1-host-1  # Let’s use the controller for triggering the OpenStack commands
$ source ~/devstack/openrc admin demo
$ openstack port create --security-group sg-testing --network public vm1-port
$ openstack server create --flavor m1.nano --nic port-id=$(openstack port list | grep vm1-port | awk {'print $2'}) --availability-zone nova:rack-1-host-2 --image cirros-0.5.1-x86_64-disk vm1-provider
```

This results in the next VM:

```
$ openstack server list
+--------------------------------------+--------------+--------+-----------------------------------+--------------------------+---------+
| ID                                   | Name         | Status | Networks                          | Image                    | Flavor  |
+--------------------------------------+--------------+--------+-----------------------------------+--------------------------+---------+
| b5895d69-c4ed-46f9-91ae-e34297fc67ab | vm1-provider | ACTIVE | public=2001:db8::21c, 172.24.4.92 | cirros-0.5.1-x86_64-disk | m1.nano |
+--------------------------------------+--------------+--------+-----------------------------------+--------------------------+---------+
```

As you see, the IP that got allocated is 172.24.4.92. Now let's see what the agent has done and how the incoming/outcoming traffic looks like.

Only the agent where the VM is created has an event raised: **PortBindingChassisCreatedEvent**. Then agent react to this event, by detecting it is a VM allocated in a provider network and does the next two actions:

- (BGP) expose the ip by adding it to the ovn dummy interface:

```
$ vagrant ssh rack-1-host-2
$ ip a sh ovn
19: ovn: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovn-bgp-vrf state UNKNOWN group default qlen 1000
    link/ether 3e:40:6f:58:c3:1b brd ff:ff:ff:ff:ff:ff
    inet 172.24.4.92/32 scope global ovn
       valid_lft forever preferred_lft forever
```

- Add the rule needed to redirect incoming traffic to br-ex routing table, and from there the default route will send it to br-ex ovs bridge:

```
$ vagrant ssh rack-1-host-2
$ ip rule
0:      from all lookup local
1000:   from all lookup [l3mdev-table]
32000:  from all to 172.24.4.92 lookup br-ex
32766:  from all lookup main
32767:  from all lookup default

$ ip route show table br-ex
default dev br-ex proto static
```

As regards to the traffic, let's follow the round trip for 3 different cases:

- External traffic, from spine VM
- VM on different compute on the provider network
- VM on same compute on the provider network

### External traffic, from spine VM

The external traffic, or from the spine VM in our testing environment follows a path similar to the one depicted in the next figure:

![](https://lh4.googleusercontent.com/8hrmB0hvmk24K-tOxorBNRmrHeOZJU_POXwub58-A4qBE21caBaBNqASatCoC92SDLCudF_3K_5t8CyLOMa0pqnpsRtS-Rvhrbbx3uRyaZMsFODAEoEQBbwMlMQa2Qwmxd_jGsKd)

- First, the external VM or the spine/leaf VMs will see the BGP route as it is exposed on the ovn dummy interface of rack-1-host-2 in our case (Host-1 in the figure):

```
$ vagrant ssh spine-1
$ sudo vtysh -c "show ip route" 
...
B>* 172.24.4.92/32 [20/0] via fe80::4638:39ff:fe00:2, swp1, weight 1, 04:50:24
  *                       via fe80::4638:39ff:fe00:4, swp2, weight 1, 04:50:24
...
```

- Then, traffic will go through one of the leafs and from there to the host where the VM is (rack-1-host-2)
- It will hit the ip rule, and then the br-ex routing table will be used, redirecting the traffic to br-ex.
- br-ex (1.1.1.1/32) will ARP for VM IP, which will reach ovn-controller via br-int and this will reply with VM MAC address

If the traffic was in the other direction (from VM to spine), the steps would be the next:

- VM sends the ARP request for the destination IP
- Local ovn-controller will forward the ARP request to br-ex as it does not know about it
- Br-ex will reply to the ARP request with its own MAC address (as we have proxy ARP enabled)
- icmp request will arrive then to br-ex, and will rewrite the destination MAC address to that of the br-ex interface. Then default ECMP route through eth1/eth2 will send the traffic to the Leaf(s).

### VM on different compute on the provider network

The traffic between VMs on provider networks on different computes is depicted in the next figure:

![](https://lh5.googleusercontent.com/avFCiYbTPFZLzDhbXShHZqNTXK3H745bPcy7CuHNr-4dodwZr9cc1wYfpVVsCcPcW-Jlte3tgNoSiWGY6n6uifngRm_PAl5j3xXrDmbzwv7ceuwfPLPkCQXAakq_vYZydMow4tgs)

The flows are exactly the same as before. The first part (VM to leaf) is the same as before for the VM to spine, with the difference that this time the ovn-controller will reply with the MAC address of the destination VM, as it knows about it. Then the request will arrive to br-ex, get the destination MAC rewritten and sent to the leaf. BGP will be in charge of redirecting the traffic to the host where the destination VM is. And the process to redirect the traffic to the OVN overlay is the same as in the previous case.

### VM on the same compute on the provider network

This case is simpler, as the request will be sent directly to the VM tap interface and never hit the kernel or the physical interfaces, since the VM is directly connected to the br-int bridge.

## VMs on Tenants Networks

To test the BGP connectivity for VMs on tenant networks, we are first going to create a new router and a new network

```
$ vagrant ssh rack-1-host-1  # Let’s use the controller for triggering the OpenStack commands
$ source ~/devstack/openrc admin demo
$ openstack router create router-test
$ openstack network create private-test
$ openstack subnet create --subnet-pool shared-default-subnetpool-v4 --network private-test private-test-subnet
$ openstack router set --external-gateway public router-test
$ openstack port list --router router-test
+--------------------------------------+------+-------------------+-----------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                          | Status |
+--------------------------------------+------+-------------------+-----------------------------------------------------------------------------+--------+
| b6329de2-f2a7-44ba-8f19-d238acd6df11 |      | fa:16:3e:bf:93:93 | ip_address='172.24.4.163', subnet_id='ef0ebaa1-f64f-4e1a-845e-a2d832d3468e' | ACTIVE |
|                                      |      |                   | ip_address='2001:db8::20', subnet_id='838ed0ad-3ba2-4242-8a7c-f4a6b250443e' |        |
+--------------------------------------+------+-------------------+-----------------------------------------------------------------------------+--------+
```

After attaching the router to the external provider network the ovn cr-lrp port gets created, which in turn triggers the **PortBindingChassisCreatedEvent** on the chassis where that port is created. Then the agent detects that the event is for a cr-lrp port and does the next (in our case in rack-1-host-1):

- (BGP) expose the cr-lrp ip by adding it to the ovn dummy interface:

```
$ vagrant ssh rack-1-host-1
$ ip a sh ovn
13: ovn: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovn-bgp-vrf state UNKNOWN group default qlen 1000
    link/ether 62:ec:ea:09:a8:c9 brd ff:ff:ff:ff:ff:ff
    inet 172.24.4.163/32 scope global ovn
       valid_lft forever preferred_lft forever
```

- Add the rule needed to redirect incoming traffic to the cr-lrp to br-ex routing table, and from there the default route will send it to br-ex ovs bridge:

```
$ vagrant ssh rack-1-host-1
$ ip rule
0:      from all lookup local
1000:   from all lookup [l3mdev-table]
32000:  from all to 172.24.4.163 lookup br-ex
32766:  from all lookup main
32767:  from all lookup default

$ ip route show table br-ex
default dev br-ex scope link 
172.24.4.163 dev br-ex scope link
```

After that, we attach the subnet to the router, which triggers the **SubnetRouterAttachedEvent** event, but only the node with the associated router cr-lrp port (as shown before, rack-1-host-1 in our case) will process the event and add the rule and route associated to the subnet CIDR:

```
$ vagrant ssh rack-1-host-1
$ source ~/devstack/openrc admin demo
$ openstack router add subnet router-test private-test-subnet
$ $ openstack port list --router router-test
+--------------------------------------+------+-------------------+-----------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                          | Status |
+--------------------------------------+------+-------------------+-----------------------------------------------------------------------------+--------+
| 41389d96-6a70-4147-954c-a15a7678367b |      | fa:16:3e:34:30:23 | ip_address='10.0.0.65', subnet_id='8584f8db-c7f7-46b4-b829-eef795354ea4'    | ACTIVE |
| b6329de2-f2a7-44ba-8f19-d238acd6df11 |      | fa:16:3e:bf:93:93 | ip_address='172.24.4.163', subnet_id='ef0ebaa1-f64f-4e1a-845e-a2d832d3468e' | ACTIVE |
|                                      |      |                   | ip_address='2001:db8::20', subnet_id='838ed0ad-3ba2-4242-8a7c-f4a6b250443e' |        |
+--------------------------------------+------+-------------------+-----------------------------------------------------------------------------+--------+

$ ip rule
0:      from all lookup local
1000:   from all lookup [l3mdev-table]
32000:  from all to 172.24.4.163 lookup br-ex
32000:  from all to 10.0.0.65/26 lookup br-ex
32766:  from all lookup main
32767:  from all lookup default

$ ip route show table br-ex                                                                                                                                                              
default dev br-ex scope link 
10.0.0.64/26 via 172.24.4.163 dev br-ex 
172.24.4.163 dev br-ex scope link
```

Last step is to actually create the VM. This will trigger the event **TenantPortCreatedEvent**. Which will only be processed in the node with the cr-lrp port as the traffic needs to be injected into OVN overlay through that port. In our case this means that the VM IP will be exposed by the agent through rack-1-host-1:

```
$ vagrant ssh rack-1-host-1
$ source ~/devstack/openrc admin demo
$ openstack server create --flavor m1.nano --network private-test --availability-zone nova:rack-2-host-1 --image cirros-0.5.1-x86_64-disk --security-group sg-testing vm-tenant
$ openstack server list
+--------------------------------------+--------------+--------+-----------------------------------+--------------------------+---------+
| ID                                   | Name         | Status | Networks                          | Image                    | Flavor  |
+--------------------------------------+--------------+--------+-----------------------------------+--------------------------+---------+
| 46ed9470-82bd-4c7f-b42f-e783c3daf231 | vm-tenant    | ACTIVE | private-test=10.0.0.95            | cirros-0.5.1-x86_64-disk | m1.nano |
| b5895d69-c4ed-46f9-91ae-e34297fc67ab | vm1-provider | ACTIVE | public=2001:db8::21c, 172.24.4.92 | cirros-0.5.1-x86_64-disk | m1.nano |
+--------------------------------------+--------------+--------+-----------------------------------+--------------------------+---------+
$ ip a sh ovn
13: ovn: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovn-bgp-vrf state UNKNOWN group default qlen 1000
    link/ether 62:ec:ea:09:a8:c9 brd ff:ff:ff:ff:ff:ff
    inet 172.24.4.163/32 scope global ovn
       valid_lft forever preferred_lft forever
    inet 10.0.0.95/32 scope global ovn
       valid_lft forever preferred_lft forever
```

Note we have selected this time a different host (rack-2-host-1). Same as before, related to the traffic, let's follow the round trip for 3 different cases:

- External traffic, from spine VM
- VM on different compute on the tenant network: no need to cover as it works exactly the same as without all the BGP configurations.
- VM on same compute on the tenant network: no need to cover as it is the same as before

### External traffic, from spine VM

Due to the centralized nature of OVN N/S routing, traffic coming through the external network to a tenant network must traverse an OVN gateway node. This is the reason to expose the cr-lrp port. And the traffic will go through it as highlighted on the next figure:

![](https://lh6.googleusercontent.com/O8p1PPcq9F9kmpGqdjmasEcSTml-y890wA76-SUqb6jbjFcevUEnVtam9zyZwCCLfJbZfYxyFqD-DtKlcXPvrcx57lkNdAnCSEdj9_C6XSla5KmmgDOzP8nFeXGBzyhjP2xQi4-_)

- Similarly to the provider case, the external VM or the spine/leaf VMs will see the BGP route as it is exposed on the ovn dummy interface of rack-1-host-1 in our case (Host-2 in the figure):

```
$ vagrant ssh spine-1
$ sudo vtysh -c "show ip route" 
...
B>* 10.0.0.95/32 [20/0] via fe80::4638:39ff:fe00:2, swp1, weight 1, 03:33:26
  *                     via fe80::4638:39ff:fe00:4, swp2, weight 1, 03:33:26

...
```

- Then, traffic will go through one of the leafs and from there to the host where the CR-LRP for the router where the VM subnet is connected to (rack-1-host-1)
- It will hit the ip rule, and then the br-ex routing table will be used, redirecting the traffic to br-ex via the cr-lrp ip (172.24.4.163 in this case).
- The ARP request to the gateway IP will be send to br-ex but ovn-controller would have not answer to those requests as they come from a different broadcast domain (br-ex loopback IP is 1.1.1.1/32). However the agent added a static ARP entry for the gateway (172.24.4.163). This way, the traffic will flow from br-ex to br-int where the ovn router pipeline gets executed:

```
$ vagrant ssh rack-1-host-1
$ ip nei
172.24.4.163 dev br-ex lladdr fa:16:3e:bf:93:93 PERMANENT
```

- Traffic gets redirected to the hypervisor that host the VM, using the geneve tunnel (L3). The traffic will traverse the reverse path.

If the traffic was in the other direction (from VM to spine), the steps would be the next:

- VM sends the ARP request for the destination IP
- As the VM does not have a FIP associated, the flow gets redirected through the tunnel to the network gateway port
- Local ovn-controller will forward the ARP request to br-ex as it does not know about it
- Br-ex will reply to the ARP request with its own MAC address (as we have proxy ARP enabled)
- icmp request will arrive then to br-ex, and will rewrite the destination MAC address to that of the br-ex interface. Then the default ECMP route through eth1/eth2 will send the traffic to the Leaf(s).

## VM on Tenant Networks with FIPs

Finally, we will add a floating IP to the previously created tenant VM, and see how the traffic goes directly to the node where the VM is (instead of through the node with the cr-lrp port).

```
$ vagrant ssh rack-1-host-1  # Let’s use the controller for triggering the OpenStack commands
$ source ~/devstack/openrc admin demo
$ openstack floating ip create public
	...
| floating_ip_address | 172.24.4.79                          |
...
$ openstack port list | grep 10.0.0.95
| c46f95fa-6070-4364-bad7-d7ef8e95dd06 |          | fa:16:3e:0b:99:2b | ip_address='10.0.0.95', subnet_id='8584f8db-c7f7-46b4-b829-eef795354ea4'         | ACTIVE |
$ openstack floating ip set --port c46f95fa-6070-4364-bad7-d7ef8e95dd06 172.24.4.79
```

This triggers the **FIPSetEvent** event, which gets only processed by the agent running on the chassis where the VM port (not the FIP) is associated. Similarly to the provider VM case, the next is generated on the node that host the VM (rack-2-host-1):

- (BGP) expose the ip by adding it to the ovn dummy interface:

```
$ vagrant ssh rack-2-host-1
$ ip a sh ovn
19: ovn: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovn-bgp-vrf state UNKNOWN group default qlen 1000
    link/ether c2:56:95:2c:82:8f brd ff:ff:ff:ff:ff:ff
    inet 172.24.4.79/32 scope global ovn
       valid_lft forever preferred_lft forever
```

- Add the rule needed to redirect incoming traffic to br-ex routing table, and from there the default route will send it to br-ex ovs bridge:

```
$ vagrant ssh rack-1-host-2
$ ip rule
0:      from all lookup local
1000:   from all lookup [l3mdev-table]
32000:  from all to 172.24.4.79 lookup br-ex
32766:  from all lookup main
32767:  from all lookup default

$ ip route show table br-ex
default dev br-ex proto static
```

Finally, related to the traffic, let's follow the round trip for 3 different cases:

- External traffic, from spine VM
- VM on different compute on the tenant network: no need to cover as it is the same as for the VMs on the provider networks
- VM on same compute on the tenant network: no need to cover as it is the same as for the VMs on the provider networks

### External traffic, from spine VM

Unlike for the VM tenant IP, once the FIP is associated to it, the traffic no longer needs to traverse the node with the cr-lrp, and it behaves in the same way as for the VMs on the provider networks, as highlighted in the next figure.

![](https://lh3.googleusercontent.com/BFHl3CIEiTYIdjQu_n5WxQ25kFcWyxChPy9Z05PTIYQNiDQ4dwezQ8HAUYOiZaaYElC4J9eiza6ZpCtUMWmqMKAq5jxu9gZXOEKv43P_11zZ-h6fp6DVVuPzn0munznbOdIJHkxN)
