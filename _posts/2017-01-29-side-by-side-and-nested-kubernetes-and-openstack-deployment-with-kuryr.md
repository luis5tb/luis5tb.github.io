---
title: "Side-by-side and nested Kubernetes and OpenStack deployment with Kuryr"
date: "2017-01-29"
categories: OpenStack
layout: post
---

Kuryr enables both **_side by side_** Kubernetes and OpenStack deployments, as well as **_nested_** ones where Kubernetes is installed inside OpenStack VMs. As highlighted in previous posts, there is nothing that precludes having a hybrid deployment, i.e., both side by side and nested containers at the same time. Thanks to Kuryr, a higher flexibility for the deployment is achieved, enabling diverse use cases where containers, regardless they are deployed bare metal or inside VMs, are in the same Neutron network as other co-located VMs.

This blog post is a step by step guide to deploy such a hybrid environment. The next steps describe how to create and configure an all-in-one devstack deployment that includes Kubernetes and Kuryr installation. We call this **_undercloud_** deployment. Then, we will describe the steps to create an OpenStack VM where Kubernetes and Kuryr are installed and configured to enabled the nested-containers. This deployment inside the VM is named _**overcloud**_.

## Undercloud deployment

The first step is to clone the devstack git repository:

`$ git clone https://git.openstack.org/openstack-dev/devstack`

We then create the devstack configuration file (local.conf) inside the cloned directory:

```bash
$ cd devstack
$ cat local.conf
[[local|localrc]]

LOGFILE=devstack.log
LOG_COLOR=False

HOST_IP=CHANGEME
# Credentials
ADMIN_PASSWORD=pass
MYSQL_PASSWORD=pass
RABBIT_PASSWORD=pass
SERVICE_PASSWORD=pass
SERVICE_TOKEN=pass
# Enable Keystone v3
IDENTITY_API_VERSION=3

Q_PLUGIN=ml2
Q_ML2_TENANT_NETWORK_TYPE=vxlan

# LBaaSv2 service and Haproxy agent
enable_plugin neutron-lbaas \
 git://git.openstack.org/openstack/neutron-lbaas
enable_service q-lbaasv2
NEUTRON_LBAAS_SERVICE_PROVIDERV2="LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default"

enable_plugin kuryr-kubernetes \
 https://git.openstack.org/openstack/kuryr-kubernetes

enable_service docker
enable_service etcd
enable_service kubernetes-api
enable_service kubernetes-controller-manager
enable_service kubernetes-scheduler
enable_service kubelet
enable_service kuryr-kubernetes

[[post-config|/$Q_PLUGIN_CONF_FILE]]
[securitygroup]
firewall_driver = openvswitch
```

In this file we need to change the HOST_IP value by our IP. Note that OVS-firewall is used (last two lines of the file) to improve the nested container performance.

Finally, we start the devstack deployment with:

`$ ./stack.sh`

This command may take some time. Once the installation is completed, the next steps are focused on the configuration steps as well as the VM creation. As we are going to use the TrunkPort functionality from the Neutron, we need to enable it at neutron.conf configuration file by include 'trunk' at the service_plugins:

```
$ cat /etc/neutron/neutron.conf | grep service_plugins
service_plugins = neutron.services.l3_router.l3_router_plugin.L3RouterPlugin,lbaasv2, trunk
```

And then restart the neutron server (q-svc)  service:

```
$ screen -r
ctrl + a + n (until reaching q-svc tab)
crtl + c
up_arrow + enter
ctrl + a + d (exit)
```

The next step is to use the "demo" tenant created by devstack to create the VM and the Neutron networks to be used:

`$ source openrc demo demo`

We generate the key to log-in into the VMs:

```
$ ssh-keygen -t rsa -b 2048 -N '' -f id_rsa_demo
$ nova keypair-add --pub-key id_rsa_demo.pub demo
```

And create the networks to be used by the VMs and the containers:

```
$ # Create the networks
$ openstack network create net0
$ openstack network create net1

$ # Create the subnets
$ openstack subnet create --network net0 --subnet-range 10.0.4.0/24 subnet0
$ openstack subnet create --network net1 --subnet-range 10.0.5.0/24 subnet1

$ # Add subnets to the router (router1 is created by devstack and connected to the OpenStack public network)
$ openstack router add subnet router1 subnet0
$ openstack router add subnet router1 subnet1
```

Then, we modify the default security group rules to allow both ping and ssh into them:

```
$ openstack security group rule create --protocol icmp default
$ openstack security group rule create --protocol tcp --dst-port 22 default
```

And create the parent port (trunk port) to be used by the overcloud VM:

```
$ # create the port to be used as parent port
$ openstack port create --network net0 --security-group default port0

$ # Create a trunk using port0 as parent port (i.e. turn port0 into a trunk port)
$ openstack network trunk create --parent-port port0 trunk0
```

Finally, we boot the VM by using the parent port just created as well as the modified security group:

```
$ # get a vlan-aware-vm suitable image
$ wget https://download.fedoraproject.org/pub/fedora/linux/releases/24/CloudImages/x86_64/images/Fedora-Cloud-Base-24-1.2.x86_64.qcow2
$ openstack image create --container-format bare --disk-format qcow2 --file Fedora-Cloud-Base-24-1.2.x86_64.qcow2 fedora24

$ # Boot the VM, using a flavor with at least 4GB and 2 vcpus
$ openstack server create --image fedora24 --flavor m1.large --nic port-id=port0 --key-name demo fedoraVM
```

## Overcloud deployment

