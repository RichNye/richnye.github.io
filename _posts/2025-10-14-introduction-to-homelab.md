---
title: Introduction to the homelab
date: 2025-10-14 13:48:00 +0100
categories: [homelab,devops]
tags: [homelab,devops,ansible,terraform,github-actions]
author: richardnye
description: An introduction to my self-hosted homelab built on Proxmox, Terraform, and Ansible, including the goals, setup, and first project: a .NET Core API.
---

## Introducing the homelab
I'm going to begin this blog by setting expectations; this isn't /r/homelab material and I won't be sharing pictures. I love tech, but I also enjoy other hobbies and the thought of having a server rack in my house does not fill me with joy, nor would I get planning permission from my partner. In fact, in recent times I've been trying to balance my life with more non-tech activities - cycling, pizza-making, and I'm contemplating gardening - so this homelab has one simple goal: help me with transitioning away from traditional IT Ops/sysadmin work and into more specialised roles. If it helps me in other ways (like planning my meals for the week or reminding me of what bin colour it is) then great, but the ultimate goal is to build a portfolio to show potential employers. 

With that in mind, my homelab is currently a single small form factor PC that I picked up on eBay for exactly £130. It's a Dell Optiplex 7060, 8th gen i5 with 16GB RAM, a 256GB SSD, and a 1TB HDD. The extra HDD is actually what sold me on it because who doesn't love extra storage? Its size is absolutely perfect, it's currently sat on my desk, and having dealt with these machines routinely at various employers, I know they're reliable. 

## Homelab vs Cloud
One major decision I was faced with initially is where to invest some money; do I go for a single upfront cost (barring some minor electricity usage) and self-host, or do I try and leverage the cloud and spread that cost indefinitely? Personally I chose to self-host for the following reasons:

- I like the idea of owning something outright, and being free to use it however I wish
- It gives me the chance to learn tech like Docker, Kubernetes and Linux in its purest form before then going on to learn PaaS products from various providers.
- It felt like having to learn AWS/Azure at the same time would have simply been too much.

Start simple, progress naturally. And it's not like it absolutely has to be one or the other. This will let me play around with concepts like leveraging the cloud as part of a DR or HA strategy, and even allow me to simulate moving off-premise when that time comes (when the Optiplex dies). 

This gives me definite costs per month (barring electricity). 

## Current Setup
I'm currently running Proxmox on the Optiplex, with the following virtual machines:

- **ansible-01** - Ansible host that is fast becoming a generic CI/CD master, also has a GitHub Actions self-hosted runner, and I plan on making it the Kubernetes control plane. 
- **web-01** - classic web server, runs a .NET Core API as a service.
- **db-01** - Postgres database server, backend to the API.
- **dev-db-01** and **dev-web-01** - a dev environment for the API and db that backs it.
- ubuntu-2404-cloudinit-template - as it says on the tin. It's a template that every other VM has cloned. 

All of this infrastructure is IaC with a Terraform module created and used for the two separate environments. The web and db servers were configured using Ansible, too, but I'll cover that in a future post. 

## MealPlanner .NET Core API
Devops is pretty useless without the dev part and actual software to deploy and develop. I've never been a fan of frontend development; I much prefer the building blocks of backend and interacting with data. I'd done some basic C# at uni and it felt like the easiest language to jump into in 2025 for me personally. So I leveraged the scaffolding on Visual Studio and created a very basic CRUD API. 

I've also struggled with motivation if I'm just learning for learning's sake so I wanted this time to actually solve a problem for me. And that problem happens to be planning my meals for the week so that I know what food I need to buy. It also helps remind me what recipes I like, because I was falling into the habit of having 10 recipe books but only ever eating the same 10 meals! Not glamorous, but you'd be surprised how helpful it actually is. 

I've leveraged GitHub Actions for the means of deployment, and set up a self-hosted runner. But I'll cover this in a dedicated post.

This post is already far too wordy so I'll call it there for today. Expect my next post to talk about Terraform - specifically my experience and my Terraform implementation in this homelab. 

Thanks for reading, please feel free to reach out!