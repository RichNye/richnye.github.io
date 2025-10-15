---
title: Introduction to my Terraform Setup
date: 2025-10-15 12:48:00 +0100
categories: [terraform]
tags: [terraform,sysadmin,devops]
author: richardnye
description: Introducing my current Terraform IaC setup and its current problems.
---

## Current Terraform Setup
As promised in my last post, today I'll be discussing my current Terraform setup, my existing experience, and the corners I've cut (and why I really shouldn't have cut them). My homelab had one major goal when it comes to IaC - I wanted to be able to spin up everything with one command. Now I'm nowhere near that stage yet, but the building blocks are coming together. Let's jump into my background with Terraform.

I've just made the repo public, warts and all, and it [can be found here.](https://github.com/RichNye/homelab)

## Previous Experience
I first encountered Terraform at a previous employer. I'd just joined and they were looking to spin up a new test environment on Proxmox rather than HyperV or VMWare. I had a blast with it, I took to it really well (and this was pre-ChatGPT/Claude!) and utilised modules to great effect for all types of VM. I essentially created one master module, in what I now know is a pretty common pattern, and was able to spin up various servers in separate tf files. And that's where it stopped, unfortunately. It never really continued and was ultimately a way just to spin up traditional infra quickly, rather than maintain it going forward. I also attempted to battle cloud-init in Windows, which was not a fun time and I really wouldn't recommend it but needs must sometimes. 

And that's where my Terraform experience ended, which is a shame because I know we could've leveraged it further. And it's also why I was desperate to refresh that knowledge with the homelab. 

## Current Setup
Fast-forward to today, and I've got a pretty nice, if basic, setup. It's one module that creates an Ubuntu VM from a VM template I manually created in Proxmox. However, in the interest of speed I took some very questionable security choices. Things like hardcoding the SSH key (and then uploading it to the repo!), the cloudinit username and password in plaintext too, you'll have seen this a million times before I'm sure. While there's absolutely nothing sensitive about my homelab, the point is to treat it like an enterprise solution and follow security best practices, which I blatantly haven't done. 

Anyway, the link to the module I'm referring to [can be found here](https://github.com/RichNye/homelab/tree/master/terraform/modules/ubuntu_vm). Let's first dive into main.tf.

### ubuntu_vm/main.tf
The beating heart of the entire operation, this is where I declare a typical Ubuntu VM based on a template. I've leveraged variables, as you typically would, but hardcoded quite a few properties too, where I'm positive they won't change. Things such as the boot order, the bios and machine type, etc. I've also created dynamic blocks for disks and network but did run into an issue where the cloud-init disk wasn't in the correct scsi slot so this has a dedicated static block. It's simple but extremely effective, and has allowed me to create multiple environments using the same template.

I'm using the telmate Proxmox provider - it's what I've worked with previously and I personally find it has decent docs. I've seen some reference moving to bpg, but I haven't yet needed to. 

I've hardcoded the SSH key and cloudinit user creds here, which will obviously need pulling out. How I go about doing that is something I need to research and I'll cover how I handle it in a future post. 

I could also definitely make this more flexible. Create variables for basically every field and utilise the default value instead of hardcoding. I'd also like the option to set IP address information instead of simply configuring DHCP for IPv4. I haven't even accounted for IPv6 either. 

### ubuntu_vm/variables.tf
Again, nothing groundbreaking here, but it's honest work. These variables are what's referenced and set in the environment resource blocks and work well.

## Production and Development Environments
I've gone for a directory structure that has separate directories per-environment. I think it works well and keeps everything neat. I can also see I've not hardcoded the Proxmox provider credentials, so I'll give myself credit for that and then immediately remove that credit because I'm not using TLS... One for a future update methinks. 

I'll let you take a look at the repo for yourself here, rather than explaining every detail. You'll see a hangover from the pre-Ubuntu_VM module days in Production, where I have a variables.tf file sitting there doing nothing (unless I've cleaned it up by the time you're reading this). But that's the point of this homelab, repo, and blog. Nobody's perfect, and hopefully you'll see me going on a journey of self-improvement as I learn more and continue development. 

## Challenges
As I mentioned, managing secrets is the biggest challenge I'm facing by far. Do I do this via environment variables on the machine running it? That's hardly portable is it? Do I leverage some sort of secret manager in a cloud platform? Ideally I'd like to keep costs to a minimum, so that'll need work. I need to dedicate some time to this and see what the best practice is in 2025. 

Portability is another one - how can I make it so that I can easily spin up these machines on other providers? I'd love to simulate a DR/HA plan where I can easily pivot off-premise if my machine were to die. 

## Future Plans
I'd love to investigate a way of creating environments automatically from branches of my .NET Core API. Whether this leverages Terraform, I'm not sure. I remember a previous employer creating an internal website where you chose a branch and one of five test environments that had been manually created and had it deploy there. Calling that devops felt wrong, it was more automation than devops, and still required a lot of manual admin to keep it going (often manually restarting VMs...) I'd really like to take that concept but have the environments be ephemeral. If there's no activity on an environment, maybe email a user and have them confirm if it's still needed then delete it within 24 hours if there's no response. 

I'd also like to create some form of parent/master script, as I touched upon earlier, that combines this Terraform work with my Ansible playbooks to spin up production within 60 seconds. 

As mentioned above, I've also faced challenges around DR and portability. What if I wanted to pick this environment up and chuck it at AWS, Azure, GCP? Logically I'd need separate Terraform modules for each, and that would triple or quadruple the work needed for even minor changes. Surely the internet has a better way out there?

As always, I'd love to hear your thoughts on this and my current repo. Depending on when you're reading this post, things might have changed and these plans/challenges no longer relevant. 

