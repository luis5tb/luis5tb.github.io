---
title: "OpenStack with BGP accelerated with eBPF/XDP"
date: "2022-01-10"
layout: post
categories: BGP
---

In a [previous blog post](../../../2021/02/04/openstack-networking-with-bgp) we described how the [ovn-bgp-agent](https://opendev.org/x/ovn-bgp-agent) works and how the traffic gets routed to the nodes (through BGP) and then redirected to the OVN overlay by leveraging kernel routing (more information about the traffic flow details [here](../../../2021/02/04/ovn-bgp-agent-in-depth-traffic-flow-inspection)).

Once the traffic reaches the node where the VM is, the kernel is in charge of redirecting the traffic to the OVN overlay by using a:

- Set of **ip rules** to enforce using the routing table created for the ovs bridge associated with the Neutron provider network
- Set of **ip routes** inside that routing table to redirect the traffic to the provider network ovs bridge (e.g., br-ex)

In this blog post we are going to focus on a simple Proof of Concept where we **replaced those set of rules and routes by an eBPF/XDP module**. In this way the routing happening at the kernel layer could be speeded up and even offloaded to the NIC if supported.

First let's cover the basic concepts.

## eBPF

eBPF is the extended version of Berkeley Packet Filter (BPF). In a nutshell, it is an abstract virtual machine that runs within the Linux Kernel (more similar to Java Virtual Machine than OpenStack/qemu VMs), and that can run applications in a controller environment, by executing user-defined programs inside a sandbox in the kernel. Most frequent usage is to enable developers to write low-level monitoring or tracing, as well as networking (loadbalancing, DDoS protection, ...).

## XDP

eXpress Data Path (XDP) is a framework that enables high performance packet processing within eBPF applications, by running them as soon as possible, usually immediately after a packet is received by the network interface. XDP programs can be directly attached to network interfaces and whenever a packet is received on that interface, the XDP program receives a callback and perform the related operations.

### XDP Models

- **Generic XDP**: Program loaded into the kernel as part of the ordinary network path. It does not provide the full performance benefit, but provides an easy way to test XDP programs on generic hardware without support for XDP.
- **Native XDP**: XDP program is loaded by the network card driver. Requires support from the network card driver
- **Offloaded XDP:** XDP program is loaded directly on the NIC. Thus it executes without using the node's CPU. This requires support from the network interface device.

### XDP Operations on packets

- **XDP\_DROP:** Drop and does not process the packet. Allows to analyze traffic patterns and filters and then in real time drop specific types of packets (DDoS attack)
- **XDP\_PASS:** Indicates the packet should be forwarded to the normal network stack for further processing. But it allows the XDP program to modify the content before this happens.
- **XDP\_TX:** Forwards the packet (which may have been modified) to the same network interface it was received.
- **XDP\_REDIRECT:** Bypassed the normal network stack and redirects the packet to another NIC on the network. It can also modified the packet before the redirection.

## Usage at ovn-bgp-agent

In our use case we will focus on the **XDP\_REDIRECT**. The idea is that instead of relying on the ip rules and routes to redirect the traffic from the host NIC(s) to be OVS bridge interface (in our case br-ex), we will have an eBPF application loaded on the incomming NIC(s) with XDP.

Of course, not every packet needs redirection to the OpenStack overlay, in our case we only need to redirect when the destination IP are the FIPs or IPs of VMs on the provider network. To do that we have implemented 2 modules. The base code used was taken from https://github.com/xdp-project/xdp-tutorial and adapted for our use case. Code for this PoC is available at https://github.com/luis5tb/xdp-tutorial/tree/bgp-redirect.

### Kernel module

The kernel module is _**xdp\_prog\_kern\_03.c**_. This module gets attached to the NIC by using the xdp\_loader (**_xdp\_loader.c_**). It needs to be loaded to the NIC where the traffic is incoming (note it can be attached to both NICs as in our case we were using ECMP).

First we need to compile both the XDP program, as well as the loader we are going to use to attach it to a NIC:

```
$ clang -O2 -g -Wall -target bpf -c xdp_prog_kern_03.c -o xdp_prog_kern_03.o
$ gcc xdp_loader.c -o xdp_loader ../common/common_params.o ../common/common_user_bpf_xdp.o -lbpf
```

Then we can use the loader to attach the XDP program to a NIC by:

```
$ sudo ./xdp_loader -d enp2s0 -S --filename xdp_prog_kern_03.o --progsec xdp_redirect_map
Success: Loaded BPF-object(xdp_prog_kern_03.o) and used section(xdp_redirect_map)
 - XDP prog attached on device:enp2s0(ifindex:3)
 - Unpinning (remove) prev maps in /sys/fs/bpf/enp2s0/
 - Pinning maps in /sys/fs/bpf/enp2s0/
```

Note we only attach the **xdp\_redirect\_map** function. This function is pretty straightforward:

```c
SEC("xdp_redirect_map")
int xdp_redirect_map_func(struct xdp_md *ctx)
{
	void *data_end = (void *)(long)ctx->data_end;
	void *data = (void *)(long)ctx->data;
	struct hdr_cursor nh;
	struct ethhdr *eth;
	int eth_type;
	int action = XDP_PASS;
	unsigned char *dst;
	struct iphdr *iph;
	__u64 nh_off;

	/* These keep track of the next header type and iterator pointer */
	nh.pos = data;

	/* Parse Ethernet and IP/IPv6 headers */
	eth_type = parse_ethhdr(&nh, data_end, &eth);
	if (eth_type == -1)
		goto out;

	/* Do we know where to redirect this packet? */
	nh_off = sizeof(*eth);
	iph = data + nh_off;
	if (iph + 1 > data_end) {
		action = XDP_DROP;
		goto out;
	}
	dst = bpf_map_lookup_elem(&redirect_params, &iph->daddr);
	if (!dst)
		goto out;

	/* Set a proper destination address */
	memcpy(eth->h_dest, dst, ETH_ALEN);

	action = bpf_redirect_map(&tx_port, 0, 0);
```

As it can be seen, besides the initial variables declaration and packet parsing and checks, it just:

- makes a lookup on the **_redirect\_params_** map, based on the destination IP of the packet, which will return the destination MAC to use (see how this is filled in the next subsection).
- modifies the packet so that the destination MAC is replaced with the one obtained from the **_redirect\_params_** map.
- executes the XDP\_REDIRECT (aka bpf\_redirect\_map) to redirect the traffic to the NIC specified on **_tx\_port_** map (see how this map is filled in the next subsection).

### User-land program

This is the program that will be invoked by the ovn-bgp-agent when new VMs are created on the host. More specifically, the ovn-bgp-agent reacts to VMs creation on the local hypervisor (i.e., chassis), and calls this program to ensure traffic with destination the VM IP is redirected to OVN overlay by the eBPF/XDP program.

This program is invoked with 4 parameters:

- The device in which it will operate (enp2s0 in the next example)
- The device where the traffic needs to be redirected to (br-ex in the next example)
- The destination IP that the traffic should match to be redirected (172.24.100.139 in the next example)
- The destination MAC (fa:16:3e:73:21:74 in the next example) that should be set on the modified packet before being redirected to OVN overlay (br-ex in this case)

The next is an example of the call that ovn-bgp-agent should do:

```
$ sudo ./xdp_prog_user -d enp2s0 -r br-ex --dest-ip 172.24.100.139 --dest-mac fa:16:3e:73:21:74
```

The main idea about this user-land side is that it is the one able to write the list of IPs,MACs pairs that should be used for redirecting the traffic to ONV overlay, and this can be dynamically managed by the user without changing the kernel module. Besides the arguments parsing and checkings, let´s take a look at the main part of the code:

```c
...
if (parse_mac(cfg.dest_mac, dest) < 0) {
	fprintf(stderr, "ERR: can't parse mac address %s\n", cfg.dest_mac);
	return EXIT_FAIL_OPTION;
}
/* Open the tx_port map corresponding to the cfg.ifname interface */
map_fd = open_bpf_map_file(pin_dir, "tx_port", NULL);
if (map_fd < 0) {
	return EXIT_FAIL_BPF;
}

printf("map dir: %s\n", pin_dir);

/* setup a virtual port for the static redirect */
i = 0;
bpf_map_update_elem(map_fd, &i, &cfg.redirect_ifindex, 0);
printf("redirect from ifnum=%d to ifnum=%d\n", cfg.ifindex, cfg.redirect_ifindex);

/* Open the redirect_params map */
map_fd = open_bpf_map_file(pin_dir, "redirect_params", NULL);
if (map_fd < 0) {
	return EXIT_FAIL_BPF;
}

/* Setup the mapping containing MAC addresses */
inet_pton(AF_INET, cfg.dest_ip, &(ip.sin_addr));
inet_ntop(AF_INET, &(ip.sin_addr), some_addr, INET_ADDRSTRLEN);
printf("%s\n", some_addr);
if (write_iface_params(map_fd, &(ip.sin_addr.s_addr), dest) < 0) {
	fprintf(stderr, "can't write iface params\n");
	return 1;
}
```

The first part adds information into the tx\_port map about the redirect mappings between NICs (in our case enp2s0 -> br-ex). Once that is done, the redirect\_params map is opened and the IP (from **_cfg.dest\_ip_**) and MAC (from **_dest_**) are stored into it. This information is the one used by the kernel side to take the redirect decisions.

### Demo

In the video you can see a quick demo where:

- Initial deployment with ovn-bgp-agent working with ip rules and routes. ICMP request can be seen with tcpdump
- Stop the agent and remove the ip rules and routes, and therefore the connectivity is lost. Request visible with tcpdump but not being redirected to the VM, so no reply
- Load the kernel module
- By using the XDP prog\_user, add to the shared map (redirect\_map) information about the neutron port to have its traffic redirected to br-ex (IP: 172.24.100.139, MAC: fa:16:3e:73:21:74).
- Check the connectivity is back, but it cannot be seen in tcpdump as this is processed by XDP with XDP\_REDIRECT and therefore it's redirected to br-ex before tcpdump hooks can see it on the NIC. Note the reply is there, as we only loaded XDP module to redirect the incoming traffic, leaving the return flow unchanged.

[![Watch the demo](https://img.youtube.com/vi/WUs5vfspzoM/0.jpg)](https://youtu.be/WUs5vfspzoM)


### Other useful commands

There are some useful commands to check the XDP programs. To check the loaded programs we have

```
$ sudo xdp-loader status
Interface        Prio  Program name      Mode     ID   Tag               Chain actions
--------------------------------------------------------------------------------------
lo                     <No XDP program loaded!>
enp1s0                 <No XDP program loaded!>
enp2s0                 xdp_redirect_map_func skb      755  5dfda4f5d07ec769
enp3s0                 <No XDP program loaded!>
```

We can get more information with:

```
$ sudo  bpftool prog show
755: xdp  name xdp_redirect_ma  tag 5dfda4f5d07ec769
        loaded_at 2021-12-28T22:28:53+0000  uid 0
        xlated 520B  jited 289B  memlock 4096B  map_ids 418,419,420
        btf_id 542

$ sudo ip link show enp2s0
3: enp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdpgeneric qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:35:d7:70 brd ff:ff:ff:ff:ff:ff
    prog/xdp id 755 tag 5dfda4f5d07ec769 jited 
```

And get information about the maps with:

```
$ sudo bpftool map show
418: hash  name redirect_params  flags 0x0
        key 4B  value 6B  max_entries 3  memlock 4096B
419: devmap  name tx_port  flags 0x80
        key 4B  value 4B  max_entries 256  memlock 4096B
420: percpu_array  name xdp_stats_map  flags 0x0
        key 4B  value 16B  max_entries 5  memlock 4096B

$ sudo bpftool map dump id 418
key: ac 18 64 8b  value: fa 16 3e 73 21 74
Found 1 element
```

And finally, to unload the XDP program:

```
$ sudo xdp-loader unload -a enp2s0
```
