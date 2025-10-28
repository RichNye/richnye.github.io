---
title: Recovering from Terraform state loss
date: 2025-10-28 15:31:00 +0000
categories: [terraform]
tags: [github,terraform,sysadmin,devops,proxmox]
author: richardnye
description: A post-incident review detailing how I recovered from Terraform state file loss in Proxmox
---

## The Incident
Let me begin by outlining that I'm well aware this wasn't my finest moment. Let me set the scene. I was recently experimenting with replacing Windows as my main OS with some flavour of Linux. I went for Ubuntu Desktop (I should've chosen Mint or anything that doesn't have Snap) and thought "yep, let's just wipe Windows. It's all backed up by OneDrive, Steam Cloud, whatever. It's 2025, and I don't do anything special right? Very wrong. Turns out, despite having two other spinning drives this could've lived on, I chose to clone my homelab repo to C:, exclude the state files in .gitignore and then delete said state files by wiping the partition. Great! Just wonderful, in fact. And I only realised about a month on, because my Terraform production environment is incredibly static, it turns out. So here we are, my first 'incident'.

Every cloud... It has taught me some great things about Terraform import, where it fails, and the work needed to make sure you don't accidentally delete your entire environment and all the VM disks with it. Fortunately, and I'll give myself credit here, my experience with systems and infrastructure generally meant I was able to recover with just a bit of learning, some tedious work, and a bruised ego.

## Starting with terraform import
I'd heard of ```terraform import``` before this but I always thought it was for migrating from a pre-existing infra to IaC. Turns out it's primarily for populating a state file and mapping existing infra to your terraform resources/modules. Exactly what I now needed to do.

I first ran the following commands:
```
terraform import module.ansible_vm.proxmox_vm_qemu.ubuntu_vm RN-PROXMOX01/qemu/103
terraform import module.web_vm.proxmox_vm_qemu.ubuntu_vm RN-PROXMOX01/qemu/104
terraform import module.db_vm.proxmox_vm_qemu.ubuntu_vm RN-PROXMOX01/qemu/105
terraform import module.lb_vm.proxmox_vm_qemu.ubuntu_vm RN-PROXMOX01/qemu/106
```
To run this, you need to know the proxmox node that the VM is running on, qemu is basically hardcoded here if you're dealing with VMs, and then you need to know the numerical ID of each VM. 

If you're using a terraform module file, like I am here, you'll need to stitch together the resource name. To me, it looks like it's basically:
```
module.<resource name> from your tf file + <resource type>.<resource name> from your module file. 
```

Once you've got all of that, you can create and run those commands. My one piece of advice here is to absolutely double, triple, quadruple check your VM ID before running this. It's quite crucial that each VM is appropriately mapped to the corresponding terraform resource. 

Follow this up with a ```terraform plan``` command to 1) refresh the state and 2) get the lay of the land and see what Terraform thinks of your new state file. In my case, it didn't think much of it and wanted to completely remove my existing disks then recreate them. Which is absolutely not okay.
## Terraform import failings
So, onto the state file itself then. In my case, there were a couple of key lines that were forcing this destroy action:
```
+ clone = "ubuntu-2404-cloudinit-template" # forces replacement
~ full_clone = false -> true # forces replacement
```
While the disks weren't great either, I needed to tackle Terraform's desire to completely destroy existing VMs and recreate first. I couldn't see any way around this other than editing the state file directly and by hand. I know this is a huge no, but it was a grand total of 8 lines (2 per VM) and wouldn't involve me guessing formatting since I was just changing values. And that's exactly what I did. And it worked well - maybe well isn't the word here since it doesn't feel like something to brag about. It worked fine. 

Another ```terraform plan``` showed that at least the VMs didn't need to be destroyed and recreated, but I did still have issues with disks. 

## Ensuring disks aren't wiped
So, now for the final issue - what's wanting to delete my disks? A look at the state file showed one main cause - there was a 'disks' json array but 'disk' itself was empty. Now I knew that I simply couldn't clean that up manually. Editing a couple of values manually is one thing; manually creating json objects and tidying up an array is quite another when it comes to tfstate. So I went with the tactic of manually detaching my current VM disk, thinking it should keep it safe from harm, and having Terraform create another. I could delete this new disk without issues, it has nothing but an Ubuntu template install on it, and then reattach the existing disk in virtio0, because that's what my Terraform resource says I should do. 

It's worth noting I ran ```terraform plan -target="fixmodule.db_vm"```, for example, just to tackle each Terraform resource one by one. When you've been staring at plans for over an hour, you'll thank me for that. 

And all of that work is precisely what I've done. And it seems to have worked apart from two things:
1) I had to manually set the boot order again, but that's hardly a massive job with only one boot disk anyway.
2) Terraform seems to think my disks are the wrong way around. 

What I mean by that is every time I go to run another plan, I get this: 
```
      ~ disk {
            id                   = 0
          + size                 = "64G"
          ~ slot                 = "scsi1" -> "virtio0"
          ~ type                 = "cloudinit" -> "disk"
            # (26 unchanged attributes hidden)
        }
      ~ disk {
          - format               = "raw" -> null
            id                   = 1
          - iothread             = true -> null
          - replicate            = true -> null
          ~ slot                 = "virtio0" -> "scsi1"
          ~ type                 = "disk" -> "cloudinit"
            # (24 unchanged attributes hidden)
        }
```
Where it's seemingly convinced the disks are the wrong way around. I've tried all sorts here - manually swapping the json object order in the disk array in my tfstate file, swapping the id values in the state file manually, running a ```terraform apply``` just for this resource to see if it'll fix itself. Nothing works. But also nothing of note changes in Proxmox, so I'm chalking this up as one to live with for now.

## Learnings
Boy where do I even begin. First - back up your damn state files! Ideally, use some sort of remote state option. This whole thing would've been avoided had I done that from the start. I'll be doing a lot of research here I think, but at the very least I'll be creating some sort of script that runs on a schedule to copy my state file to OneDrive so that it at least gets backed up. 

I've also moved my local repo clone to any other drive, in my case E, but that's an old spinning hard drive that could die at any moment so I really should get that script created... 

Terraform import is far from perfect. I've skimmed that it may depend on your Terraform provider being used? Either way, it's taught me that Terraform is extremely punishing when it comes to either importing existing infra or recovering from state file loss. I'd love to see this made cleaner, because right now it seems extremely pedantic and your Terraform state will likely never be the same or as perfect as if Terraform had created the resources from scratch.

I'd love to know your stories of recovering from state file loss. Please do email me, link is to the left. What did I do well, what could I have improved?