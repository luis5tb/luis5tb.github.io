---
title: "Kuryr Ports Pool: Speeding up containers booting time on Neutron networks"
date: "2017-05-09"
categories: OpenStack
layout: post
---

As described in the previous blog post series: [Superfluidity: Containers and VMs in the Mobile Network](../../../../research/2017/01/16/superfluidity-containers-and-vms-at-the-mobile-network-part-1), **kuryr** enables both side-by-side and nested kubernetes and OpenStack deployments, where containers can be created and connected to Neutron networks either on baremetal hosts or inside OpenStack VMs.

One of the main advantages that containers offer over VMs is that they boot up faster. However, when containers are connected to Neutron networks through Kuryr, the communication between the Kuryr controller and the Neutron server may take some considerable time compared to the container booting up individually  (even though it is negligible for VMs). Consequently, there is a need to speed up these actions to boot up containers without any remarkable delay.

## Ports Pool Design

Every time a container is created or deleted, Kuryr makes a call to Neutron to create or remove the port used by the container. Interactions between Kuryr and Neutron may take more time than it is desired from the container management perspective. Fortunately, some of these interactions between Kuryr and Neutron can be optimized.

To tackle this problem, we started working on an approach that tries to **_minimize the Kuryr-Neutron interactions_** by reusing Neutron resources (in this case the ports) that have already been createdand using the Neutron bulk requests. We created a blueprint to organize and gather all the work activities, i.e., design discussions and code patch sets:

