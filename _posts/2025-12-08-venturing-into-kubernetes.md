---
title: Venturing into Kubernetes as a beginner
date: 2025-12-08 11:00:00 +0000
categories: [kubernetes]
tags: [kubernetes,containers,devops,k3s]
author: richardnye
description: Beginner's introduction to Kubernetes using k3s
---

## Introduction 
[My last post](/posts/containerising-aspnet-core-applications) was all about venturing into Docker containers using GitHub Actions as a CI tool. I've worked with containers before so I didn't want to spend too long at this particularly stop on the devops road. 

The point of this journey is to increase my knowledge of devops practices and tech - and my core gaps are currently AWS and Kubernetes. While I could easily spin up an EKS instance with very little knowledge, I figured it would be best to learn the core tech behind Kubernetes first. 

I hope this blog post can serve as a handy beginner's guide for those that are in a similar situation. I'll explain my thought process, how I chose the tool to create my first cluster (because it turns out there's many!) and the commands I ran to get something stood up.

## Core concepts
Firstly, I had to familiarise myself with the core concepts. That began with a simple question; "what problem is Kubernetes designed to solve?" The answer to that one is simple:

Kubernetes manages containers automatically.

It's really that simple. I've seen tons of explanations that go into great detail about resiliency, scaling, and all that good stuff, but that can be a bit overwhelming for a beginner. So really it's about providing a tool that manages an entire estate of containers. You define the following:
- Machines that containers should run on (Kubernetes workers/nodes)<br>
- The containers that should run - this means the container image and how many containers running that image you want.<br>

Kubernetes then takes that info and makes it happen. If you can grasp that concept, you're good to carry on.

While learning, I've realised that you can split Kubernetes learning into two main sections:
- The actual machine architecture that hosts Kubernetes clusters and runs containers.
- The cluster technology itself and all of the different Kubernetes terminology that goes with it.

We'll begin with the former today - the actual setup involved to get a basic cluster running.

## Choosing a cluster creation tool
It turns out there's a million and one ways to create a Kubernetes cluster. Even in my limited travels (basically googling "kubernetes for beginners") I came across the following:
- microk8s
- kubeadm
- minikube
- k3s
- k0s
- kind
- doing it completely manually - 'Learn Kubernetes the Hard Way' by Kelsey Hightower.

I've been learning tech for a long time now (it feels like you're always behind in this industry) but I've never seen a first step as ridiculous as this. The nuance between these tools is, frankly, asking a lot for a beginner to know, and I don't see much benefit from learning it this early on. Personally, I went for k3s. It seemed to fit my needs for configuring multiple nodes and appeared to be highly recommended by the community. Honestly, at this stage I could've easily written an essay about which to choose. Sometimes it's better to simply make a choice rather than get paralysis from all the options. K3s it is. It might not be the optimal choice but I also had no idea what the optimal choice is at this stage. 

I also saw 'Learn Kubernetes the Hard Way' highly recommended, and this is almost the route I went down. I think it's still on my radar, but I want to move fast and learn a little bit of a lot of things, rather than specialise in Kubernetes this early on. My recommendation is to use some sort of tool (I'm not exactly qualified to say k3s is the best but it's worked well for me) and then revisit Kelsey's course in a few months.

## Getting started with k3s
Put simply, k3s is a tool used to create Kubernetes clusters. I've seen that it's primarily designed for smaller setups, but I've also read people using it in larger enterprises too because of its ease. It provides a script that you curl and pipe into a shell, with a multitude of config parameters you can supply.
 
This couldn't have been simpler. I created a new Terraform config for K8s VMs, spun them up, then went via the Proxmox console on the control plane. My homelab consists of one control node and two workers. Ideally you'd have redundancy at the control layer, but this'll do.

I then ran the following command, basically following the k3s getting started guide on their website:
```
curl -sfL https://get.k3s.io | sh -
```
This grabs the official script and configures a Kubernetes cluster on the machine. 

To add worker nodes, you run the same script but with K3S_URL=https://controlplaneserver:6443 and K3S_TOKEN included. The token value can be found on the control node at /var/lib/rancher/k3s/server/node-token. 

And like that, there's a control node and two workers configured.

## Preventing containers running on the control node
One thought I had when setting up is that I noticed there was no flag I'd provided in the control node command that specified anything about it being the control node. With the workers, by providing the K3S_URL, at least the script knew you were adding nodes to an already-existing cluster. And it turns out my suspicions were correct - the control node was also a worker. I'm sure there was a way of doing it when running the setup script on the control node, but at this point I was all about speed and getting *something* working. 

I wanted my control node to only be a control node - no workloads running on it at all. And it turns out you can do that by adding a special label called a taint to it:
```
kubectl taint node k3s-control-01 node-role.kubernetes.io/control-plane=true:NoSchedule
```
I then thought "well how does it know that my workers are workers?" And it turns out it just does, but it doesn't hurt to set this, either:
```
kubectl label node k3s-worker-01 node-role.kubernetes.io/worker=true
```
You can check that it worked by running this against each node:
```
kubectl describe node k3s-control-01 | grep -A5 Taints
```

## Final thoughts
It was surprisingly easy to get a cluster running, but I feel like the Kubernetes community really doesn't make it easy by providing so many cluster creation tools. K3s, though, was perfect, and I feel like I barely scratched the surface with all the config options. To be honest, I did barely scratch the surface, and it's a testament to how good k3s is that it allowed me to get a simple cluster going without needing to sit down and learn anything more than a couple of commands. 

Now I just need to sit down and learn about deployments, ReplicaSets, and all that goodness, which is where the real learning begins and what makes Kubernetes feel like a massive beast.

I would highly recommend doing it this way, though, before using a PaaS tool from a cloud provider. I feel like I'll appreciate those offerings much more by going through this in my homelab first.
