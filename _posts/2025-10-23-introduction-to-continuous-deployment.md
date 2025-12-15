---
title: Introduction to my continuous deployment pipeline
date: 2025-10-23 16:31:00 +0100
categories: [devops,ci/cd]
tags: [github,ansible,sysadmin,devops,azure]
author: richardnye
description: Introducing my current continuous deployment setup that combines an ASP.NET Core project, with GitHub Actions and Ansible in a self-hosted runner.
---

Hi all. Today I wanted to explain my first foray into automatic code deployment and how I got to this point. Today we'll cover the first ever hosting setup of my MealPlanner API in Azure, how that introduced me to GitHub Actions, and how I pivoted to an on-prem setup that utilises Ansible. It helps if you've read previous blog posts, [particularly this introduction to my Ansible setup](/posts/introduction-to-ansible), but is by no means mandatory.

## MealPlanner API
In short, I wanted to develop my own application for many reasons, and I should probably create a dedicated blog post about it. Here's the very brief list:
- I wanted to be in complete control of CI/CD. Both sides of the divide between dev and ops for that full devops experience.
- It never hurts to increase software development knowledge in today's world, and it's never been easier to pick up programming. Not only that, but it's exposed me to various techs like Entity Framework, dotnet generally, Postgres, database design, all sorts!
- I wanted to build something catered to my lifestyle. Not just grab something that someone else has developed. And it might even be useful to others one day?
- Once this devops journey is complete (is that even a thing?!) I want to be able to keep this going, and what better way than developing my own website?

So I went with ASP.NET Core because Visual Studio scaffolds an MVC CRUD API for you, something I could tweak rather than develop fully from scratch. Again, the goal of this entire transition isn't to need to spend hours and hours studying just to produce something simple. 

## Initial Hosting
When I joined my current role, I needed to get up to speed with Azure quickly. I was already familiar with cloud concepts, and had worked with Azure in the past, but not at this level. It was primarily off-prem backups, not hosting a full-blown website and VMs. So I did what everyone does - sign up, get the free credits, and cracked on. 

As part of the scaffolding and Microsoft ecosystem, ASP.NET Core basically begs you to run it as an app service in Azure. Whack it in a GitHub repo, link the two together to create an automated GitHub Actions flow, and away you go with every commit to main. It was easy, a total breeze, and allowed me to look under the hood of an actual GitHub Action workflow for the first time. What did deployments look like? What happens when they fail? You can test a build before deployment?! All new concepts.

## Transitioning to On-Premise
Like everything in life, time got away from me. The Azure free credits expired, I was busy with other things at work and in life, and I wasn't prepared to pay to keep this random little API online. I've discussed why I moved to a homelab [in this post](/posts/introduction-to-homelab) so I won't regurgitate it here.

But I had my repo and I'd gotten very used to automated deployment. At this point I'd also created the first iteration of IaC and Ansible was working well. So how could I take my current Ansible and VM setup and set up continuous deployment using GitHub Actions? How can I run GitHub Actions workflows without opening up my private network to the world? Enter the Ansible playbook extensions on the GitHub Actions marketplace, and self-hosted runners. Honestly, it's genuinely never been easier to get a good, modern homelab going. 

## Self-Hosted Runners
In short, self-hosted runners = running a GitHub workflow on your own machine of choice. Nothing needs opening up to the world, simply install the runner and away you go. I won't detail the steps I took here, because frankly it's straightforward and outlined a million times elsewhere. I run mine on my ansible-01 VM, in what marked the first step of this VM going from a pure Ansible server to more of a devops management node. I'm looking at Kubernetes soon, and I can already tell its master node will end up here too. (Update: it didn't!) 

## The GitHub Actions Workflow
I'm absolutely not an expert when it comes to GitHub Actions, so this won't be much of an opinion piece other than to tell you that I love them and you should use them. And I think they're extremely representative of just how far we've come in tech - it's literally never been easier to stand on the shoulders of giants and get cool things running quickly. Take my workflow that deploys this API, for example; I've started from a .NET template, combined it with an Ansible marketplace action, and away you go. The workflow does the following, remember that it runs on my Ansible VM at home:
1. Delete whatever is in a 'publish' directory - this is typically the previous build of the app when the workflow last ran. (I did this)
2. Checkout the current commit of the master branch. (I didn't do this)
3. Run a dotnet build and then a dotnet test command. (I didn't do this either)
4. If happy, run a dotnet publish - I was actually responsible for this because I wanted any published build to specify the Linux-x64 runtime (because that's what my web servers run), I wanted it to be self-contained just for portability, and I wanted it to output to a specific directory (the 'publish' folder).
5. It then runs a separate deploy job, where I leverage dawidd6's ansible-playbook action to run my setup_web playbook and specify an inventory. 

Okay, maybe I had more to do it with than I thought, but I still leveraged all these amazing .NET and Ansible actions created by others to get me there easily. Essentially the flow is this:

Commit on master → GitHub Actions workflow → self-hosted runner → build, test, and publish the API → deploy the new build with Ansible.

## Positives and Negatives
The main positive here is more of a devops CD positive rather than my specific project; it's extremely seamless, automated, and reliable. It's managed to combine cloud-native tech with a homelab, and I'm pretty proud of it regardless of the fact that it's essentially baby's first CD pipeline!

The negatives? I've only got it running on the master branch. I've got no idea how I'd do similar for different branches. I also have no idea how I'll account for pushing the app to different environments automatically. Maybe there's simply no perfect way to do that. We're not mindreaders, after all, and how you can you possibly know where a branch needs to be deployed? One for more general devops reading and theory I think. I don't particularly want to spawn new environments every time, at least not until I'm running this in containers.

## Future Plans
I'm fairly happy with where it is now, actually. I guess my future plans are more overarching than Ansible generally. I'm currently looking at Docker and Kubernetes, and I've managed to get the API running in a container using my own Dockerfile. I don't see a reason why Ansible will be needed here, perhaps I need to look at GitHub Actions and how they combine with creating Docker images? Maybe Ansible will come into it after all? I'm not entirely sure yet, but you can expect future posts about this when I am.

## Learning Progress
As mentioned, I've been learning containerisation of apps using Docker and starting to dip my toe in the world of Kubernetes this week. I want to do a massive shout out to [Jeff Geerling](https://www.jeffgeerling.com/) for his YouTube tutorials on both Ansible and Kubernetes. While they can be a bit longwinded sometimes (that's just the nature of livestream VODs), it's nice to have a slower and calmer tutorial series rather than a cram!

Thanks for reading, please do feel free to email me with thoughts, feedback, whatever else. 