Once the VM is running, we should log in into it, and similarly to the steps for the undercloud, clone the devstack repository and configure the local.conf file

```
$ sudo ip netns exec `ip netns | grep qrouter` ssh -l fedora -i id_rsa_demo `openstack server show fedoraVM -f value | grep net0 | cut -d'=' -f2`
[vm]$ sudo dnf install -y git
[vm]$ git clone https://git.openstack.org/openstack-dev/devstack
[vm]$ cd devstack
[vm]$ cat local.conf
[[local|localrc]]

RECLONE="no"

enable_plugin kuryr-kubernetes \
 https://git.openstack.org/openstack/kuryr-kubernetes

OFFLINE="no"
LOGFILE=devstack.log
LOG_COLOR=False
ADMIN_PASSWORD=pass
DATABASE_PASSWORD=pass
RABBIT_PASSWORD=pass
SERVICE_PASSWORD=pass
SERVICE_TOKEN=pass
IDENTITY_API_VERSION=3
**ENABLED_SERVICES=""**

**SERVICE_HOST=UNDERCLOUD_CONTROLLER_IP**
**MULTI_HOST=1**
KEYSTONE_SERVICE_HOST=$SERVICE_HOST
MYSQL_HOST=$SERVICE_HOST
RABBIT_HOST=$SERVICE_HOST

enable_service docker
enable_service etcd
enable_service kubernetes-api
enable_service kubernetes-controller-manager
enable_service kubernetes-scheduler
enable_service kubelet
enable_service kuryr-kubernetes
```

Note the MULTI_HOST line to indicate the VM is not a controller node, but it has to connect to the undercloud, whose IP is provided at the SERVICE_HOST. It must also be noted the ENABLED_SERVICES="", which disables all the services by default, so that only the ones later enabled are installed.

Before executing the devstack stack.sh script, we need to clone the kuryr-kubernetes repository and change the plugin.sh script to avoid problems due to not installing neutron components:

```
[vm]$ # Clone the kuryr-kubernetes repository
[vm]$ sudo mkdir /opt/stack
[vm]$ sudo chown fedora:fedora /opt/stack
[vm]$ cd /opt/stack
[vm]$ git clone https://github.com/openstack/kuryr-kubernetes.git
```

In the /opt/stack/kuryr-kubernetes/devstack/plugin.sh we comment out [configure_neutron_defaults](https://github.com/openstack/kuryr-kubernetes/blob/master/devstack/plugin.sh#L453). This method is getting UUID of default Neutron resources project, pod_subnet etc. using local neutron client and setting those values in /etc/kuryr/kuryr.conf. This will not work at the moment because Neutron is running remotely. That is why this is being commented out and we will manually configure them later. After this we can trigger the devstack stack.sh script:

```
[vm]$ cd ~/devstack
[vm]$ ./stack.sh
```

Once the installation is completed, we update the configuration at /etc/kuryr/kuryr.conf to include the information missing:

- Set the UUID of Neutron resources from the undercloud Neutron:

  ```
  [neutron_defaults]
  ovs_bridge = br-int
  pod_security_groups = <UNDERCLOUD_DEFAULT_SG_UUID> (in our case default)
  pod_subnet = <UNDERCLOUD_SUBNET_FOR_PODS_UUID> (in our case subnet1)
  project = <UNDERCLOUD_DEFAULT_PROJECT_UUID> (in our case demo)
  worker_nodes_subnet = <UNDERCLOUD_SUBNET_WORKER_NODES_UUID> (in our case subnet0)
  ```

- Configure "pod_vif_driver" as "nested-vlan":

  ```
  [kubernetes]
  pod_vif_driver = nested-vlan
  ```

- Configure the binding section:

  ```
  [binding]
  driver = kuryr.lib.binding.drivers.vlan
  link_iface = <VM interface name eg. eth0>
  ```

- Restart the kuryr-kubernetes-controller service from the devstack screen (similarly to what we did for q-svc at the undercloud)

## Demo

Now we are ready to test the connectivity between VMs and containers in the hybrid environment just deployed. First we create a container in the undercloud and ping the VM from it:

```
$ cat pod1.yml
apiVersion: v1
kind: Pod
metadata:
 name: busybox-sleep
spec:
 containers:
 - name: busybox
   image: busybox
   args:
   - sleep
   - "1000000"

$ # Create the container using the above template
$ kubectl create -f pod1.yml
$ kubectl get pods
NAME           READY STATUS            RESTARTS AGE
busybox-sleep  0/1   ContainerCreating 0        5s
$ kubectl get pods
NAME           READY STATUS  RESTARTS AGE
busybox-sleep  1/1   Running 0        3m

$ # Get inside the container to ping the VM
$ kubectl exec -it busybox-sleep -- sh
[container]$ ping VM_IP

...CHECK PING WORKS!!...

[container]$ exit
```

Then we log in into the VM and create a similar container inside, and check the connetivity with the other bare metal container, and the VM:

```
[vm]$ # Create the container
[vm]$ kubectl create -f pod1.yml

[vm]$ # Waint until pod becomes active
[vm]$ kubectl exec -it busybox-sleep -- sh
[container]$ ping CONTAINER_IP

...CHECK PING WORKS!!...

[container]$ ping VM_IP

...CHECK PING WORKS!!...

[container]$ exit
```

Finally, we ping both containers from the VM:

```
[vm]$ ping CONTAINER_UNDERCLOUD_IP

...CHECK PING WORKS!!...

[vm]$ ping CONTAINER_OVERCLOUD_IP

...CHECK PING WORKS!!...
```