[https://blueprints.launchpad.net/kuryr-kubernetes/+spec/ports-pool](https://blueprints.launchpad.net/kuryr-kubernetes/+spec/ports-pool)

The design of the ports pool functionality is being discussed and agreed on at:

[https://review.openstack.org/#/c/427681/](https://review.openstack.org/#/c/427681/)

The proposed solution suggests to create Neutron ports ahead and keep them in a pool rather than to create them during pod life-cycle pipeline, which considerably shortens the waiting time for:

- Creating ports and waiting for them to become active when booting containers
- Deleting ports when removing containers

The design is based on two different pool types:

- **_Available_pools:_** A pool of ports for each tenant, host (or trunk port for the nested case), and security group is ready to be used by the pods.  Note initially there is no pools. The pool is only created whenever a tenant creates a pod at a given host or a VM, with a specific security group(s). It is then populated by a predefined number  of ports with a bulk creation request.
- **_Recyclable_pool:_** Instead of deleting the ports during pods removal, they are moved into this pool. The ports in this pool are later recycled and put back into the corresponding **Available_pools** pool.

To provide this functionality, a new **VIF Pool driver** has been designed (one for the baremetal and one for the nested deployment types) that manages the ports pools upon pods creation and deletion events. It makes sure that at least a certain number of available ports exist in each  pool (i.e., for each security group, host or trunk, and tenant) which already has a pod on it. The ports  in each Available_pool are created in batches, i.e., instead of creating one port at a time, a configurable amount of them are created at once through Neutron bulk calls.

Having the ports ready at the Available_pools during the container creation process will speed up the process. Instead of calling Neutron port_create and then waiting for the activation of the port, it will be taken from the _available_pool_ (hence, no need to call Neutron) and only the port info will be updated later with the proper container name (i.e., call Neutron port_update). Consequently, at least two calls to Neutron can be skipped (to create a port and wait for port to become ACTIVE), in favour of one extra step (the port name update), that is faster than the others. On the other hand, if the corresponding pool is empty, a _ResourceNotReady_ exception is triggered and the pool is repopulated. After that, a port can be taken from that pool and used  for  another pod.

Similarly, the pool driver ensures that ports are regularly recycled after pods are deleted and put back in the corresponding _available_pool_ pool to be reused. Therefore, Neutron calls are skipped  as there is no need to delete and create another port for a future pod. The port cleanup actions return ports to the corresponding available_pool after re-applying security groups and changing the device name to ‘available-port’. A maximum limit for the pool can be specified to ensure that once the corresponding available_pool reach a certain size, the ports above this number get deleted instead of recycled. This upper limit can be disabled by setting it to 0.

## How to enable it

To have either a side-by-side or a nested Kubernetes and OpenStack deployment with Kuryr you can follow the steps described in a previous blog post: [Side-by-side and nested kubernetes and OpenStack deployment with Kuryr](../../01/29/side-by-side-and-nested-kubernetes-and-openstack-deployment-with-kuryr)

Once the desired deployment is up and running, the next step is to fetch the kuryr-kubernetes ports pool patches for either baremetal or nested and configure the related drivers.

The steps for the baremetal case are:

```
# Navigate to the kuryr-kubernetes directory
cd /opt/stack/kuryr-kubernetes

# Download the newest version of the packages
git fetch git://git.openstack.org/openstack/kuryr-kubernetes refs/changes/77/436877/15 && git checkout FETCH_HEAD
sudo python setup.py install

# Edit /etc/kuryr/kuryr.conf to select the proper pool and vif drivers at kubernetes section:
[kubernetes]
pod_vif_driver = generic
vif_pool_driver = generic
```

And the steps for the nested deployment are:

```
# Navigate to the kuryr-kubernetes directory
cd /opt/stack/kuryr-kubernetes

# Download the newest version of the packages
git fetch git://git.openstack.org/openstack/kuryr-kubernetes refs/changes/94/436894/12 && git checkout FETCH_HEAD
sudo python setup.py install

# Edit /etc/kuryr/kuryr.conf to select the proper pool and vif drivers at kubernetes section:
[kubernetes]
pod_vif_driver = nested-vlan
vif_pool_driver = nested
```

Note for both cases you can use _vif_pool_driver = noop_ to disable ports pool. Also, note that we fetch the patch sets 15 and 12 respectively in our procedures. You should always use the latest version available.

In both cases, you need to restart the kuryr-k8s-controller service after you apply the new settings.

Let’s see an example of how this works for the nested deployment. To demonstrate that,  I used a sample busybox container, with the next template:

```
apiVersion: v1
kind: ReplicationController
metadata:
 name: busybox-sleep-perf
spec:
 replicas: 1
 selector:
   name: busybox-sleep-perf
 template:
   metadata:
     labels:
       name: busybox-sleep-perf
   spec:
     containers:
     - name: busybox
       image: busybox
       args:
       - sleep
       - "1000000"
```

We created the replication controller with:

`kubectl create -f busybox_example.yml`

As expected, this creates a pod (as the number of replicas is set to 1), and waits until the pool for the given security_group/tenant is populated. We can see the next message into the kuryr-k8s-controller:

`kuryr_kubernetes.controller.drivers.vif_pool [-] Ports pool does not have available ports!`

This creates a new pool with the default amount of ports in the config variable ‘_ports_pool_batch_’ (the default number is 10 but you can change it in kuryr.conf), triggering a Neutron bulk request for the ports creation and attaching them to the VM trunk port where the pod has been scheduled. For the purpose of this demo, we only have a 1 VM kubernetes deployment, but if a multi-node (VM) kubernetes cluster is used, there will be a different pool per trunk port, i.e., per VM.

The output of Openstack port list should be similar to the following:

```
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
| ID                                   | Name                                              | MAC Address       | Fixed IP Addresses                                  | Status |
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
| 214c1680-4ccf-4829-8408-9e68c04b74f7 | available-port                                    | fa:16:3e:3b:a4:ec | ip_address='10.0.5.13', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 2fc4995c-38ff-41e7-abd6-fae849c581e9 | available-port                                    | fa:16:3e:a4:25:32 | ip_address='10.0.5.12', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 5ed1da34-f278-49d4-aabb-2683befed76b | available-port                                    | fa:16:3e:f1:44:a3 | ip_address='10.0.5.10', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 6d908bca-fa3a-4505-8afb-c312f9d2effb | available-port                                    | fa:16:3e:57:0d:90 | ip_address='10.0.5.20', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 864e6785-a88d-4c79-bd17-09a183b9d42c | available-port                                    | fa:16:3e:57:e9:05 | ip_address='10.0.5.18', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 94cd72e0-80ee-47e1-95cf-302bade6ca50 | available-port                                    | fa:16:3e:1a:70:7b | ip_address='10.0.5.16', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 9620976b-754f-47bc-aaed-32714659321f | available-port                                    | fa:16:3e:69:6c:40 | ip_address='10.0.5.5', subnet_id='f869449e-9fcc-    | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| dae252fd-76c2-4561-8cd7-9e255a3287e4 | available-port                                    | fa:16:3e:c7:70:34 | ip_address='10.0.5.22', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| eb70e52b-d098-4c39-9daa-6f752696f41e | available-port                                    | fa:16:3e:17:42:55 | ip_address='10.0.5.21', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| f22ceab9-c534-440b-9b5e-8bc4b5134420 | busybox-sleep-perf-j2qka                          | fa:16:3e:aa:ea:e3 | ip_address='10.0.5.4', subnet_id='f869449e-9fcc-    | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
```

This time, and with my current deployment, the pod took around 50 seconds to boot up since it had to populate the pool first, which means  to create 10 ports and attach them to the VM as sub-ports, and attach the container to one of these ports..

We can also scale up the replication controller by setting the number of replicas to a higher number, for example to 5. This creates 4 extra pods, but as the pool is already populated, the booting time is now faster, around 15 seconds for all of them, as the ports have already been created and are in the ACTIVE state:

```
$ kubectl scale rc busybox-sleep-perf --replicas=5
replicationcontroller "busybox-sleep-perf" scaled

$ kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
busybox-sleep-perf-19vdf   1/1       Running   0          24s
busybox-sleep-perf-afhn4   1/1       Running   0          24s
busybox-sleep-perf-j2qka   1/1       Running   0          28m
busybox-sleep-perf-jlnq7   1/1       Running   0          24s
busybox-sleep-perf-w196w   1/1       Running   0          24s
$ openstack port list
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
| ID                                   | Name                                              | MAC Address       | Fixed IP Addresses                                  | Status |
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
| 214c1680-4ccf-4829-8408-9e68c04b74f7 | available-port                                    | fa:16:3e:3b:a4:ec | ip_address='10.0.5.13', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 2fc4995c-38ff-41e7-abd6-fae849c581e9 | busybox-sleep-perf-afhn4                          | fa:16:3e:a4:25:32 | ip_address='10.0.5.12', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 5ed1da34-f278-49d4-aabb-2683befed76b | busybox-sleep-perf-19vdf                          | fa:16:3e:f1:44:a3 | ip_address='10.0.5.10', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 6d908bca-fa3a-4505-8afb-c312f9d2effb | available-port                                    | fa:16:3e:57:0d:90 | ip_address='10.0.5.20', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 864e6785-a88d-4c79-bd17-09a183b9d42c | available-port                                    | fa:16:3e:57:e9:05 | ip_address='10.0.5.18', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 94cd72e0-80ee-47e1-95cf-302bade6ca50 | busybox-sleep-perf-jlnq7                          | fa:16:3e:1a:70:7b | ip_address='10.0.5.16', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 9620976b-754f-47bc-aaed-32714659321f | busybox-sleep-perf-w196w                          | fa:16:3e:69:6c:40 | ip_address='10.0.5.5', subnet_id='f869449e-9fcc-    | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| dae252fd-76c2-4561-8cd7-9e255a3287e4 | available-port                                    | fa:16:3e:c7:70:34 | ip_address='10.0.5.22', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| eb70e52b-d098-4c39-9daa-6f752696f41e | available-port                                    | fa:16:3e:17:42:55 | ip_address='10.0.5.21', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| f22ceab9-c534-440b-9b5e-8bc4b5134420 | busybox-sleep-perf-j2qka                          | fa:16:3e:aa:ea:e3 | ip_address='10.0.5.4', subnet_id='f869449e-9fcc-    | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
```

On the other hand, if we reduce the number of replicas, the ports will be returned to the pool after the pod is deleted and the ports get cleaned up. These cleanup actions happen with a given frequency configured at ‘_ports_pool_update_frequency_’. Similarly to the previous command, we can just reduce the amount of replicas to 3 and we see that 2 pods get terminated, and their associated ports get recycled, i.e., become available again to be used by other pods:

```
$ kubectl scale rc busybox-sleep-perf --replicas=3
replicationcontroller "busybox-sleep-perf" scaled

$ kubectl get pods
NAME                       READY     STATUS        RESTARTS   AGE
busybox-sleep-perf-19vdf   1/1       Terminating   0          9m
busybox-sleep-perf-afhn4   1/1       Running       0          9m
busybox-sleep-perf-j2qka   1/1       Running       0          37m
busybox-sleep-perf-jlnq7   1/1       Terminating   0          9m
busybox-sleep-perf-w196w   1/1       Running       0          9m
kubectl get pods

$ kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
busybox-sleep-perf-afhn4   1/1       Running   0          9m
busybox-sleep-perf-j2qka   1/1       Running   0          37m
busybox-sleep-perf-w196w   1/1       Running   0          9m

$ openstack port list
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
| ID                                   | Name                                              | MAC Address       | Fixed IP Addresses                                  | Status |
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
| 214c1680-4ccf-4829-8408-9e68c04b74f7 | available-port                                    | fa:16:3e:3b:a4:ec | ip_address='10.0.5.13', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 2fc4995c-38ff-41e7-abd6-fae849c581e9 | busybox-sleep-perf-afhn4                          | fa:16:3e:a4:25:32 | ip_address='10.0.5.12', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 5ed1da34-f278-49d4-aabb-2683befed76b | available-port                                    | fa:16:3e:f1:44:a3 | ip_address='10.0.5.10', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 6d908bca-fa3a-4505-8afb-c312f9d2effb | available-port                                    | fa:16:3e:57:0d:90 | ip_address='10.0.5.20', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 864e6785-a88d-4c79-bd17-09a183b9d42c | available-port                                    | fa:16:3e:57:e9:05 | ip_address='10.0.5.18', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 94cd72e0-80ee-47e1-95cf-302bade6ca50 | available-port                                    | fa:16:3e:1a:70:7b | ip_address='10.0.5.16', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 9620976b-754f-47bc-aaed-32714659321f | busybox-sleep-perf-w196w                          | fa:16:3e:69:6c:40 | ip_address='10.0.5.5', subnet_id='f869449e-9fcc-    | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| dae252fd-76c2-4561-8cd7-9e255a3287e4 | available-port                                    | fa:16:3e:c7:70:34 | ip_address='10.0.5.22', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| eb70e52b-d098-4c39-9daa-6f752696f41e | available-port                                    | fa:16:3e:17:42:55 | ip_address='10.0.5.21', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| f22ceab9-c534-440b-9b5e-8bc4b5134420 | busybox-sleep-perf-j2qka                          | fa:16:3e:aa:ea:e3 | ip_address='10.0.5.4', subnet_id='f869449e-9fcc-    | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
```

Finally, if we scale it up again, and reach the point where the number of ports in the pool is below the defined minimum at the config variable ‘_ports_pool_min_’, another bulk port creation is triggered and new available-ports are created. As we are using the default values, every time the pool is below 5 ports, a new bulk port creation request for 10 ports is triggered. Therefore, if we scale up the replication controller to 10 pods, the ones already created are used by the newly created pods, but another 10 more ports will be created to support future requests:

```
$ kubectl scale rc busybox-sleep-perf --replicas=10
replicationcontroller "busybox-sleep-perf" scaled

$ kubectl get pods
NAME                       READY     STATUS              RESTARTS   AGE
busybox-sleep-perf-2svkx   0/1       ContainerCreating   0          4s
busybox-sleep-perf-5epnj   0/1       ContainerCreating   0          4s
busybox-sleep-perf-6i415   0/1       ContainerCreating   0          4s
busybox-sleep-perf-72wv2   0/1       ContainerCreating   0          4s
busybox-sleep-perf-afhn4   1/1       Running             0          19m
busybox-sleep-perf-ego9t   0/1       ContainerCreating   0          4s
busybox-sleep-perf-j2qka   1/1       Running             0          47m
busybox-sleep-perf-kpjsy   0/1       ContainerCreating   0          4s
busybox-sleep-perf-um9iq   0/1       ContainerCreating   0          4s
busybox-sleep-perf-w196w   1/1       Running             0          19m

$ kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
busybox-sleep-perf-2svkx   1/1       Running   0          19s
busybox-sleep-perf-5epnj   1/1       Running   0          19s
busybox-sleep-perf-6i415   1/1       Running   0          19s
busybox-sleep-perf-72wv2   1/1       Running   0          19s
busybox-sleep-perf-afhn4   1/1       Running   0          20m
busybox-sleep-perf-ego9t   1/1       Running   0          19s
busybox-sleep-perf-j2qka   1/1       Running   0          48m
busybox-sleep-perf-kpjsy   1/1       Running   0          19s
busybox-sleep-perf-um9iq   1/1       Running   0          19s
busybox-sleep-perf-w196w   1/1       Running   0          20m

$ openstack port list
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
| ID                                   | Name                                              | MAC Address       | Fixed IP Addresses                                  | Status |
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
| 040063a5-c3b4-4b62-8769-08fe19ae29fa | busybox-sleep-perf-ego9t                          | fa:16:3e:66:68:8b | ip_address='10.0.5.30', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 04db3c00-0d70-4f2d-91c5-a26c5c198116 | busybox-sleep-perf-kpjsy                          | fa:16:3e:ea:da:ec | ip_address='10.0.5.17', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 2fc4995c-38ff-41e7-abd6-fae849c581e9 | busybox-sleep-perf-afhn4                          | fa:16:3e:a4:25:32 | ip_address='10.0.5.12', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 3b5f0ab6-269d-4b81-a195-b09c47ff07e4 | available-port                                    | fa:16:3e:c0:9c:39 | ip_address='10.0.5.29', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 42b88def-3c5d-4c65-ae2f-eee4ed755cf7 | available-port                                    | fa:16:3e:11:e9:45 | ip_address='10.0.5.38', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 452e7e4c-b438-4fd8-b426-ef3d73b99c15 | available-port                                    | fa:16:3e:d2:72:40 | ip_address='10.0.5.37', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 69abd6de-d68f-4c62-a4a2-c8931938249b | busybox-sleep-perf-6i415                          | fa:16:3e:95:ae:0c | ip_address='10.0.5.27', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 6d908bca-fa3a-4505-8afb-c312f9d2effb | available-port                                    | fa:16:3e:57:0d:90 | ip_address='10.0.5.20', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 83fa49bf-4693-407a-9e4a-5f2b01abad0b | available-port                                    | fa:16:3e:5e:11:16 | ip_address='10.0.5.11', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 94cd72e0-80ee-47e1-95cf-302bade6ca50 | available-port                                    | fa:16:3e:1a:70:7b | ip_address='10.0.5.16', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 9620976b-754f-47bc-aaed-32714659321f | busybox-sleep-perf-w196w                          | fa:16:3e:69:6c:40 | ip_address='10.0.5.5', subnet_id='f869449e-9fcc-    | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 9d5bd345-141a-4a00-b29c-93f35f525eaa | busybox-sleep-perf-2svkx                          | fa:16:3e:e1:ba:27 | ip_address='10.0.5.26', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| 9e85c926-1ddc-43df-ba76-9e3a463830e4 | busybox-sleep-perf-um9iq                          | fa:16:3e:50:71:8f | ip_address='10.0.5.9', subnet_id='f869449e-9fcc-    | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| d660a796-2269-4cd2-99eb-267975463363 | available-port                                    | fa:16:3e:d8:fc:32 | ip_address='10.0.5.33', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| d7c4ead6-470c-4ac4-bd00-1a45c98dceed | available-port                                    | fa:16:3e:76:36:16 | ip_address='10.0.5.6', subnet_id='f869449e-9fcc-    | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| dae252fd-76c2-4561-8cd7-9e255a3287e4 | available-port                                    | fa:16:3e:c7:70:34 | ip_address='10.0.5.22', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| e2d429a4-450a-4645-9964-7e01b4e7b366 | busybox-sleep-perf-5epnj                          | fa:16:3e:6c:5b:8f | ip_address='10.0.5.25', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| eb70e52b-d098-4c39-9daa-6f752696f41e | available-port                                    | fa:16:3e:17:42:55 | ip_address='10.0.5.21', subnet_id='f869449e-9fcc-   | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| ed595592-9af2-437f-8823-427ad75263e2 | busybox-sleep-perf-72wv2                          | fa:16:3e:a3:71:76 | ip_address='10.0.5.7', subnet_id='f869449e-9fcc-    | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
| f22ceab9-c534-440b-9b5e-8bc4b5134420 | busybox-sleep-perf-j2qka                          | fa:16:3e:aa:ea:e3 | ip_address='10.0.5.4', subnet_id='f869449e-9fcc-    | ACTIVE |
|                                      |                                                   |                   | 4b1d-9865-f1ae6b73f890'                             |        |
+--------------------------------------+---------------------------------------------------+-------------------+-----------------------------------------------------+--------+
```

## Performance evaluation

To conclude this post, I did a performance evaluation to compare both baremetal and nested deployment, with and without the ports pool functionality enabled. I have evaluated the pod creation time (i.e., time to have all pods in a _Running_ status) for a replication controller varying the number of pods from 1, 5, 10, 20, 30, 40 and 50. I did not increase the replica from, let’s say 20 to 30, but created a replica controller with 20 replicas, and once all the pods were running, I deleted that replica controller and created a new one with 30 instead. So, in this example, for the second replica controller, 30 pods are created, not just 10 (different between 30 and 20).

The next figure shows the comparison for the baremetal deployment. We can see that the difference between both mechanisms increases as the number of pods increase because more and more Neutron calls are saved, which in turns reduces the stress on the Neutron server side, speeding up its actions.

![image (1).png](../../../../images/image-1.png)

As for the nested comparison, a similar behavior can be observed in the following figure, but with even bigger differences. The main reasons behind are:

1. Less neutron calls are needed, which means skipping ports creation as well as attaching the ports as subports.
2. Ports are already active for the nested case as they are already connected to the VM. Thus, there is no need to wait for the port to become active, avoiding the pooling to check the port status until it becomes active (i.e., Neutron calls).

![image (2).png](../../../../images/image-2.png)
