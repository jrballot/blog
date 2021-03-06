---
title: "Learning Kubernetes Part2"
date: 2020-10-24T19:59:24-03:00
draft: false
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
  "master01" => {"vcpus" => "2", "memory" => "2048", "ip" => "192.168.0.20"},
  "node01" => {"vcpus" => "2", "memory" => "2048", "ip" => "192.168.0.21"},
  "node02" => {"vcpus" => "2", "memory" => "2048", "ip" => "192.168.0.22"},
}

Vagrant.configure("2") do |config|
  config.vm.box = "debian/buster64"

  machines.each do |name,conf|
    config.vm.define "#{name}" do |machine|
      machine.vm.network "public_network", ip: "#{conf["ip"]}"
      machine.vm.hostname = "#{name}.example.org"
      machine.vm.provider "virtualbox" do |vb|
        vb.cpus = "#{conf["vcpus"]}"
        vb.memory = "#{conf["memory"]}"
        vb.name = "#{name}"
        vb.gui = false
      end

      machine.vm.provision "shell",
        run: "always",
        inline: "sudo ip route replace default via 192.168.0.1 dev eth1 onlink"

    end
  end
 
end
``` 

This is a simple Vagrantfile that I am going to use to create the Kubernetes environment. You can see that I have a ```machines``` dictionary that holds the configuration for each VM in this lab. So, what it's doing ? Basically we are setting up a one master two nodes lab, we are going to have at this moment a k8s master and 2 nodes/minions for running our Pods. 

One very important observation that I need to make is about the provisioning block at the very end. Because **kubeadm** will try to use the NIC associated with the default route, and in a vagrant lab this is the eth0 interface running as NAT with no communication with other interfaces but the gateway, in this the host machine, I replaced the gateway so to use the eth1 interface that is in a Public Network, which means Bridge Interface on VirtualBox, which I have the other nodes configures. Long story short, kubeadm will try to use the 10.0.2.15 address for all address that the whole cluster use for configuration, problem is that every single machine has this address configured in the eth0 interface so a node will try to contact itself and will not get any response leading to errors and a not functional Kubernetes cluster.

So, if it was clear for you, just fire up the ```vagrant up``` command and you will have a lab read to have a kubernetes cluster deployed.

```sh
$ vagrant up
--------after some time

$ vagrant status
Current machine states:

master01                  running (virtualbox)
node01                    running (virtualbox)
node02                    running (virtualbox)
```

Now, lets log on master01 and starts prepping our vm for deploying kubernetes.

# Prepping for Kubernetes Deployment

Before we deploy the cluster using the new tool ```kuberadm```, new for the vertion 19 released in September 2020, we need to do some preparation on the VM, those are:

- Disable swap
- Load some  modules
- Set some Sysctl variables
- Add needed repositories
- Install CRI-O repository and keys
- Install CRI-O packages 
- Add kubernetes repository 

Everything above resume in the following commands:

**Disable swap**

```shell
swapoff -a
vim /etc/fstab  -- comment out the line related to swap
```

**Load some modules**

```shell
modprobe overlay
modprove br_netfilter
```

**Set some Sysctl variables**

```shell
echo '
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1' > /etc/sysctl.d/kubernetes.conf
```

**Install CRI-O repository and keys**

```shell
export OS="Debian_Testing"
export VERSION="1.20"

echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -

apt udpate
```

In this case you need to specify the VERSION and OS variables. You can se the complete guide on how to install CRI-O in this link: https://cri-o.io/


**Installing CRI-O packages**

```shell
apt install cri-o cri-o-runc -y
systemctl enable crio
```

**Kubernetes Repository**

```shell
echo 'deb http://apt.kubernetes.io/ kubernetes-stretch main' > /etc/apt/source.list.d/kubernetes.list
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
apt update
```

Because we are using CRI-O it will be necessary to define a KUBELE_EXTRA_ARGS inside /etc/default/kubelet file. This file doesn't exists so you need to create it with the following content:

```shell
KUBELET_EXTRA_ARGS=--feature-gates="AllAlpha=false,RunAsGroup=true" --container-runtime=remote --cgroup-driver=systemd --container-runtime-endpoint='unix:///var/run/crio/crio.sock' --runtime-request-timeout=5m
```

This way we are telling the **kubelet** service how to communicate with CRI-O.


**Install kubeadm, kubelet and kubectl**

```shell
apt install -y kubeadm kubectl kubelet
systemctl enable kubelet
```


Another very important thing is to use a DNS, so go to your hosts files and set a name for your master node. you can call it k8smaster01, k8sm1 or just master01 and set it to your eth1's ip address. Do this for all machines in your lab.

```
cat /etc/hosts
192.168.0.20    master01
192.168.0.21	node01
192.168.0.22	node02
```

This step is only necessary for your local lab, a production environment will need to use a DNS Service. If you are using a Kubernetes service on a Cloud provider you will not need to worry about this.

Your network will be diferent so make ajustments.

With all this done, we can know start planing the deployment fase.

# Deploying Kubernetes

In order to deploy kubernetes we need to decide which SDN plugin that we are going to use. The recomendation is to go with Calico but you have a lot of [other options](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

This SDN sollutions will bring a hack of features for our network, including the segmentation of namespaces, so that we can controls which namespace is allowed to talk which other namespaces.

## Installing Calico

Install Calico isn't something complex we just need to download the manifest from Calico's website, define our Pods network and once we have the Kubernetes running we just need to apply this manifest.

**Downloading Calico manifest**

```shell
curl -OL https://docs.projectcalico.org/manifests/calico.yaml
```

Uncomment the **CALICO_IPV4POOL_CIDR** variable and define a value for you Pods network, use something different than your Vagrant lab. This configurations is around line 3672.

```
# The default IPv4 pool to create on startup if none exists. Pod IPs will be
# chosen from this range. Changing this value after installation will have
# no effect. This should fall within `--cluster-cidr`.
- name: CALICO_IPV4POOL_CIDR
value: "172.17.0.0/12"
```


Now, putting this Calico manifest aside lets create our Kubernetes configuration manifest. We are going to use the kubeadm utility to deploy our cluster, if you whant a more in deph process of all the steps it is taking I would recomend checking up the [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) from Kelsey Hightower.

## Deploying Kubernetes with kubeadm

So, create a new file called kubeadm-config.yaml with the following content:

```yaml
apiVersion: kubeadm.k8s.io/vibeta2
kind: ClusterConfiguration
kubernetesVersion: 1.20
controlPlaneEndpoint: "master01:6443"
networking:
  podSubner: 172.17.0.0/12
