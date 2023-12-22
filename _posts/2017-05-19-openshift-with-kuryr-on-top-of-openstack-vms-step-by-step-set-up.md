---
title: "OpenShift with Kuryr on top of OpenStack VMs: step by step set up"
date: "2017-05-19"
categories: OpenStack
layout: post
---

As motivated in previous blog posts ([Superfluidity series](../../../../research/2017/01/16/superfluidity-containers-and-vms-at-the-mobile-network-part-1)) there is a need to handle both containers and VMs at the Edge Clouds. There are several reasons to do so, among others: VMs are more secure than containers, but containers are faster to boot up and quick reaction is needed at the edge, but so is multi-tenancy. Not all the applications can be containerized, definitely not at the same time; the hardware resources at the edge are more limited than at the core, so higher density is beneficial; ...

In the previous blog posts, we also present Kuryr as the key-enabler allowing the coexistence of containers and VMs (especially layer 2 communication between them). We have worked on ways to automate the deployment steps to make it easier for the user to test Kuryr benefits. This blog post presents them in a step by step guide to deploy OpenStack in a server, and configure an OpenShift deployment with Kuryr as the SDN component on top of that (i.e., OpenStack VMs). It covers the following steps:

- Install OpenStack (Ocata) with RDO
- Extra configuration steps
- Deploy of OpenShift environment with Heat
- Configure master VM to execute openshift-ansible playbooks and execute it
- Configure OpenShift docker-registry and router services
- Configuration of DNS at the server for the OpenShift router service
- Deployment example

### Install OpenStack Ocata

