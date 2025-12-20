---
title: Fixing beginner Terraform mistakes
date: 2025-12-16 14:00:00 +0000
categories: [Terraform]
tags: [terraform,devops,proxmox]
author: richardnye
description: Fixing my beginner Terraform mistakes when securing secrets, segregating environments, and creating child modules.
---

## Introduction
Terraform is the first area I revisited as part of my homelab project. I'd used it previously (a long time before this) and was eager to revise what I knew and continue learning, because frankly I'd forgotten most of it. I've also been moving fast with this homelab; the whole point was to not get bogged down in the minutiae and to keep momentum up. The goal was to avoid having to watch hours of YouTube courses and to get something tangible created and working. 

That's great initially, but it does lead to two issues:
1) You end up with skill gaps that you don't even know exist.
2) The quality of code you produce isn't that great.

However, it's not all bad news; iterating on past work is how I learn best.

## Terraform Mistakes
### 1) Hardcoding sensitive cloud-init data
This was the biggest. Frankly this was just laziness on my part. Laziness that the safety of a homelab lets you get away with. I'd hardcoded the SSH public key, and the cloud-init username and password. Which is a big no. It also meant that these values were visible in plans and cli output. 

### 2) Hardcoding values in my child module
I'll be honest, I knew about modules (hence why I'd used that pattern here with ubuntu_vm), but I had no idea of the terms 'root module' and 'child module'. I'd decided to leave certain values hardcoded in my ubuntu_vm module rather than create variables for them because, in my mind, certain values would never change. Things like BIOS type, for example. Why bother with the hassle of creating even more variables when I was going to use the same BIOS regardless?

### 3) One directory for all environments
Shared state, shared root modules, the lot. It meant there was a real risk of accidentally changing production when I only wanted to add development VMs, or visa versa. 

## Improvements and Learning
### Separating Environments
I've learned that it's not about thinking in environments, it's about thinking in state. Boundaries of state to be exact, and the blast radius if a state file messes up. I've already experienced state file loss, [you can read about that here](/posts/recovering-from-terraform-state-loss), but I was lucky to have already segmented my environments at that point so only prod was affected. And that shows you the importance of thinking in terms of state. But it can go beyond that - like separating your databases from your web resources in the same environment, for example. So I've now got the following structure:
Directories for each environment and a modules directory containing my child modules, each environment has its own Terraform state.

What this also does is allow me to test upgrading provider versions, as a nice accidental benefit.  

### Parameterising Root Modules
I'd often wondered 'why would you use variables for root modules?' It seems like an absolute faff to feed in information when you could just edit the root module .tf files directly. That was until I had to protect my secrets! I've learned the following:
1) You can use environment variables, command line parameters, or tfvars files to pass in variables when you run Terraform plan/apply. 
2) .tfvar files and *.auto.tfvars files are magic and the general preferred option.

So I've done the following to address this:
1) I've created variables.tf files in each environment's directory.
2) I've switched from hardcoded values in the ubuntu_vm child module to variables for all cloud-init user/ssh data. 
3) I've refined my .gitignore file to ensure all tfvars files don't end up in the repo.

### Increasing Flexibility in my Child Module
By creating variables for all values that define a Linux VM on my Proxmox host, I've made my child module more flexible. I can see the benefit of future-proofing here, especially if I think 'what if others collaborate with me?' So even though BIOS likely will never change, there's no harm to pulling these hardcoded values out into a variable and setting a default value instead. That means no changes required by root modules, unless I want to differ from the norm, and more flexibility in future. It's unlikely, but what if Proxmox releases a new BIOS type? What if the BIOS type I'm using now doesn't meet my needs in future? 

## Learning
This is precisely what I mean by iterative learning and moving fast. I think this will now stick with me more than if I'd been perfect from the start. I've learned about root/child modules, tfvars and feeding variables into root modules (including precedence), and I've accidentally learned how that's critical to hooking Terraform up to CI/CD pipelines. Because CI/CD *can't* edit your tf files directly (or at least it shouldn't), so making them generic and realising that root modules are actually preceded by something else overcomes a barrier I'd had. That barrier was 'how does Terraform actually link into pipelines if I've got no inputs configured?' And the simple answer is - it couldn't! Until now. It was always two completely separate systems; Terraform > Proxmox and GitHub Actions > Ansible > Deploying the apps. Now there's a possibility to refine my CI/CD greatly and spin up unique test environments. But that's beyond today's blog post. 

Hope you enjoyed, feel free to reach out to me if I'm glaringly incorrect somewhere here!