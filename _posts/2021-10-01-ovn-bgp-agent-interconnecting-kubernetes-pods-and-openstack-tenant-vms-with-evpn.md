---
layout: post
title: "OVN BGP Agent: Interconnecting Kubernetes pods and OpenStack tenant VMs with EVPN"
date: "2021-10-01"
categories: BGP
---

In a previous blog post I presented the [OVN-BGP-Agent and its EVPN capabilities.](../../06/25/openstack-networking-with-evpn) Now we are going to use it to showcase how it can be leveraged to provide connectivity in a multicloud environment without the needs for floating IPs. Note the agent codebase was moved from [my personal github account](https://github.com/luis5tb/bgp-agent) to an upstream opendev project: https://opendev.org/x/ovn-bgp-agent

## Multi Cloud Environment

The environment is composed of 2 OpenStack environments. The main objective is to test the tenant to tenant multicloud connectivity where those OpenStack deployments are connected through a (VM) router, in different AS, as in the next figure:

![](https://ltomasbo.files.wordpress.com/2021/10/bgpvpn-for-ovn-bgp-agent-4.jpg?w=960)


## RDO-OSP cloud

The first deployment is an RDO-master deployment (TripleO-based). It uses BGP for the OpenStack control plane (HA), enabling each of the 3 controllers to be on different racks. It is deployed on an Spine-Leaf topology which includes two Top of Racks per each node (2 leafs), also having ECMP capabilities -- default route of each compute is an ECMP route to both leafs.

This setup is running the [ovn-bgp-agent](http://In a previous blog post I presented the OVN-BGP-Agent and its EVPN capabilities. Now we are going to use it to showcase how it can be leveraged to provide connectivity in a multicloud environment without the needs for floating IPs. Note the agent codebase was moved from my personal github account to an upstream opendev project: https://opendev.org/x/ovn-bgp-agent) containerized in all the nodes. In the controller nodes in EVPN mode, and in the compute nodes in BGP mode. In this blog post we focus on the functionality provided by the ones in EVPN mode.

## DevStack cloud

In order to have a more heterogeneous deployment, we deployed a different (external) OpenStack, in a different AS (65001). This deployment is based on DevStack, it is single node, and it does not have ECMP support -- just one NIC connecting the VM to the router where the spine-leaf deployment is connected to.

In addition to that, the DevStack deployment is configured to set up a Kubernetes cluster, with [Kuryr-kubernetes](https://github.com/openstack/kuryr-kubernetes) to ensure that Kubernetes pods are directly plugged into OpenStack networks as Neutron ports.

On top of this DevStack deployment we manually installed and configured the ovn-bgp-agent in EVPN mode, with the objective of allowing the kubernetes pods in OpenStack tenant networks in this deployment to reach the VMs on tenant networks on the other OpenStack deployment (and vice versa) -- if exposed in the same BGPVPN, without the need for floating IPs.

Note both OpenStack deployments have networking-bgpvpn installed/enabled, with [this work in progress patch](https://review.opendev.org/c/openstack/networking-bgpvpn/+/803161/) to ensure support for OVN at networking-bgpvpn. More information about this support/integration effort is explained [the previous blog post](2021-06-25-openstack-networking-with-evpn).

## Demo

The next video shows the connectivity checks from one deployment to the other with EVPN -- by using different BGPVPNs.

<iframe width="507" height="286" src="https://www.youtube.com/embed/ieis0fLX_SY" title="MultiCloud-EVPN-Connectivity" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>


The left side represents the tripleO based RDO-OpenStack cloud, while the right side represents the DevStack OpenStack cloud. The tripleO based RDO-OpenStack cloud has 2 VMs running (vm-tenant and vm2-tenant) in 2 different tenant networks (private and private2 respectively), which are using exactly the same IP (20.0.0.100) to also showcase the overlapping IPs support thanks to EVPN. In this OpenStack there are 2 BGPVPNs created, one for each one of the networks where the VMs are connected, with VNIs 1001 and 1003, respectively.

In the DevStack OpenStack deployment we have a demo pod (just running a web server application replying on port 8080) on top of the kubernetes deployment. Thanks to Kuryr the pod is directly plug into the OpenStack networking (using a Neutron port, with IP 10.0.0.84). Therefore exposing the OpenStack network/router where the pod is connected with networking-bgpvpn makes the pod reachable in the specified EVPN from the remove OpenStack cluster.

### Demo script:

- Minute 1:40: we show there is no connectivity from within that pod (on DevStack deployment) to any of the VMs on the other OpenStack deployment.
- Minute 1:50 - 3:40: we check (with tcpdump) that the right VM is getting the traffic after exposing the router associated to the pod with networking-bgpvpn, on the desired EVPN/VNI (in this case 1001, therefore enabling connectivity towards vm-tenant). First we create the bgpvpn with vni 1001 (as an admin) and then we associate it to the router used by the Kubernetes deployment (as a tenant).
- Minute 3:40: check there is connectivity form the VM to the pod too
- Minute 4:00: check the removal of the bgpvpn association (or the whole bgpvpn) breaks the connectivity as expected.
- Minute 4:35: create a new BGPVPN (same name) with a different VNI, in this case the same as the one associated to vm2-tenant (ie., 1003). This makes the connectivity to work again, but instead of reaching the previous VM (vm-tenant), the traffic goes to the vm2-tenant VM (traffic on tcpdump), as this is the one on the same VNI (1003).
