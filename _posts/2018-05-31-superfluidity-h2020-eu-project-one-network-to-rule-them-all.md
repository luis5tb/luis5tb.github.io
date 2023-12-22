---
layout: post
title: "Superfluidity H2020 EU Project: One network to rule them all!"
date: "2018-05-31"
categories: Research
---

The [SUPERFLUIDITY project](http://superfluidity.eu/) was a 33 months European (H2020) research project (July 2015 - April 2018) aimed at achieving superfluidity on the Internet: the ability to instantiate services on-the-fly, run them anywhere in the network (core, aggregation, edge) and shift them transparently to different locations. The project especially focused on 5G networks, and tried to go one step further into the virtualization and orchestration of different network elements, including radio and network processing components, such as BBUs, EPCs, P-GW, S-GW, PCRF, MME, load balancers, SDN controllers, and others.

For more information about it, you can visit both the official [project website](http://superfluidity.eu), as well as my [previous blog post](../../../2017/01/16/superfluidity-containers-and-vms-at-the-mobile-network-part-1)

## Integration Work

One of the main achievements of the project was the design of an orchestration framework composed of many different open source projects, enabling management of both VMs and containers -- regardless if the containers are running bare-metal or nested. Among others, the framework included the integration of (see figure below):
 - **VIM level**: OpenStack, Kubernetes, and OpenShift, all of them using Kuryr to ensure VMs and containers L3/L2 network connectivity
 - **VNFM level**: Heat templates, Mistral, Kubernetes/OpenShift templates and Ansible Playbooks 
 - **NFVO level**: OSM and ManageIQ

![Orchestrators](../../../../images/orchestrators.jpg)

One of the main problems addressed by the above framework was enabling container orchestration at any point of the network together with virtual machines (VMs). This orchestration was needed by the different partners, as some of the applications and components were running on VMs and others on containers, depending on their performance, isolation, or network performance needs, to name a few. The problem of having both VMs and containers together is not just how to create computational resources, but also how to connect these computational resources among themselves and to the users, in other words, _networking_.  That was one of the main problems we addressed by extending Kuryr OpenStack project: enabling nested containers (running on top of OpenStack VMs) without double encapsulation.

Another important problem that we addressed at Kuryr was the booting time of the containers when they are connected to the OpenStack Neutron Networks. Note that spending one or two seconds creating and configuring the Neutron ports is not an important issue for VM booting time, but it is of great impact for the container booting time. Thus, a new feature named ports pool was added to pre-create Neutron ports to have them ready for the containers before they booted up, enabling an order of magnitude faster booting times, especially at scale, as well as reducing the load on the Neutron server-side upon containers creation spikes. More information about this feature can be found at [this previous blog post](../../../../openstack/2017/05/09/kuryr-ports-pool-speeding-up-containers-booting-time-on-neutron-networks).

## Demonstrators

We prepared a final demonstrator where different parts of the integration work were shown. Specially, we created a set of scenes where different functionalities were presented:

![scenes.jpg](../../../../images/scenes.png)

We had a distributed cloud (Edge + Core) managed from a central ManageIQ appliance. Notice we moved the Edge servers to the review meeting room!

![edge](../../../../images/edge.jpg)

During the demo, from the ManageIQ appliance we deployed a new mobile network by instantiating the CRAN (containerized) components at Edge -- RRH, BBU, and EPC; then the MEC components (MEO at Core, TOF at Edge); and finally a video streaming application at Core. Once everything was deployed, we tested that a mobile device was able to connect to the just created network and watch the videos being streamed from the Core Cloud, as well as not losing the connectivity while later moving components of the video streaming application from Core to Edge.

The live demo could not be recorded, but here are some pre-recorded demos of some pieces of that demo:

- CRAN deployment: https://drive.google.com/open?id=1TuBwMaiaKJsVOl5bWVo6o5dZheiRqQ3Y
- MEC deployment: https://drive.google.com/open?id=1nluN993g3mNxq6hKwcCc7vcm3Jmyy\_\_7

## Review and lessons learnt

The project was rated with the highest score: '_**excellent**_'. Besides the technical work and the upstream contributions, the project also had other impacts, such as: 30 talks/keynotes presentations, more than 50 scientific research publications, 9 organized events, contributions to standardization bodies such as ETSI NFV ISG, ETSI MEC ISG, OASIS TOSCA, and ISO MPEG, to name a few.

In addition, part of the work was demonstrated at the Keynote of DevConf 2018, Brno:

- https://www.youtube.com/watch?v=BloeOXBsETo&feature=youtu.be

As well as presented at the OpenStack Summit (Vancouver 2018):

- https://www.openstack.org/summit/vancouver-2018/summit-schedule/events/21112/superfluidity-one-network-to-rule-them-all

To sum it up, we accomplished great things and it was a positive experience. It was a bit chaotic to synchronize between all the partners (this was a really big project with 18 partners!) and the reporting overhead was a bit too high, sometimes feeling like you need to spend more time just filling reports rather than doing the actual work (the fun part!). Anyway, it was a nice experience to learn about new problems as well as to figure out how to help addressing issues with already existing technologies -- but adding the needed modifications/extensions that in turn make these technologies better.
