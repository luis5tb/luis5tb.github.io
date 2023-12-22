---
layout: post
title: "How to make the OVN BGP Agent ready for HWOL and OVS-DPDK"
date: "2023-01-09"
categories: BGP
---

Currently, the [OVN BGP Agent](https://opendev.org/x/ovn-bgp-agent) depends on Kernel networking capabilities (ip rules and ip routes) to redirect traffic in/out of OpenStack OVN Overlay. This blocks the option to speed up the connectivity to the VMs exposed through BGP, i.e., it won't work for Hardware Offloading (HWOL) or OVS-DPDK.

This blog post describe how we can make it HOWL and/or OVS-DPDK ready by leveraging OVN capabilities.

## Main Idea

The main idea to make the BGP capabilities of OVN BGP Agent compliant with OVS-DPDK and HWOL is to move what the OVN BGP Agent is currently doing with Kernel networking (to redirect traffic to/from the OpenStack OVN Overlay) to OVN/OVS.

To accomplish that we need to:

1. Ensure the traffic in gets redirected from the physical NICs to _br-int_ (through _br-ex_) without using kernel routing/rules.

2. Ensure the traffic out gets redirected to the physical NICs without using the default kernel routes.

3. Expose the IPs in the same way as we did before.

The third point is simple as it is already being done, but for the first 2 points we would need OVN virtual routing capabilities, ensuring the traffic gets routed from the NICS to the OpenStack Overlay and vice versa.

We propose to have a separate OVN cluster, per node, in charge of setting up the needed virtual infrastructure between the OpenStack networking overlay and the physical network. The next figure shows the extra OVS bridges as well as the OVN virtual networking infrastructure per node.

![](../../../../images/traffic-flows-charts1.jpg?w=960)

In a standard deployment _br-int_ is directly connected to the OVS external bridge (_br-ex_) -- where the physical NICs are attached. By contrast, in a standard BGP deployment, the physical NICs are not directly attached to _br-ex_, but we rely on kernel networking (ip routes and ip rules) to redirect the traffic to _br-ex_. And in this new architecture being proposed, we have:

- _br-int_ connected to an external OVS bridge (_br-provider_)

- _br-provider_ does not have any physical resources attached, just patch ports connecting them to _br-int_ and _br-bgp_

- _br-bgp_ is the integration bridge managed by the extra OVN cluster deployed per node. This is where the virtual OVN resources will be created (routers and switches). It will create mappings to _br-provider_ and _br-ex_ (patch ports)

- _br-ex_ keeps being the external bridge, where the physical NICs are attached. But instead of being directly connected to _br-int_, is connected to _br-bgp_

Regarding the virtual OVN resources we require, we need:

- Virtual router (bgp-router): this is in charge of the routing that was previously done in the kernel networking layer between both networks (physical and OpenStack OVN overlay). It has 2 connections (i.e., Logical Router Ports) towards the bgp-ex Logical Switch to add support for ECMP, and 1 connection to the bgp-provider Logical Switch to ensure traffic to/from the OpenStack networking Overlay.

- Virtual Switch (bgp-ex): it is connected to the bgp-router, and has a localnet to connect it to _br-ex_ and therefore the physical NICs.

- Virtual Switch (bgp-provider): it is connected to the bgp-router, and has a localnet to connect it to _br-provider_ and therefore being able to interact (send traffic back and forth) to the OpenStack OVN overlay.

## Requirements

In order to support 2 different OVN controllers in the same node, talking to the same vswitchd, we need the "multibridge" support introduced in these series: [Support 2+ controllers on the same vswitchd](https://patchwork.ozlabs.org/project/ovn/cover/20221018183150.1213728-1-ihrachys@redhat.com/). This work allows us to have the OpenStack OVN controller (managing _br-int_) and our new OVN controller (managing _br-bgp_) working together.

In addition, to ensure ARP is properly handled (as there is no L2 between computes/controllers) the following modifications to OVN are also needed, which add per logical router port generic ARP proxy support (for any IP): [https://github.com/dceara/ovn/tree/multi-ovn-controller-arp-proxy](https://github.com/dceara/ovn/tree/multi-ovn-controller-arp-proxy)

## Main Steps

Note this is a proof-of-concept, so we are going to detail a list of manual steps to perform per node. The idea is that once the steps are polished we will implement a new driver within the OVN BGP Agent to take care of them. Most of the steps are initialization steps, which do not need to be changed over time. This will also make the part about exposing the IPs simpler, as we won't need all the steps related to kernel networking that are done in the current BGP driver.

The steps are the next:

1. Install the OVN version with the above mentioned functionality, both on the nodes, as well as replacing the ovn\_controller on the OpenStack ovn\_controller containers

2. Start the cluster

    ```bash
    # Start DBS
    $ /usr/share/ovn/scripts/ovn-ctl start\_ovsdb --db-nb-create-insecure-remote=yes --db-sb-create-insecure-remote=yes

    # Start NORTHD
    $ /usr/share/ovn/scripts/ovn-ctl start\_northd

    # Add external-ids information related to the NODE OVN Controller
    $ ovs-vsctl set open . external-ids:ovn-bridge-bgp=br-bgp
    $ ovs-vsctl set open . external-ids:ovn-remote-bgp=unix:/var/run/ovn/ovnsb\_db.sock

    # In case there is an schema mismatch 
    $ sudo ovs-vsctl set open . external-ids:ovn-match-northd-version="false"
    ```

3. Create the topology

    ```bash
    # Create virtual router
    $ ovn-nbctl lr-add bgp-router

    # Create virtual switches
    $ ovn-nbctl ls-add bgp-ex
    $ ovn-nbctl ls-add bgp-provider

    # Export some env vars for simplifying the other commands
    $ export ENP2S0\_MAC=52:54:00:3f:90:97
    $ export ENP3S0\_MAC=52:54:00:fb:69:d4
    $ export LEAF1\_MAC=52:54:00:9e:ac:43
    $ export LEAF2\_MAC=52:54:00:4e:f1:eb
    $ export ROUTER\_MAC=40:44:00:00:00:06
    $ export PROVIDER\_NET=172.16.100.0/24
    $ export ENP2S0\_IP=100.65.3.6/30
    $ export ENP3S0\_IP=100.64.0.6/30
    $ export LEAF1\_IP=100.65.3.5
    $ export LEAF2\_IP=100.64.0.5

    # Create Logical Router Ports to connect to the LS
    $ ovn-nbctl lrp-add bgp-router bgp-router-public $ROUTER\_MAC $PROVIDER\_NET
    $ ovn-nbctl lrp-add bgp-router bgp-router-ex-1 $ENP2S0\_MAC $ENP2S0\_IP
    $ ovn-nbctl lrp-add bgp-router bgp-router-ex-2 $ENP3S0\_MAC $ENP3S0\_IP

    # Ensure ARP Proxy is enabled on the router port connected to the OSP OVN overlay
    $ ovn-nbctl lrp-set-options bgp-router-publicproxy-arp=True

    # Create Logical Switch Ports to connect to the LR
    $ ovn-nbctl lsp-add bgp-provider provider-bgp-router
    $ ovn-nbctl lsp-set-type provider-bgp-router router
    $ ovn-nbctl lsp-set-addresses provider-bgp-router router
    $ ovn-nbctl lsp-set-options provider-bgp-router router-port=bgp-router-public

    $ ovn-nbctl lsp-add bgp-ex ex-bgp-router-1
    $ ovn-nbctl lsp-set-type ex-bgp-router-1 router
    $ ovn-nbctl lsp-set-addresses ex-bgp-router-1 router
    $ ovn-nbctl lsp-set-options ex-bgp-router-1 router-port=bgp-router-ex-1

    $ ovn-nbctl lsp-add bgp-ex ex-bgp-router-2
    $ ovn-nbctl lsp-set-type ex-bgp-router-2 router
    $ ovn-nbctl lsp-set-addresses ex-bgp-router-2 router
    $ ovn-nbctl lsp-set-options ex-bgp-router-2 router-port=bgp-router-ex-2
    ```

4. Add the routing and policies for ECMP

    ```bash
    # Create static mac binding to avoid issues with FRR
    $ ovn-nbctl create static\_mac\_binding  ip="$LEAF1\_IP" logical\_port=bgp-router-ex-1 mac="52\\:54\\:00\\:9e\\:ac\\:43"
    $ ovn-nbctl create static\_mac\_binding  ip="$LEAF2\_IP" logical\_port=bgp-router-ex-2 mac="52\\:54\\:00\\:4e\\:f1:\\eb"

    # Add ECMP routes
    $ ovn-nbctl --ecmp lr-route-add bgp-router 0.0.0.0/0 $LEAF1\_IP
    $ ovn-nbctl --ecmp lr-route-add bgp-router 0.0.0.0/0 $LEAF2\_IP

    # Add Policy to force the traffic out when the destination IP is on the provider network CIDR
    $ ovn-nbctl lr-policy-add bgp-router 10 'inport=="bgp-router-public"' reroute $LEAF1\_IP,$LEAF2\_IP
    ```

5. Connect topology to NICs (physical network) and OpenStack OVN (openstack overlay)

    ```bash
    # Create br-provider bridge (assuming br-ex is already created)
    $ ovs-vsctl add-br br-provider

    # Create the LSP localnet to connect to ovs br-ex and ovs br-provider
    $ ovn-nbctl lsp-add bgp-ex bgp-ex-localnet
    $ ovn-nbctl lsp-set-type bgp-ex-localnet localnet
    $ ovn-nbctl lsp-set-addresses bgp-ex-localnet unknown
    $ ovn-nbctl lsp-set-options bgp-ex-localnet network\_name=bgp-ex

    $ ovn-nbctl lsp-add bgp-provider bgp-provider-localnet
    $ ovn-nbctl lsp-set-type bgp-provider-localnet localnet
    $ ovn-nbctl lsp-set-addresses bgp-provider-localnet unknown
    $ ovn-nbctl lsp-set-options bgp-provider-localnet network\_name=bgp-provider

    # Add physical NICs to OVN/OVS
    $ ovs-vsctl add-port br-ex enp2s0
    $ ovs-vsctl add-port br-ex enp3s0

    # Extra tweaks for proper ARP handling (at physical infrastructure layer)
    $ ip a add $ENP2S0\_IP dev br-ex 
    $ ip a add $ENP3S0\_IP dev br-ex
    $ ip nei replace $LEAF1\_IP lladdr $LEAF1\_MAC dev enp2s0 nud permanent
    $ ip nei replace $LEAF2\_IP lladdr $LEAF2\_MAC dev enp3s0 nud permanent

    # Add the bridge mappings
    # Connect OpenStack OVN cluster to br-provider instead of br-ex
    $ ovs-vsctl set open . external-ids:ovn-bridge-mappings="provider1:br-provider" 
    # Connect in-node OVN cluster to both br-provider and br-ex
    $ ovs-vsctl set open . external-ids:ovn-bridge-mappings-bgp="bgp-provider:br-provider,bgp-ex:br-ex" 

    # Force a local chassis port binding so that patch ports are created between the br-bgp and br-provider/br-ex
    $ ovn-nbctl lrp-set-gateway-chassis  bgp-router-public bgp 1
    ```

6. Start ovn controller, which will create the needed patch ports based on the previous configuration

    ```bash
    # Start ovn controller pointing to the BGP setup
    $ ovn-controller -n bgp
    ```

7. Add extra OVS flows needed for MAC tweaking

    ```bash
    # Incoming traffic need to have its destination mac changed to map the Logic Router Port destination, depending on the NIC (ECMP) that the traffic came in (connected to br-ex)
    $ export patch=\`ovs-ofctl show br-ex | grep patch | cut -d "("  -f1 | xargs\`
    $ ovs-ofctl add-flow br-ex "cookie=0xbadcaf2,ip,nw\_dst=$PROVIDER\_NET,in\_port=enp2s0,priority=100,actions=mod\_dl\_dst:$ENP2S0\_MAC,output=$patch"
    $ ovs-ofctl add-flow br-ex "cookie=0xbadcaf2,ip,nw\_dst=$PROVIDER\_NET,in\_port=enp3s0,priority=100,actions=mod\_dl\_dst:$ENP3S0\_MAC,output=$patch"

    # Outgoing traffic (connectivity between nodes) requires the dst mac to be replaed with the Logical Router Port connected to the br-provider (in\_port is the patch port between br-int and br-provider)
    $ export patch=\`ovs-ofctl show br-provider | grep patch | grep provnet | cut -d "("  -f1 | xargs\`
    $ sudo ovs-ofctl add-flow br-provider "cookie=0xbadcaf3,ip,in\_port=$patch,actions=mod\_dl\_dst:$ROUTER\_MAC,NORMAL"
    ```

## Traffic flow

The next figure shows how the traffic gets out when the destination is another VM on the same provider network, also highlighting what rule/route/flow is hit at each step. As it can be seen, first _br-int_ is replying with the known MAC the arp request. And then, when the traffic is sent through _br-provider_ there is a flow that gets matched and replaces the destination MAC by the LogicalRouterPort MAC (in this case 40:44:00:00:00:06).

When the traffic enters the flows related to in-node OVN setup, the bgp-provider LS sends the traffic to the LRP. The router (bgp-router) pipeline gets then executed and the route policy gets applied, rerouting any traffic coming through bgp-router-public LRP to the IPs of the leafs (ECMP). The traffic is output through one of the LRP connected to the bgp-ex LS (and _br-ex_ bridge). Finally, once in _br-ex_, it simply forwarded through the selected port (ECMP) to the associated physical NIC.

![](../../../../images/traffic-flows-charts7.jpg?w=960)

The next figure shows the same flow but for the external traffic in the case _br-int_ does not know the MAC about the destination IP. Due to that we rely on the proxy arp feature introduced here [https://github.com/dceara/ovn/tree/multi-ovn-controller-arp-proxy](https://github.com/dceara/ovn/tree/multi-ovn-controller-arp-proxy) which ensure that the LRP replies with its own mac. This time, as the IP is not in the provider network range, the OVN (ECMP) route gets applied and redirect the traffic to the leafs. And the rest of the steps are the same as in the previous example.

Note this is the case for external IPs (such as 8.8.8.8) or for IPs belonging to the same network as the provider network (172.16.100.0/24 in the example), but not known by OpenStack -- in the later, in a standard L2 deployment it is the own server replying with it's MAC due to being on the same L2 domain, but in this L3 datacenter (spine/leaf) setup it must be routed instead, meaning it needs to go through the LRP with the arp-proxy enabled.

![](../../../../images/traffic-flows-charts8.jpg?w=960)

## Conclusions and future work

This blog post focuses on detailing the needed steps to make the BGP support currently provided by OVN BGP Agent ready for OVS-DPDK and/or HWOL. The steps described here were executed manually and are considered the base for the future implementation of this support at the OVN BGP Agent, as a new driver.