For this deployment, we will use a 64 GB server with CentOS 7.3. This time, unlike in previous blog posts, we have chosen RDO as the tool to install the Ocata release of OpenStack. We are going to follow the steps described at [quickstart](https://www.rdoproject.org/install/quickstart) with a couple of modifications.

First we create the root keys and install the necessary packages:

```
# ssh-keygen -t rsa

# yum install -y https://rdoproject.org/repos/rdo-release.rpm
# yum install -y centos-release-openstack-ocata
# yum update -y
# yum install -y openstack-packstack
```

Then we create a packstack answer file to perform some modifications over the default one. We have to include the HEAT and LBAAS components, but we do not need others, such as SWIFT or the monitoring components as we are not going to use them in this demo:

`# packstack --gen-answer-file packstack-ocata.txt`

Then, in that file, we change the following configuration information:

```
> CONFIG_SWIFT_INSTALL=n
> CONFIG_CEILOMETER_INSTALL=n
> CONFIG_AODH_INSTALL=n
> CONFIG_GNOCCHI_INSTALL=n
> CONFIG_HEAT_INSTALL=y
> CONFIG_CINDER_VOLUMES_SIZE=50G
> CONFIG_NOVA_SCHED_RAM_ALLOC_RATIO=3
> CONFIG_NOVA_LIBVIRT_VIRT_TYPE=kvm
> CONFIG_LBAAS_INSTALL=y
> CONFIG_NEUTRON_METERING_AGENT_INSTALL=n
```

Finally, we execute the packstack command with the modified version of the answer-file to perform the OpenStack installation:

`#  packstack --answer-file packstack-ocata.txt`

### Extra configuration steps

Once the previous command has (successfully) finished, a few extra steps are needed. First we enable the **OVS firewall** in order to gain some performance, specially for the nested cases. To do this, we need to include the following settings at _/etc/neutron/plugins/ml2/openvswitch_agent.ini_

```
[securitygroup]
firewall_driver = openvswitch
```

We also need to enable the **trunk ports** support by including '_trunk_' at the service_plugins to load at _/etc/neutron/neutron.conf:_

```
[DEFAULT]
service_plugins=trunk,...
```

The next step is to ensure that the **DNS services** are properly configured:

```
# Edit /etc/neutron/dhcp_agent.ini to include the dns servers to use
dnsmasq_dns_servers = 8.8.8.8,8.8.4.4 # Or the ones you want to use

# Add dns to extension_drivers in the [ml2] section of /etc/neutron/plugins/ml2/ml2_conf.ini.
[ml2]
extension_drivers = port_security,dns

# Set dhcp_domain at /etc/nova/nova.conf
dhcp_domain=superfluidity

# Set dns_domain at /etc/neutron/neutron.conf (same as in dhcp_domain at nova.conf)
dns_domain=superfluidity
```

Once this is done, we restart the affected services. For simplicity (and to avoid skipping rebooting a service that needs reboot) we just do:

```
systemctl restart openstack-nova-*
systemctl restart neutron-*
```

We also configure the br-ex so that the created public network (floating) IPs are reachable:

```
$ sudo ip link set br-ex up
$ sudo ip route add 172.24.4.0/24 dev br-ex
$ sudo ip addr add 172.24.4.1/24 dev br-ex
```

Finally, the last configuration step is to increase (or remove) some of the **quota limitations** not to block the VMs/Containers creation. To do this, we use the admin Openstack credentials to remove the instances, cores, ports and ram quota limitations for the demo tenant that we will use for the OpenShift deployment:

```
# source keystone_admin
(keystone_admin)# openstack quota set --instances -1 --cores -1 --ports -1 --ram -1 demo
```

### Deploy OpenShift environment with Heat

The OpenShift environment, consisting of VMs, networks and subnets, security groups, trunk ports, ..., will be created by using a Heat template that we have created for commodity. So, the first steps is to clone the git repository:

`# git clone https://github.com/danielmellado/kuryr_heat.git`

Inside that repository, there is a file (hot/parameters.yaml) with the default configurable options that should be updated based on our needs. For instance:

```
parameters:
 image: centos7
 master_flavor: m1.medium
 worker_flavor: m1.large
 public_net: public
 vm_net_cidr: 10.11.0.0/24
 vm_net_gateway: 10.11.0.1
 pod_net_cidr: 10.10.0.0/16
 pod_net_gateway: 10.10.0.1
 service_net_cidr: 172.30.0.0/16
 service_net_gateway: 172.30.255.254
 service_router_port_ip: 172.30.255.254
 service_lbaas_ip: 172.30.0.1
 master_num: 2
 worker_num: 2

resource_registry:
 OS::Kuryr::PodInVMNetworking: networking_deployment.yaml
 resources:
   worker_nodes:
     hooks: pre-create
   trunk_ports:
     hooks: pre-delete
```

There are 4 main points to notice:

1. The information about subnets IPs ranges for VMs, pods and kubernetes services that will be used for the Kubernetes/OpenShift services.
2. There is no current support for trunk ports at Heat and this is needed for the nested pod deployment with Kuryr. Therefore we need to make use of Heat hooks to stop the heat stacking process after the ports are created and make those ports into parent ports of a trunk (for the worker VMs) and then continue with the Heat stacking process. Note that the trunk support is being added and therefore this could be removed in the future: _https://blueprints.launchpad.net/heat/+spec/support-trunk-port_
3. There are 2 master and 2 workers, but as many as desired can be added. We will use the 2 master as the _etcd_ and _Kubernetes/OpenShift master_ nodes, respectively.
4. The desired image will becentos7, so we will need to download it and make it available for the OpenStack demo user:

```
# # Download the centos image to be used
# wget https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1703.qcow2

# # get OpenStack demo user credentials
# source keystone_demo

# # Import image into OpenStack
(keystone_demo)# openstack image create --container-format bare --disk-format qcow2 --file CentOS-7-x86_64-GenericCloud-1703.qcow2 centos7
```

There is another point to look at it in the git repository: the **kuryr_heat** script. Basically this is a tool that we will use to perform the stack/unstack actions, it performs the extra steps needed for the Heat hooking, trunk ports creation and resuming the stacking.

Once everything is in place, we can trigger the stack by using the kuryr_heat script like this:

```
# Execute the stack with: Kuryr_heat stack NAME_OF_THE_STACK
(keystone_demo)# ./kuryr_heat stack OpenShift_Deployment
```

And wait some time for the stack to be completed:

```
(keystone_demo)# openstack stack list
+--------------------------------------+----------------------+-----------------+----------------------+--------------+
| ID                                   | Stack Name           | Stack Status    | Creation Time        | Updated Time |
+--------------------------------------+----------------------+-----------------+----------------------+--------------+
| 80f54720-80c5-4ce1-94e0-5762dd65dfc8 | OpenShift_Deployment | CREATE_COMPLETE | 2017-05-17T08:38:32Z | None         |
+--------------------------------------+----------------------+-----------------+----------------------+--------------+
```

## Configure master VM to execute openshift-ansible playbooks and execute it

When the stack is completed, the next step is to connect to one of the master VMs, let's say master-0, and execute the openshift-ansible playbook so that OpenShift is installed into the master/worker VMs. To do that, we need to login onto master-0, after we obtain the key generated by the kuryr_heat:

```
(keystone_demo)# ./kuryr_heat getkey OpenShift_Deployment > ~/VMs_key.pem
(keystone_demo)# chmod 600 VMs_key.pem
```

Then we just use the floating IP assigned to master-0 and ssh into it with the VMs_key.pem:

```
(keystone_demo)# openstack server list | grep master-0

# Use the floating IP to ssh into the master-0 VM
(keystone_demo)# ssh -l centos -i ~/VMs_key.pem FLOATING_IP
[centos@master-0 ~]$
```

Inside that VM, we can see that the openshift-ansible repository has already been cloned. Note that this is not the upstream version but a modified version where the kuryr roles have been included (_https://github.com/celebdor/openshift-ansible_, branch: _kuryr_rebase_). These modifications basically do:

- install the different kuryr components: kuryr-k8s-controller for the master, kuryr-cni for the worker nodes
- ensure openshift is using kuryr-cni instead of the default one
- create the kuryr.conf files, with the proper certificates and the rest of the configuration parameters, such as VMs subnets, pod subnets, security groups, etc.
- disable dns and proxy for openshift origin-nodes
- ...

Before starting the ansible playbook, we need to copy the example inventory file to _/etc/ansible/hosts_ and edit it with the needed information for a proper configuration of OpenShift with kuryr-kubernetes:

```
# Copy the origin (upstream OpenShift) sample file
[centos@master-0 ~]$ cp openshift-ansible/inventory/byo/host.origin.example

[centos@master-0 ~]$ sudo vi /etc/ansible/hosts
[OSEv3:children]
masters
nodes
etcd

[OSEv3:vars]
openshift_use_kuryr=True
openshift_use_openshift_sdn=False
os_sdn_network_plugin_name=cni
kuryr_cni_link_interface=eth0
openshift_use_dnsmasq=False
openshift_dns_ip=10.11.0.2

# Set userspace so that there's no iptables remains
openshift_node_proxy_mode=userspace

ansible_ssh_user=centos
ansible_become=yes
debug_level=2

openshift_deployment_type=origin

openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]

openshift_master_api_env_vars={"ENABLE_HTTP2": "true"}
openshift_master_controllers_env_vars={"ENABLE_HTTP2": "true"}
openshift_node_env_vars={"ENABLE_HTTP2": "true"}

# Disable management of the OpenShift Registry
openshift_hosted_manage_registry=false
# Disable management of the OpenShift Router
openshift_hosted_manage_router=false

# Openstack
kuryr_openstack_auth_url=http://KEYSTONE_IP:5000/v3
kuryr_openstack_user_domain_name=Default
kuryr_openstack_user_project_name=Default
kuryr_openstack_project_id=DEMO_PROJECT_ID
kuryr_openstack_username=demo
kuryr_openstack_password=mypassword
kuryr_openstack_pod_sg_id=DEFAULT_SECURITY_GROUP_ID
kuryr_openstack_pod_subnet_id=POD_SUBNET_ID
kuryr_openstack_worker_nodes_subnet_id=VM_SUBNET_ID
kuryr_openstack_service_subnet_id=SERVICE_SUBNET_ID
kuryr_openstack_pod_project_id=DEMO_PROJECT_ID

[masters]
master-1.superfluidity
[nodes]
worker-0.superfluidity
worker-1.superfluidity
[etcd]
master-0.superfluidity
```

In the hosts file, you need to change the capital words with the actual IDs of your deployment. Most of this information is printed out after the heat stack deployment finishes. And the rest can be obtained through the openstack cli (e.g.: openstack subnet list; openstack security group list; ...) or at the keystone_demo (for the password).

Finally we can execute the playbook and wait for it to complete:

```
[centos@master-0 ~]$ cd openshift-ansible
[centos@master-0 ~]$ ansible-playbook playbooks/byo/config.yml -vvv
```

### Configure OpenShift docker-registry and router services

When the openshift-ansible playbook execution has (successfully) completed, there are still a couple of steps to be made before we finish with the environment configuration. We have disabled the creation of the _docker-registry_ and the _router_ services during the installation, so we have to create them manually. To do that, we execute the next two commands inside the OpenShift master node, i.e., master-1 based on what was defined at the hosts file:

```
(keystone_demo)# openstack server list | grep master-1

# Use the floating IP to ssh into the master-1 VM
(keystone_demo)# ssh -l centos -i ~/VMs_key.pem FLOATING_IP

# To ensure everything works on the kuryr controller side, reboot the service and take a look at the logs
[centos@master-1 ~]$ sudo systemctl restart kuryr-controller
[centos@master-1 ~]$ sudo cat /var/log/kuryr/kuryr-controller.log

# Create the docker-registry service
[centos@master-1 ~]$ sudo oadm registry --config=/etc/origin/master/admin.kubeconfig --service-account=registry
--> Creating registry registry ...
 serviceaccount "registry" created
 clusterrolebinding "registry-registry-role" created
 deploymentconfig "docker-registry" created
 service "docker-registry" created
--> Success

# Create the router service
[centos@master-1 ~]$ sudo oadm policy add-scc-to-user hostnetwork -z router
[centos@master-1 ~]$ sudo oadm router router --replicas=1 --config=/etc/origin/master/admin.kubeconfig --service-account=router
--> Creating router router ...
 serviceaccount "router" created
 clusterrolebinding "router-router-role" created
 deploymentconfig "router" created
 service "router" created
--> Success

# Wait for them to become Running
[centos@master-1 ~]$ oc get pods

[centos@master-1 ~]$ oc get pods
NAME                     READY STATUS  RESTARTS AGE
docker-registry-1-zddch  1/1   Running 0        2m
router-1-qsrlh           1/1   Running 0        1m

[centos@master-1 ~]$ oc get svc
NAME             CLUSTER-IP     EXTERNAL-IP PORT(S)                 AGE
docker-registry  172.30.196.14  <none>      5000/TCP                2m
kubernetes       172.30.0.1     <none>      443/TCP,53/UDP,53/TCP   1h
router           172.30.152.127 <none>      80/TCP,443/TCP,1936/TCP 1m
```

We have to make some extra modifications at the docker daemon at the worker nodes to ensure they can use the docker-registry service. For this demo, we simply allow the insecure registry option towards the just created docker-registry service. To do this, the steps are the same for both workers:

```
[centos@master-1 ~]$ ssh worker-0
[centos@worker-0 ~]$ sudo vi /etc/sysconfig/docker
# Add this line (using the Cluster IP of your docker-registry, in my case 172.30.196.14):
INSECURE_REGISTRY='--insecure-registry 172.30.196.14:5000'

[centos@worker-0 ~]$ sudo systemctl restart docker
[centos@worker-0 ~]$ exit
```

Then, for the router service, we need to make sure that the worker VM, where the router pod was scheduled, is a member of the newly created (by kuryr-controller) Neutron load balancer for the router:

```
# Get the IP of the host where the router pod is scheduled
[centos@master-1 ~]$ oc describe pod router-1-XXX | grep Node 

# At the server level, get the VM_SUBNET_ID
(keystone_demo)# openstack subnet list | grep vm_subnet

# Add member to the pool for the router loadbalancer
(keystone_demo)# neutron lbaas-member-create --subnet VM_SUBNET_ID --address ROUTER_POD_HOST_IP --protocol-port 80 default/router:TCP:80
(keystone_demo)# neutron lbaas-member-create --subnet VM_SUBNET_ID --address ROUTER_POD_HOST_IP --protocol-port 443 default/router:TCP:443
(keystone_demo)# neutron lbaas-member-create --subnet VM_SUBNET_ID --address ROUTER_POD_HOST_IP --protocol-port 1936 default/router:TCP:1936
```

### Configure DNS at the server to use it for the router

Finally, in order to have the router routes accessible from the outside (the server), we need to assign a floating IP to the Neutron router load balancer, as well as to configure a DNS at the server side. For the floating IP, the steps are:

```
# Obtain the IP given to the router service
[centos@master-1 ~]$ oc get svc | grep router

#Get the port id of the router load balancer port
(keystone_demo)# openstack port list | grep ROUTER_SERVICE_IP

# Create the floating ip association to the router loadbalanacer port
(keystone_demo)# openstack floating ip create --port ROUTER_LOADBALANACER_PORT_ID public
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| created_at          | 2017-05-19T07:04:55Z                 |
| description         |                                      |
| fixed_ip_address    | 172.30.152.127                       |
**| floating_ip_address | 172.24.4.2                           |**
| floating_network_id | 5dc31971-8cf6-4441-be5c-dc9d4f6d5825 |
| id                  | 0251806b-a423-4c46-8166-1d3309545a56 |
| name                | None                                 |
| port_id             | e75cc8bc-f7b1-4ac9-92f1-3e0c3a1741e7 |
| project_id          | 18fbc0e645d74e83931193ef99dfe5c5     |
| revision_number     | 1                                    |
| router_id           | ff91edc0-1f76-4456-b6a2-9974c920e9cb |
| status              | DOWN                                 |
| updated_at          | 2017-05-19T07:04:55Z                 |
+---------------------+--------------------------------------+

# The floating ip will be used to configure the DNS
```

And for the DNS, the steps are:

- Install bind

  `yum install -y bind bind-utils`

- Edit the /etc/named.conf, adding the section in bold:

  ```
  options {
  listen-on port 53 { 127.0.0.1; };
  listen-on-v6 port 53 { ::1; };
  directory "/var/named";
  dump-file "/var/named/data/cache_dump.db";
  statistics-file "/var/named/data/named_stats.txt";
  memstatistics-file "/var/named/data/named_mem_stats.txt";
  allow-query { localhost; };
  allow-transfer { any; };
  ...
  ...
  ...
  **zone "demo.kuryr.org" IN {**
  **type master;**
  **file "kuryr.org";**
  **};**
  ```

- Create a named file (e.g.: /var/named/kuryr.org) with the previously associated floating IP to the router load balancer. This way, different routes can be advertised just using a single floating IP.

  ```
  $TTL 1D
  @ IN SOA @ demo.kuryr.org. (
  0 ; serial
  1D ; refresh
  1H ; retry
  1W ; expire
  3H ) ; minimum
  NS ns.demo.kuryr.org.

  ns.demo.kuryr.org. A 127.0.0.1
  *.demo.kuryr.org. A ROUTER_FLOATING_IP
  ```

- Include the localhost at the _/etc/resolv.conf_ file:

  ```
  nameserver 127.0.0.1
  nameserver 8.8.8.8
  nameserver 8.8.4.4
  ```

- And restart the named service:

  `sudo systemctl start named`

You can test if the setting works properly with the dig tool, e.g.:

  `dig test.demo.kuryr.org @localhost`

The output of that should contain the floating IP associated to the router load balancer.

### Deployment example

To try that everything works, we are going to deploy a new app, test the connectivity, scale it and route it.

- Create a new project and the application:

  ```
  **[centos@master-1 ~]$ oc new-project demo-project**
  Now using project "demo-project" on server "https://master-1.superfluidity:8443".

  You can add applications to this project with the 'new-app' command. For example, try:

  oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

  to build a new example application in Ruby.

  **[centos@master-1 ~]$ oc new-app celebdor/kuryr-demo:latest --name demo**
  --> Found Docker image 37eddba (3 months old) from Docker Hub for "celebdor/kuryr-demo:latest"

  * An image stream will be created as "demo:latest" that will track this image
  * This image will be deployed in deployment config "demo"
  * Port 8080/tcp will be load balanced by service "demo"
  * Other containers can access this service through the hostname "demo"

  --> Creating resources ...
  imagestream "demo" created
  deploymentconfig "demo" created
  service "demo" created
  --> Success
  Run 'oc status' to view your app.

  **[centos@master-1 ~]$ oc get pods**
  NAME         READY STATUS   RESTARTS AGE
  demo-1-gfp0w 1/1   Running  0        43s
  **[centos@master-1 ~]$ oc get svc**
  NAME CLUSTER-IP      EXTERNAL-IP PORT(S)  AGE
  demo 172.30.146.250  <none>      8080/TCP 57s
  **[centos@master-1 ~]$ oc get rc**
  NAME    DESIRED CURRENT READY AGE
  demo-1  1       1       1     1m
  ```

- Test connectivity:

  ```
  **[centos@master-1 ~]$ oc describe pod demo-1-gfp0w | grep IP**
  IP: 10.10.0.4

  # check pod IP
  **[centos@master-1 ~]$ curl 10.10.0.4:8080**
  demo-1-gfp0w: HELLO, I AM ALIVE!!!

  # check service IP
  **[centos@master-1 ~]$ curl 172.30.146.250:8080** 
  demo-1-gfp0w: HELLO, I AM ALIVE!!!
  ```

- Test router routes:

  ```
  **[centos@master-1 ~]$ oc expose service/demo --hostname test.demo.kuryr.org** 
  route "demo" exposed
  **[centos@master-1 ~]$ oc get route**
  NAME  HOST/PORT           PATH SERVICES PORT      TERMINATION WILDCARD
  demo  test.demo.kuryr.org      demo     8080-tcp              None

  # Then at the server check the connectivity towards the just created route
  **(keystone_demo)]# curl test.demo.kuryr.org**
  demo-1-gfp0w: HELLO, I AM ALIVE!!!
  ```

- Test scaling:

  ```
  **[centos@master-1 ~]$ oc scale dc/demo --replicas=3**
  deploymentconfig "demo" scaled
  **[centos@master-1 ~]$ oc get pods**
  NAME         READY STATUS             RESTARTS AGE
  demo-1-6gkvl 0/1   ContainerCreating  0        3s
  demo-1-gfp0w 1/1   Running            0        5m
  demo-1-l69jt 0/1   ContainerCreating  0        3s

  # Check the neutron ports are created at the server (openstack deployment)
  **(keystone_demo)]# openstack port list | grep demo**
  | 088a8e83-3a16-48ba-a0cc-c688180fa352 | demo-1-l69jt | fa:16:3e:cd:fc:15 | ip_address='10.10.0.14', subnet_id='1d203a3b-6114-4941-8f62-06636dccf16c' | ACTIVE |
  | 8f44bdcc-9f11-42d0-8069-e1e499cac7d0 | demo-1-6gkvl | fa:16:3e:22:72:42 | ip_address='10.10.0.10', subnet_id='1d203a3b-6114-4941-8f62-06636dccf16c' | ACTIVE |
  | f0a6ef0f-75d1-44df-9344-abbd14e9f122 | demo-1-gfp0w | fa:16:3e:4c:89:8c | ip_address='10.10.0.4', subnet_id='1d203a3b-6114-4941-8f62-06636dccf16c' | ACTIVE |

  # Test load balancing works
  **(keystone_demo)]# curl test.demo.kuryr.org**
  demo-1-6gkvl: HELLO, I AM ALIVE!!!
  **(keystone_demo)]# curl test.demo.kuryr.org**
  demo-1-l69jt: HELLO, I AM ALIVE!!!
  **(keystone_demo)]# curl test.demo.kuryr.org**
  demo-1-gfp0w: HELLO, I AM ALIVE!!!

  # Each time a different container replies
  ```

- Remove the application:

  `[centos@master-1 ~]$ oc delete project demo-project`

And that's all! Thanks for reading!
