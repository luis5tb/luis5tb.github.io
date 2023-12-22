---
title: "Superfluidity: Containers and VMs deployment for the Mobile Network (Part 2)"
date: "2017-01-24"
categories: Research
layout: post
---

Once we have the ‘glue’ between VMs and containers as presented in the [previous blog post](../16/superfluidity-containers-and-vms-at-the-mobile-network-part-1), an important decision is what type of deployment is most suitable for each use case. Some applications (MEC Apps) or Virtual Network Functions (VNFs) may need really fast scaling or spawn responses and require therefore to be run directly on bare metal deployments. In this case, they will run inside containers to take the advantage of their easy portability and the life cycle management, unlike the old-fashioned bare metal installations and configurations. On the other hand, there are other applications and VNFs that do not require such fast scaling or spawn times. On the contrary, they may require higher network performance (latency, throughput) but still retain the flexibility given by containers or VMs, thus requiring a VM with SRIOV or DPDK. Finally, there may be other applications or VNFs that benefit from extra manageability, consequently taking advantage of running in nested containers, with stronger isolation (and thus improved security), and where some extra information about the status of the applications is known (both the hosting VM and the nested containers). This approach also allows other types of orchestration actions over the applications. One example being the functionality provided by Magnum OpenStack project which allows to install Kubernetes on top of the OpenStack VMs, as well as some extra orchestration actions over the containers deployed through the virtualized infrastructure.

To sum it up, a common IT problem is that there is no unique solution to fit all the use cases, and therefore having a choice between side-by-side (some applications running in VMs while others running on bare metal containers) and nested (containers running inside VMs) deployments is a great advantage. Luckily, the latest updates on the Kuryr project give us the possibility to choose any of them based on given requirements.

### Side-by-side OpenStack and OpenShift deployment

In order to enable side by side deployments through Kuryr, a few components have to be added to handle the OpenShift (and similarly the Kubernetes) container creation and networking. An overview of the components is presented in the next image

![](../../../../images/untitled-presentation-1.png)

The main Kuryr components are highlighted in yellow. The Kuryr-Controllers is a service in charge of the interactions with the OpenShift (and similarly Kubernetes) API server, as well as the Neutron one. By contrast, the Kuryr CNI is in charge of the networking binding  for the containers and pods at each worker node, therefore there will be one kuryr CNI instance in each one of them.

