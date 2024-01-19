---
layout: post
title: "OpenStack Networking with EVPN"
date: "2021-06-25"
categories: BGP
---

This is a continuation of the previous blog post series regarding [OpenStack Networking with BGP](../../02/04/openstack-networking-with-bgp). Now we are covering the multitenancy aspects by using BGP in conjunction with EVPN/VXLAN. One of the main use cases for this is allowing connectivity between tenants/VMs running on different edges/clouds -- on top of OpenStack or not.

## EVPN at OVN BGP Agent

In order to add support for EVPN, we have extended the functionality of the [OVN-BGP Agent](https://github.com/luis5tb/bgp-agent), with a different "mode" for EVPN. To enable it, we simply select the appropriate driver at the bgp-agent.conf file (instead of the default osp\_ovn\_bgp\_driver for BGP mode):

```
$ cat bgp-agent.conf
[DEFAULT]
debug=True
reconcile_interval=120
driver=osp_ovn_evpn_driver

$ sudo bgp-agent --config-dir bgp-agent.conf
Starting BGP Agent...
```

### What does the agent do in EVPN mode?

In the BGP mode the agent exposed the IPs by adding the VM ip into a dummy interface associated with a VRF. The VRF device, where the dummy interface is linked to, is not connected to anything and was only used for exposing routes through BGP (see [this blog post](../../02/04/ovn-bgp-agent-in-depth-traffic-flow-inspection) for more detailed information). By contrast, in the EVPN mode, the VRF is used for the traffic associated with that tenant and for the EVPN. More specifically, the VRFs are not associated to OpenStack tenants, but to Router Gateway Ports, meaning that if a tenant has several Neutron routers connected to the provider network, it will have a different VRF associated with each one of them.

To configure the EVPN the agent creates the next:

- **_VRF device_**, using the VNI number for the **_table_** number associated to it, as well as for the name suffix: vrf-1001 for vni 1001

    `ip link add vrf-1001 type vrf table 1001`

- **_VXLAN device_**, using the VNI number as the **_vxlan id_**, as well as in the name suffix: vxlan-1001

    `ip link add vxlan-1001 type vxlan id 1001 dstport 4789 local LOOPBACK_IP nolearning`

- **_Bridge device_**, where the vxlan device will be connected, and belonging to the previously created vrf; also using the VNI number as name suffix: br-1001

    `ip link add name br-1001 type bridge stp_state 0`

    `ip link set br-1001 master vrf-1001`

    `ip link set vxlan-1001 master br-1001`

- **_Dummy device_**, where the VM IPs will be added to be exposed through BGP. Also associated to the previously created vrf, and using the VNI number as name suffix: lo-1001

    `ip link add name lo-1001 type dummy`

    `ip link set lo-1001 master vrf-1001`

With that, the agent is able to expose/withdraw VM IPs through BGP EVPN by adding/removing them from the dummy device. However, the agent also needs to ensure the FRR configuration is the right one to be able to expose those IPs to other peers. To do that the agent **_reconfigures FRR_** by using the **_vtysh shell_**. It connects to the existing FRR socket (--vty\_socket option) and executes the next commands by passing them through a file (-c FILE\_NAME option):

```
ADD_VRF_TEMPLATE = '''
vrf \{\{ vrf_name \}\}
 vni {\{\ vni \}\}
exit-vrf

router bgp \{\{ bgp_as \}\} vrf \{\{ vrf_name \}\}
 address-family ipv4 unicast
   redistribute connected
 exit-address-family
 address-family ipv6 unicast
   redistribute connected
 exit-address-family
 address-family l2vpn evpn
   advertise ipv4 unicast
   advertise ipv6 unicast
 exit-address-family

'''

DEL_VRF_TEMPLATE = '''
no vrf \{\{ vrf_name \}\}
no router bgp \{\{ bgp_as \}\} vrf \{\{ vrf_name \}\}

'''
```

Finally, the agent needs to ensure the traffic arriving to the node can be redirected from the VRF to the OVN overlay and vice versa. To do that, the agent also performs the next actions:

- Add the VRF device to the OVS provider bridge (e.g., br-ex)

    `ovs-vsctl add-port br-ex vrf-1001`

- Add routes into the VRF associated routing table (1001 in the above example) for both the cr-lrp IP (router gateway port IP) and the subnet CIDR so that the traffic is redirected to the OVS provider bridge (e.g., br-ex)

  ```
  $ ip route show vrf vrf-1001
  10.0.0.0/26 via 172.24.4.146 dev br-ex
  172.24.4.146 dev br-ex scope link
  ```

- Ensure there is no route on the VRF table for the VM IPs pointing to the dummy device

  ```
  $ ip route show vrf table 1001 | grep local
  10.0.0.5 dev lo1001
  $ ip route delete local 10.0.0.5 dev 1001 table 1001
```

- Add an ovs flow into the OVS provider bridge to redirect the traffic back to the proper VRF, based on the subnet CIDR and the router gateway port MAC address.

  ```
  $ ovs-ofctl add-flow br-ex cookie=0x3e7,priority=1000,ip,in_port=1,dl_src:ROUTER_GATEWAY_PORT_MAC,nw_src=SUBNET_CIDR, actions=mod_dl_dst:BR_EX_MAC,output=5
  ```

### Where to run the Agent?

For the BGP mode, the VMs on provider networks or with Floating IPs were exposed directly on the node where the VM was created. Therefore there was a need for running the agent in all the nodes. However, the EVPN mode targets to expose the VMs on tenant networks (on their respective EVPN/VXLAN). Given that in OpenStack with OVN networking the N/S traffic to the tenant VMs (without FIPs) needs to go through the networking nodes (the ones hosting the Neutron Router Gateway Ports, i.e., the cr-lrp ovn ports), there is no need to deploy the agent in all the nodes. Only the nodes that are able to host router gateway ports (cr-lrps), i.e., the ones tagged with the "enable-chassis-gw". As a result, the VM IPs will be advertised through BGP/EVPN in one of those nodes. From those it will follow the normal path to the OpenStack compute node where the VM is allocated -- the Geneve tunnel.

### How EVPN Routing Works?

Next figure shows how the traffic flow goes to/from the VM when exposed through BGP EVPN.

![](https://lh6.googleusercontent.com/5wfSgSOOZYRBnQzlGf63UEgLRoTdh_J5-2LclqMK1yL83FtQmKizKw1mU61oX-PXB7wltc1clKfABvnzjJfDLjl38i1ZqeEVfmI1TJcNX2zHdRP7qee4Sc31R4E_1aVp9cyvfvU0)

The IPs of both the router gateway port (cr-lrp, 172.24.1.20), as well as the IP of the VM itself (20.0.0.241/32) gets added to the dummy device (lo-101) associated to the vrf (vrf-101) which was used for defining the BGPVPN (vni 101). That together with the other devices created on the VRF (vxlan-101 and br-101), and with the FRR reconfiguration ensure the IPs get exposed in the right EVPN. This allows the traffic to reach the node with the cr-lrp, but the traffic needs to be redirected to the OVN Overlay. To do that the VRF is added to the br-ex ovs provider bridge, and two routes are added to redirect the traffic going to the network (20.0.0.0/24) through the CR-LRP port to the br-ex ovs bridge. That will inject the traffic properly into the OVN overlay, which will redirect it through the geneve tunnel (by the br-int ovs flows) to the compute node hosting the VM. The reply from the VM will come back through the same tunnel. However an extra ovs flow needs to be added to br-ex to ensure the traffic is redirected back to the VRF (vrf-101) if the traffic is coming from the exposed network (20.0.0.0/24). To that end, the next rule is added:

```
cookie=0x3e6, duration=4.141s, table=0, n_packets=0, n_bytes=0, priority=1000,ip,in_port="patch-provnet-c",dl_src=fa:16:3e:b7:cc:47,nw_src=20.0.0.0/24 actions=mod_dl_dst:1e:8b:ac:5d:98:4a,output:"vrf-101"
```

It matches the traffic coming from the cr-lrp port from br-int (in\_port="patch-provnet-c"), with the MAC address of the cr-lrp port, i.e., the router gateway port (dl\_src=fa:16:3e:b7:cc:47) and from the exposed network (nw\_src=20.0.0.0/24). And if it matches, it changes the MAC by the one on br-ex device (mod\_dl\_dst:1e:8b:ac:5d:98:4a), and redirect the traffic to the vrf device (output:"vrf-101").

### How does it get triggered?

As for the BGP mode, the agent also watches the OVN Shouthbound Database and triggers actions depending on the events detected. The agent in EVPN mode watches the same type of events but with some caveats, such as not reacting to FIPs events (FIPSetEvent and FIPUnsetEvent), as well as looking for slightly different parameters on them:

- **PortBindingChassisCreatedEvent** and **PortBindingChassisDeletedEvent**: Detects when a port of type **“chassisredirect”** gets attached to an OVN chassis. This is the case for neutron gateway router ports (CR-LRPs). Then the EVPN devices are created (as explained above), FRR gets reconfigured, and the IP gets added to the dummy device.
- **SubnetRouterAttachedEvent** and **SubnetRouterDetachedEvent**: Detects when a port of type **"patch"** gets created and has information about the VNI to use, or gets updated and information about the VNI has been added. This means a subnet is attached to a router and needs to be exposed (due to the attachment or due to adding the VNI information to an already attached one). If the chassis is the one hosting the CR-LRP port for that router where the port is getting created, then the event is processed by the agent. The agent adds the needed ip routes for the subnet as well as the ovs flows to redirect to/from OVN overlay.
- **TenantPortCreatedEvent** and **TenantPortDeletedEvent**: Detects when a port of type **""** or **"virtual"** gets updated (chassis added) or deleted. If that port is not on a provider network and the chassis where the event is detected has the LRP for the network where that port is located (meaning is the node with the CR-LRP for the router where the port’s network is connected to), then the event is processed and the port IP is (BGP) exposed by adding it to the dummy interface associated to the VRF, but only if the EVPN information is present on the related subnet (LRP port). Note the local route being added pointing to the dummy interface on the VRF for that VM IP is removed so that the traffic can get redirected properly to the OVN overlay (instead of the dummy device).

### BGPVPN API integration

We covered how the Agent works internally, the events it reacts to, and how the network traffic flow looks like. Now let's review the API side: how users can expose their tenant networks and how the needed information is translated and stored into the OVN SB DB so that the agent can detect it and process it accordingly.

For the API we saw that the [Neutron BGPVPN project](https://docs.openstack.org/networking-bgpvpn/latest/) has the APIs that we need:

- It has an Admin API to define the BGPVPN properties, such as the VNI or the BGP AS to be used, and to associate it to a tenant.
- It has a Tenant API to allow the users to associate the BGPVPN to a router or to a network.

 As regard to the architecture, the BGPVPN has 3 main components:

- **BGPVPN API:** This is the component that enables the association of RT/VNIs to tenant network/routers. It creates a couple of extra DBs on Neutron to keep the information. This is the component we want to fully leverage, perhaps even restricting some of the APIs.
- **Service plugin Driver** (e.g., bagpipe driver): This is the component in charge of triggering the needed extra actions (RPCs) to notify the backend driver about the changes needed. In our case it should be a simple driver that just integrates with OVN (OVN NB DB) to ensure the information gets propagated to the corresponding OVN resource in the OVN Southbound database -- by adding the information into the external\_ids field.
- **Backend driver** (e.g., bagpipe-bgp): This is the backend driver running on the nodes, in charge of configuring the networking layer based on the needs (e.g., RPC handling, ovs-flows, …). In our case the ovn-bgp-agent will be the backend driver, and it will still be consuming information from the OVN SB DB (reading the extra information at external\_ids), instead of relying on RPC. It will then add the needed OVS flows, configure the kernel routing (bridge, vxlan, vrf, ip routes, …), and configure FRR accordingly to those DB events, as reviewed previously in this blog post.

Currently BGPVPN only supports ML2/OVS (bagpipe driver) but we can easily integrate our agent into its architecture. The next figure highlights the integration points:

![](https://lh4.googleusercontent.com/BLihx7NptTjJFbeesYqJCeVHY4PPmdAVl67PSgH_jAG3k-s9JUWUFTynzSzEtqXvUtyGR4ClArLzTCH9UzDYH6IQEgm2-a12kaJBTcqWREQgpWflMkrCoO6jcl8P4BbQMhD2QSci)

The only need is for a **_service plugin driver (BGPVPN OVN Driver)_** that reacts to the **_BGPVPN API calls_** and adds/removes the EVPN information into/from the OVN DBs so that our agent can consume it. The ML2/OVN driver already copies the external\_ids information of the ports from the Logical\_Switch\_Port table at the OVN SB DB into the Port\_Binding table at the OVN SB DB, which is what our agent consumes. Thus the new OVN service plugin driver for BGPVPN only needs to annotate the relevant ports at the Logical\_Switch\_Port table with the required EVPN information (BGP AS number and VNI number) on the external\_ids field. That gets translated into the OVN SB DB at the Port\_Binding external\_ids field. The OVN BGP Agent reacts to it by configuring the extra ovs flows at the provider bridge, creating the needed devices (vxlan, bridges, vrfs, routes) and reconfiguring FRR.

In relation with the API actions, when the tenant associates the BGPVPN to a network, the BGPVPN OVN service plugin driver annotates that information into:

- For the specific network: External\_ids field of the Network router interface port on the Logical\_Switch\_Port table. The patch port associated with the lrp port.
- For the router where the network is connected: External\_ids field of the Router Gateway Port on the Logical\_Switch\_Port table. The patch port associated with the cr-lrp/lrp port.

If the tenant associates the router instead of the network, the process is similar, but the driver annotates the external\_id into all the network router interface ports connected to the router, i.e., for all the networks connected to the Neutron router being associated to the BGPVPN.

### How to use it without BGPVPN

If we want to use the agent without the BGPVPN API we can do so by doing the above mentioned actions by hand, i.e., by adding the relevant information into the external\_ids of the router and network ports. For example:

```
(overcloud) $ openstack port list --router router
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------------+--------+
| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                                           | Status |
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------------+--------+
| 5bc97046-6483-4e5a-98db-0e616acfac07 |      | fa:16:3e:7f:59:7f | ip_address='172.24.100.61', subnet_id='30845b27-aaec-4a6f-82b4-177908bebd02'                 | ACTIVE |
| 64e99514-6e13-43c9-a11f-7861bf6f75a4 |      | fa:16:3e:8d:c4:51 | ip_address='20.0.0.1', subnet_id='281a316f-6575-4b4c-a9a6-c3660ea8dbb3'                      | ACTIVE |
+--------------------------------------+------+-------------------+----------------------------------------------------------------------------------------------+--------+

# At OVN NB DB
# ovn-nbctl set logical_switch_port 5bc97046-6483-4e5a-98db-0e616acfac07 external_ids:"neutron_bgpvpn\:vni"=1001
# ovn-nbctl set logical_switch_port 64e99514-6e13-43c9-a11f-7861bf6f75a4 external_ids:"neutron_bgpvpn\:vni"=1001
```
