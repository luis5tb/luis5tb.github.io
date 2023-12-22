---
layout: post
title:  "Deploy OpenStack on OpenShift with Networker Nodes"
date:   2023-12-22 12:02:57 +0100
categories: OpenStack
---
In this post I'll cover the steps to deploy an environment with
[install_yamls](https://github.com/openstack-k8s-operators/install_yamls),
with two edpm computes where one of them acts as a **networker node**,
instead of having them either distributed in all the compute nodes or in
the controllers. This means that the N/S traffic for VMs without Floating
IPs will be centralized in that networker node.


## Deploy CRC

Following the steps detailed at
[install_yamls](https://github.com/openstack-k8s-operators/install_yamls)
the first step is to clone the repository and deploy CRC:

```bash
$ git clone https://github.com/openstack-k8s-operators/install_yamls.git

$ cd install_yamls/devsetup
$ CPUS=12 MEMORY=25600 DISK=100 make crc
```

## Deploy OpenStack

Once CRC is ready we can proceed to first create 2 extra VMs (edpm_computes)
and then install openstack control plane on top of CRC

```bash
# Add default network to CRC
$ make crc_attach_default_interface

# Create compute nodes
$ EDPM_TOTAL_NODES=2 make edpm_compute

# Create dependencies
$ cd ..
$ make crc_storage
$ make input

# Install openstack-operator
$ NETWORK_ISOLATION=true  make openstack_wait

# Install openstack control plane
$ NETWORK_ISOLATION=true  make openstack_wait_deploy
```

## Deploy compute node

As we only want to use one of the nodes as a compute, leaving the second one
to be a networker, we don't need to add the EDPM_TOTAL_NODES when deploying
edpm part

```bash
$ make edpm_wait_deploy
```

This will deploy the OpenStack dataplane only considering the`edpm-compute-0`
node.

## Deploy networker node

To create the networker node we can simply mimic the same
`OpenStackDataPlaneNodeSet` and `OpenStackDataPlaneDeployment` objects
created for the compute node, with some modifications and reconfigure
`OpenStackControlPlane` to ensure the Gateway configuration is not set on
the ovn-controllers running on the CRC side.

### OpenStackDataPlaneNodeSet

The NodeSet is pretty similar to the one for `edpm-compute-0` but:

* pointing to the other node (`edpm-compute-1`)
* removing the extra services not needed for networker nodes. For instance,
  the components to create VMs are not needed (such as `libvirt`, `nova`),
  or the related services such as the `metadata-agent` related items.
* ensuring the next config option is set

  `edpm_enable_chassis_gw: true`

The easy way to parse it is to copy the one already created for
`edpm-compute-0` and modify it. But of course it can be created from scratch
too, or copy the default template for the
[networker node at the dataplane-operator](https://github.com/openstack-k8s-operators/dataplane-operator/blob/main/config/samples/dataplane_v1beta1_openstackdataplanenodeset_networker.yaml)
and adapt it.

```bash
# make a copy of openstackdataplanenodeset
$ oc -n openstack get openstackdataplanenodeset -oyaml > networker_nodeset.yaml

# adapt it to cover the above points
$ cat networker_nodeset.yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
metadata:
  Name: openstack-edpm-networker
  Namespace: openstack
...
spec:
  nodeTemplate:
    ansible:
      ansiblePort: 22
      ansibleUser: cloud-admin
      ansibleVars:
        ...
        edpm_enable_chassis_gw: true
        ...
  ...
  nodes:
    edpm-compute-1:
      ansible:
        ansibleHost: 192.168.122.101
      hostName: edpm-compute-1
      networkConfig: {}
      networks:
      - defaultRoute: true
        fixedIP: 192.168.122.101
        name: CtlPlane
        subnetName: subnet1
      - name: InternalApi
        subnetName: subnet1
      - name: Storage
        subnetName: subnet1
      - name: Tenant
        subnetName: subnet1
  services:
  - repo-setup
  - download-cache
  - configure-network
  - validate-network
  - install-os
  - configure-os
  - run-os
  - ovn
```

### OpenStackDataPlaneDeployment

Create the `OpenStackDataPlaneDeployment` that points to the above
defined `OpenStackDataPlaneNodeSet`:

```bash
$ cat networker_deployment.yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneDeployment
metadata:
  name: openstack-edpm-networker
  namespace: openstack
spec:
  nodeSets:
  - openstack-edpm-networker

```

### Trigger the networker deployment

Finally, create those two objects and wait until the second node
(`edpm-compute-1`) gets configured as a networker node

```bash
$ oc apply -f networker_nodeset.yaml
$ oc apply -f networker_deployment.yaml
```

You can check that the edpm services (i.e., pods) running in `edpm-compute-0`
and `edpm-compute-1` are different. There are less services running on
`edpm-compute-1`. In addition, the `ovs` configuration is different, and you
can check that only `edpm-compute-1` (our networker node) has the 
`enable-chassis-as-gw` flag:

```bash
$ [root@edpm-compute-1 ~]# ovs-vsctl list open .
...
external_ids        : {hostname=edpm-compute-2, ovn-bridge=br-int, ovn-bridge-mappings="provider1:br-ex,provider2:br-vlan", ovn-chassis-mac-mappings="provider1:3e:09:74:7a:7e:0b,provider2:3e:6b:74:7a:7e:0b", ovn-cms-options=enable-chassis-as-gw, ovn-encap-ip="172.19.0.102", ovn-encap-type=geneve, ovn-match-northd-version=True, ovn-monitor-all=True, ovn-ofctrl-wait-before-clear="8000", ovn-openflow-probe-interval="60", ovn-remote="tcp:172.17.0.30:6642", ovn-remote-probe-interval="60000", rundir="/var/run/openvswitch", system-id="bfe1edcd-e698-4a89-bdf9-c2800b6ae0c3"}
...
```

### Reconfigure OpenStackControlPlane

To ensure only `edpm-compute-1` is our networker node, we need to ensure the
control plane is also configured to not be a networker node. This can be
checked at the `OpenStackControlPlane` object

```bash
$ oc get OpenStackControlPlane -oyaml
...
    ovn:                                              
      enabled: true                             
      template:                                      
        ovnController:                       
          debug:                                
            service: false                              
          external-ids:                         
            enable-chassis-as-gateway: true
            ovn-bridge: br-int
            ovn-encap-type: geneve              
            system-id: random 
```

In the case the flag `enable-chassis-as-gateway` is set to true, we need to
change it to false:

```bash
$ oc patch OpenStackControlPlane openstack-galera-network-isolation -n openstack --type='json' -p='[{"op": "replace", "path": "/spec/ovn/template/ovnController/external-ids/enable-chassis-as-gateway", "value": false}]'
```

After that, the flag will be disabled also at the ovn-controllers running on
top of CRC, and only the `edpm-compute-1` will work as a networker node,
hosting the OVN gateway ports and centralizing the traffic for VMs without
Floating IPs.


## Conclusions

As you can see, creating a networker node is exactly the same as creating a
compute node, with the difference of a couple of extra configurations, and a
limited amount of services running on then. In this example we just have 1 
compute and 1 networker, but as you can imaging, you can create as many as you
want/need.