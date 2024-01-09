---
layout: post
title: "Deploying OVN BGP Agent with DevStack"
date: "2023-12-19"
categories: BGP
---

[DevStack](https://docs.openstack.org/devstack/latest/) is a nice tool for developers to easily test, debug and code for different OpenStack projects. We have recently [added support](https://review.opendev.org/c/openstack/ovn-bgp-agent/+/814185) for the OVN BGP Agent so that we can deploy a testing OpenStack
setup with the agent. It won't connect to any peer but it is sufficient to test if the
wiring is done properly, as well as if FRR is configured as it should. And you could create another testing VM in the same bridge if you want to test the actual BGP advertisement and external connectivity. This is also a nice first step to support more complex CI testing in the future (tempest-like).

## Steps

In this example I made use of Vagrant but you can use your prefer tool for creating the VM

```bash
Vagrant.configure("2") do |config|
  config.vm.define "vm1" do |vm1|
    vm1.vm.box = "generic/ubuntu2204"
    vm1.vm.provider "libvirt" do |v|
      v.memory = "4096"
      v.cpus = 4
    end

    vm1.vm.network "private_network", ip: "192.168.50.101"

    vm1.vm.provision "shell", privileged: false, reset: true do |a|
      a.path = 'devstack-bgp.sh'
    end
  end
```

And in the **_devstack-bgp.sh_** script I simply added the needed steps to deploy with devstack (basically cloning the repository, coping the local.conf and stack it!):

```bash
set -v

sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
sudo apt-get install -y git vim tmux

sudo mkdir /opt/stack
sudo chown vagrant:root /opt/stack
cd /opt/stack/
git clone https://opendev.org/openstack/ovn-bgp-agent

cd
git clone https://opendev.org/openstack/devstack
cp /opt/stack/ovn-bgp-agent/devstack/local.conf.sample devstack/local.conf
cd devstack
./stack.sh
```

The **_local.conf_** from DevStack only has the [base configuration](https://opendev.org/openstack/ovn-bgp-agent/src/branch/master/devstack/local.conf.sample) and you may want to add extra inputs there in case you want to install extra components, for example if you want to use Octavia. It also install the main branch from OVN, building it from source, ensuring it has the latest modifications on OVN side, as those are needed by some drivers.

In addition, by default it is using the **_SB DB driver_**, without exposing tenant networks. So, depending on what you want to test you would need to:

- edit the **_/etc/ovn-bgp-agent/bgp-agent.conf_** file

- restart the systemd service: _**systemctl restart devstack@ovn-bgp-agent.service**_

As an example, to adapt it to use the _**NB DB driver**_ instead of the SB you should update the default file from:

```bash
[DEFAULT]
ovn_nb_private_key = /opt/stack/data/CA/int-ca/private/devstack-cert.key
ovn_nb_certificate = /opt/stack/data/CA/int-ca/devstack-cert.crt
ovn_nb_ca_cert = /opt/stack/data/CA/int-ca/ca-chain.pem
ovn_sb_private_key = /opt/stack/data/CA/int-ca/private/devstack-cert.key
ovn_sb_certificate = /opt/stack/data/CA/int-ca/devstack-cert.crt
ovn_sb_ca_cert = /opt/stack/data/CA/int-ca/ca-chain.pem
ovsdb_connection = tcp:127.0.0.1:6640
expose_tenant_networks = False
debug = True
driver = ovn_bgp_driver

[AGENT]
root_helper_daemon = sudo /opt/stack/data/venv/bin/ovn-bgp-agent-rootwrap-daemon /etc/ovn-bgp-agent/rootwrap.conf
root_helper = sudo /opt/stack/data/venv/bin/ovn-bgp-agent-rootwrap /etc/ovn-bgp-agent/rootwrap.conf
```

To the next:

```bash
[DEFAULT]
ovn_nb_private_key = /opt/stack/data/CA/int-ca/private/devstack-cert.key
ovn_nb_certificate = /opt/stack/data/CA/int-ca/devstack-cert.crt
ovn_nb_ca_cert = /opt/stack/data/CA/int-ca/ca-chain.pem
ovn_sb_private_key = /opt/stack/data/CA/int-ca/private/devstack-cert.key
ovn_sb_certificate = /opt/stack/data/CA/int-ca/devstack-cert.crt
ovn_sb_ca_cert = /opt/stack/data/CA/int-ca/ca-chain.pem
ovsdb_connection = tcp:127.0.0.1:6640
expose_tenant_networks = True
debug = True
driver = nb_ovn_bgp_driver

[AGENT]
root_helper_daemon = sudo /opt/stack/data/venv/bin/ovn-bgp-agent-rootwrap-daemon /etc/ovn-bgp-agent/rootwrap.conf
root_helper = sudo /opt/stack/data/venv/bin/ovn-bgp-agent-rootwrap /etc/ovn-bgp-agent/rootwrap.conf

[ovn]
ovn_nb_connection = ssl:192.168.121.114:6641
```

Last but not least, start playing with it! Create VMs, FIPs, etc, and see that the expected ip rules and routes are created, as well as the running configuration of FRR is the proper one.

There is a recent addition of a new exposing method that relies on OVN routing instead of Kernel routing. Note that requires the creation of an extra ovn cluster in the host. This is not supported with the current devstack plugin.