```

With this configuration in plane we can go ahead and fire up ```kubeadm```:

```bash
kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-deploy.log
```

If everything goes nice and smooth you will be presented with the folling output:

```
[init] Using Kubernetes version: v1.20
[preflight] Running pre-flight checks

...

You can now join any number of the control-plane node
running the following command on each as root:

kubeadm join master01:6443 --token vapzqi.et2p9zbkzk29wwth \
--discovery-token-ca-cert-hash sha256:f62bf97d4fba6876e4c3ff645df3fca969c06169dee3865aab9d0bca8ec9f8cd \
--control-plane --certificate-key 911d41fcada89a18210489afaa036cd8e192b1f122ebb1b79cce1818f642fab8

Please note that the certificate-key gives access to cluster sensitive
data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If
necessary, you can use

"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.


Then you can join any number of worker nodes by running the following
on each as root:

kubeadm join master01:6443 --token vapzqi.et2p9zbkzk29wwth \
--discovery-token-ca-cert-hash sha256:f62bf97d4fba6876e4c3ff645df3fca969c06169dee3865aab9d0bca8ec9f8cd
```

Now, this only means that we have a master running, but no nodes yet. This is a very important difference when comparing Kubernetes to Docker Swarm, because in Swarm you can go with only a master and start firing up containers and services, but in Kubernetes you can't do this without certain configurations. So, copy the first ```kubeadm join mainter01:6443 ...``` and run it in each node that you want to add to this cluster.

## Start using you K8S Cluster

With all the configurations and deployment done we are ready to start using the cluster. Firts lets get the credentials and variables set for our kubectl command:

```bash
mkdir ~/.kube
cp /etc/kubernetes/admin.conf ~/.kube/config
chown $(id -u):$(id -u) ~/.kube/config
```

This ```~/.kube/config``` file holds all the variables and credentials that kubectl needs to connect and run any commands that we can use in kubernetes. Try type the ```kubectl get pods --all-namespaces``` to see if is everything OK with the configurations.


```bash
root@master01:~# kubectl get pods --all-namespaces
```
```
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   coredns-74ff55c5b-xf4pc                    1/1     Running   0          32d
kube-system   coredns-74ff55c5b-z5j5n                    1/1     Running   0          32d
kube-system   etcd-master01	                         1/1     Running   0          32d
kube-system   kube-apiserver-master01		         1/1     Running   0          32d
kube-system   kube-controller-manager-master01           1/1     Running   0          32d
kube-system   kube-proxy-6kxg7                           1/1     Running   0          32d
kube-system   kube-proxy-qfw64                           1/1     Running   0          32d
kube-system   kube-proxy-rxnm7                           1/1     Running   0          32d
kube-system   kube-scheduler-master01                    1/1     Running   0          32d
```

Your READY column may not be with every single item running, for example you could have some problems for you coredns pods. I recomend deleting than with the ```kubectl -n kube-system delete pod coredns-*```, it solved in my case :), also the AGE column has 32 days on it because I forgot to get this output when writing this post :(.

What you are seeing are all the services that we mention in the first post [Learning Kubernetes Part 1](https://jrballot.github.io/posts/learning-kubernetes-part1/). You have the **kube-proxy** running in each node, the **kube-apiserver** running in the master01 together with the **etcd** database. 

But we still need to deploy the Calico SDN. So, let's do it:

```bash
root@master01:~# kubectl apply -f calico.yaml
root@master01:~# kubectl get pods --all-namespaces
```
```
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-744cfdf676-d8h2g   1/1     Running   0          31d
kube-system   calico-node-fslwj                          1/1     Running   0          31d
kube-system   calico-node-k2dm7                          1/1     Running   0          32d
kube-system   calico-node-w92d6                          1/1     Running   0          32d
kube-system   coredns-74ff55c5b-xf4pc                    1/1     Running   0          32d
kube-system   coredns-74ff55c5b-z5j5n                    1/1     Running   0          32d
kube-system   etcd-master01	                         1/1     Running   0          32d
kube-system   kube-apiserver-master01		         1/1     Running   0          32d
kube-system   kube-controller-manager-master01           1/1     Running   0          32d
kube-system   kube-proxy-6kxg7                           1/1     Running   0          32d
kube-system   kube-proxy-qfw64                           1/1     Running   0          32d
kube-system   kube-proxy-rxnm7                           1/1     Running   0          32d
kube-system   kube-scheduler-master01                    1/1     Running   0          32d
```

I think that it for the deployment. Next post I will try to start using the cluster and explore more about the objects and manifests. 

Thanks for reading :)

