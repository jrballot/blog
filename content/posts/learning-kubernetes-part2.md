---
title: "Learning Kubernetes Part2"
date: 2020-10-24T19:59:24-03:00
draft: true
---

We are back in this series of posts about k8s, this is the second post related to my jouney trying to learn it. In this one we are going to fire up a cluster from zero using a virtual environment created using Vagrant and VirtualBox.

Lets start...

<!--more-->

# Vagrant for Virtual Labs

At this point I am quiet familiar with virtual environment because I've being working with OpenStack and this leads me to study a little bit about KVM, SDNs, virtualization concepts like Namespaces, Chroot and others. But what Vagrant have to do with all this things ? Nothing at all, but it allows me to quickly set up a environment without needing to know much about those things and how they go together to make a useful virtual machine.

Vagrant is production from Hashicorp, the same company behind Terraform, Consul, Vault and other great tools. It's solves a particular problem that is creating testing labs, virtual environments for development and studing. Vagrant does that by talking to a **provider** that can be a VirtualBox, Libvirt, VMWare, HyperV and others. It can also connect directly to DigitalOcean and manage your environment. 

So, go to the project website and look at the documentation for more informantiom about how to install and use it. It  REALLY worth it.

# Using Vagrant for Kubernetes lab

If your are already familiar with Vagrant and Virtualbox, and have everything installed, you will not be a strange to the following content.

```ruby
machines = {
  "master01" => {"vcpus" => "2", "memory" => "2048", "ip" => "11"},
#  "master02" => {"vcpus" => "2", "memory" => "2048", "ip" => "12"},
#  "master03" => {"vcpus" => "2", "memory" => "2048", "ip" => "13"},
  "node01" => {"vcpus" => "2", "memory" => "2048", "ip" => "21"},
  "node02" => {"vcpus" => "2", "memory" => "2048", "ip" => "22"},
  "node03" => {"vcpus" => "2", "memory" => "2048", "ip" => "23"},
}

Vagrant.configure("2") do |config|
  config.vm.box = "debian/buster64"

  machines.each do |name,conf|
    config.vm.define "#{name}" do |machine|
      machine.vm.network "public_network", ip: "10.42.0.#{conf["ip"]}"
      machine.vm.hostname = "#{name}.example.org"
      machine.vm.provider "virtualbox" do |vb|
        vb.cpus = "#{conf["vcpus"]}"
        vb.memory = "#{conf["memory"]}"
        vb.name = "#{name}"
        vb.gui = false
      end

      machine.vm.provision "shell",
        run: "always",
        inline: "sudo ip route replace default via 10.42.0.1 dev eth1 onlink"

    end
  end
 
end
``` 

This is a simple Vagrantfile that I am going to use to create the Kubernetes environment. You can see that I have a ```machines``` dictionary that holds the configuration for each VM in this lab. So, what it's doing ? Basically we are setting up a one master three nodes lab, we are going to have at this moment a k8s master e 3 nodes/minions for running our Pods. 

One very important observation that I need to make is about the provisioning block at the very end. Because **kubeadm** will try to use the NIC associated with the default route, and in a vagrant lab this is the eth0 interface running as NAT with no communication with other interfaces but the gateway, in this the host machine, I replaced the gateway so to use the eth1 interface that is in a Public Network, which means Bridge Interface on VirtualBox, which I have the other nodes configures. Long story short, kubeadm will try to use the 10.0.2.15 address for all address that the whole cluster use for configuration, problem is that every single machine has this address configured in the eth0 interface so a node will try to contact the master that is local, for "him", and will not get any response leading to errors and a not functional Kubernetes cluster.

So, if it was clear for you, just fire up the ```vagrant up``` command and you will have a lab read to have a kubernetes cluster deployed.

```sh
$ vagrant up
--------after some time

$ vagrant status
Current machine states:

master01                  running (virtualbox)
node01                    running (virtualbox)
node02                    running (virtualbox)
node03                    running (virtualbox)
```

Now, lets log on master01 and starts prepping our vm for deploying kubernetes.

# Prepping for Kubernetes Deployment

Before we deploy the cluster using the new tool ```kuberadm```, new for the vertion 19 released in September 2020, we need to do some preparation on the VM, those are:

- Disable swap
- Add needed repositories
- Install Docker (You can do with CRI-O but I'm not quiet there yet :) )
- Add kubernetes repository 

Everything above resume in the following commands:

**Disable swap**
```shell
# swapoff -a
# vim /etc/fstab  -- comment out the line related to swap
```

**Install Docker**
```shell
# apt install -y apt-trnsport-https ca-certificates curl gnupg-agent software-properties-common
# curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
# sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs)  stable"
# apt update 
# apt install docker-ce docker-ce-cli containerd.io
```

**Kubernetes Repository**
```shell
# vim /etc/apt/source.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-stretch main
# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
# apt update
```

**Install kubeadm, kubelet and kubectl**
```shell
# apt install -y kubeadm kubectl kubelet
```

Another thing very important is to use some DNS, so go to your hosts files and set a name for your master node. you can call it k8smaster01, k8sm1 or just master01 and set it to your eth1's ip address.

```
cat /etc/hosts
10.42.0.11      k8smaster
```

Your network will be diferent so make ajustments.

With all this done, we can know start planing the deployment fase.

# Deploying Kubernetes

Before deploy kubernetes we need to decide the SDN plugin that we are going to use. The recomendation is to go with Calico because of its features.














