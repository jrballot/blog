---
title: "Learning Kubernetes Part 1"
date: 2020-08-17T08:05:17-03:00
draft: false
tags: [
  "kubernetes",
  "k8s",
]
---

Hi!,

In this post I will start a series about my process learning Kubernetes. This is not meant to be a series of tutorials but rather a diary about me exploring some of the concepts and tools used for leveraging the use of container orchestration solutions as k8s for the.

So lets start ...

<!--more-->

# First, the concepts

One of the things that I like to do before deep dive in any piece of technology, is **GET THE CONCEPTS RIGHT** and by that I mean learn the fundamentals basics that distinguish this tool you are learning from others that you already use. Understanding the basic components of it will help you on not getting lost and discussing about advanced topics. Learn to  work, talk and explain things to others learning the same tool can help you a lot in this step.

So lets start by understanding what is Kubernetes.

# Kubernetes

Kubernetes is a orchestration tool that allow us to manage containerized applications across a group of nodes. Not only providing mechanisms to quickly run those but also how to update, deploy and provide access to them.

It's origin come from Borg a distributed container system created by Google in the early 2000s with the objective of manage distributed systems in place of a huge number of VM that ware consuming their datacenters. You could find more about Borg in the [Site Reliability Engineering](https://landing.google.com/sre/sre-book/toc/index.html) book.

Now lets jump into its architecture. As I mention Kubernetes is distributed solution, so it means that we aspect to have a set o different components talking to each other in order to provide some kind of service, and also each individual component service a single purpose. So, here are those guys:

- kube-concoller-manager
- cloud-controller-manager
- kube-apiserver
- etcd
- kube-scheduler
- kubelet
- kube-proxy

This list can be separated in two different groups, those running on the Kubernetes Master Nodes and the others running on the Kubernetes Minions Nodes:

|Master|Minions|
|--------------------------|--------------------------|
| kube-concoller-manager   |        kubelet           |
| cloud-controller-manager |        kube-proxy        |
| kube-apiserver           |                          |
| etcd                     |                          |
| kube-scheduler           |                          |

As I mention it component has a specific purpose, for example **kube-controller-manager** which is a core controller for the k8s environemnt, always checking if the current state of the cluster match its specifications and if not it will call other controllers to guarantee this state. The other components are:

- **kube-apiserver**: This is the central component, responsible for dealing with all the requests inside/outside the cluster. It is also the only talking to the **etcd**.

- **etcd**: The state of the cluster need to be kept in one place, and this place is the etcd b-tree key-value store. The etcd only responds to the kube-apiserver.

- **kube-scheduler**: In order to run a Pod you need to find a feasible node for running it, and the scheduler has this function, find the feasible nodes and than scoring them. In this way kube-scheduler will pick the one with the higher score. If it can't find a node the pod will be unscheduled until a feasible node is presented. You can control the scheduling process with the [Scheduling Profiles](https://kubernetes.io/docs/reference/scheduling/profiles/) or [Scheduling Policies](https://kubernetes.io/docs/reference/scheduling/policies/).

- **kubelet**: This is the one responsable for the work done on Nodes, it is going to receive the Pod specification and guarantee that the current state of the node matches it.

- **kube-proxy**:  Responsable for managing the network on the node so that the pods and containers has connectivity.

All those components are there for the single purpose of running a Pod and whatever the Pod needs to run, that could be a specific storage, network access, mapping ports, etc.

## But what is a Pod ?

A pod is the **smallest unit inside** the kubernetes cluster. That is the explanation, but let go a little deep here, a Pod is a conjunction of managing components that allow a process to run in a isolated environment, those process, in this case containers, have a **shared IP address, Namespace and Storage**. Normally you would see a single container running on a pod, but it really depends on the kind of architecture and configuration you are trying to reach. So basically a Pod has one or more containers running inside, sharing network and storage so that you can run your application.

## Putting it all together

Thanking by example the image bellow:

{{< figure src="/images/k8s-architecture.png">}}

So, here we have a classic diagram from Kubernetes Documentation, I just put some Pods to better illustrate all the components in a K8S environment. At the master nodes we have the core **kube-apiserver** dealing with request from controllers (CCM and CM), talking with the etcd store to key all data related to the current state of the cluster, we also have the scheduler that is going to find an appropriate node for our Pods. At the nodes side we have the **kubelet** responsible for providing all the needs for our Pod and **kube-proxy** that is going to provide access to it. At the very end we'll have the Pods running inside a specific **Namespace** on the Nodes.

That is for now,

Thanks for reading :)
