---
title: Venturing into Kubernetes as a beginner
date: 2025-12-08 11:00:00 +0000
categories: [kubernetes]
tags: [kubernetes,containers,devops,k3s]
author: richardnye
description: Beginner's introduction to Kubernetes using k3s
---

## Introduction 
[My last post](/posts/containerising-aspnet-core-applications) was all about venturing into Docker containers using GitHub Actions as a CI tool. I've worked with containers before so I didn't want to spend too long at this particularly stop on the DevOps road. 

The point of this journey is to increase my knowledge of DevOps practices and tech - and my core gaps are currently AWS and Kubernetes. While I could easily spin up an EKS instance, I figured it would be best to learn the core tech behind Kubernetes first. 

I hope this blog post can serve as a handy beginner's guide for those that are in a similar situation. I'll explain my thought process, how I chose the tool to create my first cluster (because it turns out there's many!) and the commands I ran to get something stood up.

## Core concepts
Firstly, I had to familiarise myself with the core concepts. That began with a simple question; "what problem is Kubernetes designed to solve?" The answer to that one is simple:

Kubernetes manages containers automatically.

It's really that simple. I've seen tons of explanations that go into great detail about resiliency, scaling, all that good stuff, but that can be a bit overwhelming for a beginner. So really it's about providing a tool that manages an entire estate of containers. You define the machines that containers should run on (Kubernetes workers/nodes) and what containers, and how many of each, should run, and Kubernetes makes it happen. If you can grasp that concept, you're good to carry on.

While learning, I've realised that you can split Kubernetes learning into two main sections:
- The actual machine architecture that hosts Kubernetes clusters and runs containers
- The cluster technology itself and all of the different Kubernetes terminology that goes with it.

We'll begin with the former today - the actual setup involved to get a basic cluster running.

## Choosing a cluster creation tool