The interaction process between these components, i.e., the Kubernetes, OpenShift and Neutron components, is depicted in the sequence diagram (for more details you can see this [presentation](https://docs.google.com/presentation/d/1mofmPHRXzXdTx8ez3H73cegBtTQ_E4AQHnJKJNuEtbw)).

![](../../../../images/untitled-presentation-2.png)

Similarly to Kubelet, the Kuryr-Controller is watching over the OpenShift API server (or Kubernetes API server ). When a user request to create a pod reaches the API server, a notifications is sent to both, Kubelet and the Kuryr-Controller. The Kuryr-Controller then interacts with Neutron to create a Neutron port that will be used by the container later. It calls Neutron to create the port, and notifies the API server with the information about the created port (pod(vif)), while it is waiting for the Neutron server to notify it about the status of the port becoming active. Finally, when that happens, it notifies the API server about it. On the other hand, when Kubelet receives the notification about the pod creation request, it calls the Kuryr-CNI to handle the local bindings between the container and the network. The Kuryr-CNI waits for the notification with the information about the port and then starts the necessary steps to attach the container to the Neutron subnet. These consist of creating  a veth device and attaching one of its ends to the OVS bridge (br-int) while leaving the other end for the pod. Once the notifications about the port being active arrives, the Kuryr-CNI finishes its task and the Kubelet component creates a container with the provided veth device end, and connects it to the Neutron network.

In a side by side deployment, we have an OpenStack deployment on one side (which includes Keystone, Nova, Neutron, Glance, ...) in charge of the VMs operation (creation, deletion, migration, connectivity, ...), and on the other side, we have a deployment in charge of the containers, in this case OpenShift, but it could be Kubernetes as well, or even just raw containers deployed through Docker. An example of this is shown in the next figure, which depicts the environment used for the demo presented in this [video](https://www.youtube.com/watch?v=F909pmf8lbc)

![](../../../../images/untitled-presentation-3.png)

In that demo, a side-by-side deployment of OpenShift and OpenStack was made, where a Neutron subnet with Kuryr for launching VMs and Containers was used. In that figure we can see:

- An OpenStack controller: which includes the main components, and in this case also the Kuryr-Controller, although the later could have been located anywhere else if desired.
- An OpenShift controller: which includes the components belonging to standard OpenShift master role, i.e., the API server, the scheduler, and the registry.
- An OpenStack worker: where the VMs are to be deployed.
- An OpenShift worker: which, besides having the normal components for an OpenShift node, also includes the Kuryr-CNI and the Neutron OVS agent so that created containers can be attached to Neutron networks.
- ManageIQ: In addition to all the needed components, a ManageIQ instance was also present to demonstrate a single pane of glass where we can see both containers and VMs ports being created in the same Neutron network from a centralized view -- even though they are different deployments.

### Nested deployment: OpenShift on top of OpenStack

In order to make Kuryr working in nested environment, a few modifications, extensions, are needed. These modifications were recently merged into the Kuryr upstream branch, both for Docker and Kubernetes/OpenShift support:

- (Docker) [https://review.openstack.org/#/c/402462/](https://review.openstack.org/#/c/402462/)
- (Kubernetes) [https://review.openstack.org/#/c/410578/](https://review.openstack.org/#/c/410578/)

The way the containers are connected to the outside Neutron subnets is by using a new feature included in Neutron, named [Trunk Ports](https://wiki.openstack.org/wiki/Neutron/TrunkPort). The VM, where the containers are deployed, is booted with a Trunk Port, and then, for each container created inside the VM, a new subport is attached to that VM, therefore having a different encapsulation (VLAN) for different containers running inside the VM. They also differ from the own VM traffic, which leaves the VM untagged. Note that the subports do not have to be on the same subnet as the host VM. This thus allows containers both in the same and in different Neutron subnets to be created in the same VM.

To continue the previous example, we keep focusing on the OpenShift/Kubernetes scenario. A few changes were to be made to the two main components described above, Kuryr-Controller and Kuryr-CNI. As for the Kuryr-Controller, one of the main changes is regarding how the ports, which will be used by the containers, are created. Instead of just asking Neutron for a new port, there are two more steps to be performed once the port is created:

- Obtaining a VLAN ID to be used for encapsulating containers traffic inside the VM.
- Calling neutron to attach the created port to the VM's trunk port by using VLAN as a segmentation type, and the previously obtained VLAN ID. This way, the port will be attached as a subport to the VM, and can be later used by the container.

Furthermore, the modifications at the Kuryr-CNI are targeting the new way to bind the containers to the network, as in this case, instead of being added to the OVS (br-int) bridge, they are connected to the VM 's vNIC in the specific vlan provided by the Kuryr-Controller (subport).

For the nested deployment, the interactions as well as the components are mainly the same. The main difference is how the components are distributed. Now, as the OpenShift environment is installed inside VMs, the Kuryr-Controller also needs to run on a VM so that it is reachable from the OpenShift/Kubernetes nodes running in other VMs on the same Neutron network. Same as before, it can be co-located in the OpenShift master VM or anywhere else. With regards to the Kuryr-CNI, instead of being located on the servers, they need to be located inside the VMs actuating as worker nodes, so that they can plug the container to the vNIC on the VM on which they are running.

PS: In a follow up blog post I will include instructions for a step-by-step Kubernetes on top of OpenStack VM deployment with Kuryr, as well as a brief demo to test the containers and VMs connectivity.

To conclude this blog post I just want to emphasize that, thanks to Kuryr, both side-by-side and nested deployment may be used in a single deployment. The only requirement is to install the appropriate services on the server, the VMs or both, depending on where the containers are to be deployed. This enables the VMs, and both bare metal and nested containers to be plugged into the same Neutron networks.
