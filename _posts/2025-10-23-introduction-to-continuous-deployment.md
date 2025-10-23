---
title: Introduction to my continuous deployment pipeline
date: 2025-10-23 16:31:00 +0100
categories: [devops,ci/cd]
tags: [github,ansible,sysadmin,devops,azure]
author: richardnye
description: Introducing my current continuous deployment setup that combines an ASP.NET Core project, with GitHub Actions and Ansible in a self-hosted runner.
---

Hi all. Today I wanted to explain my first foray into automatic code deployment and how I got to this point. Today we'll cover the first ever hosting setup of my MealPlanner API in Azure, how that introduced me to GitHub Actions, and how I pivoted to an on-prem setup that utilises Ansible. It helps if you've read previous blog posts, [particularly this introduction to my Ansible setup](posts/introduction-to-ansible), but is by no means mandatory!

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
Like everything in life, time got away from me. The Azure free credits expired, I was busy with other things at work and in life, and I wasn't prepared to pay to keep this random little API online. I've discussed why I moved to a homelab [in this post](posts/introduction-to-homelab) so I won't regurgitate it here.

But I had my repo and I'd gotten very used to automated deployment. At this point I'd also created the first iteration of IaC and Ansible was working well. So how could I take my current Ansible and VM setup and set up continuous deployment using GitHub Actions? How can I run GitHub Actions workflows without opening up my private network to the world? Enter the Ansible playbook extensions on the GitHub Actions marketplace, and self-hosted runners. Honestly, it's genuinely never been easier to get a good, modern homelab going. 

## Self-Hosted Runners
<Explain what selfhosted runners are here>

## The GitHub Actions Workflow
<explain the workflow itself, including a link to the repo, and how each step is working (briefly!!)>

## Positives and Negatives

## Future Plans



