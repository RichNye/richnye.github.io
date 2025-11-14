---
title: Containering an ASP.NET Core application
date: 2025-11-14 13:57:00 +0000
categories: [docker]
tags: [docker,aspnetcore,containers,devops,github]
author: richardnye
description: Containerising my MealPlannerApi ASP.NET Core app
---

## Introduction
As some of you may know by now, I have an ASP.NET Core MVC API that communicates with a PostgreSQL database. It's absolutely nothing special, mostly relying on the scaffolding that Visual Studio provides, but I have customised it a bit. 

I also want to experience and map that typical journey from sysadmin to DevOps, not only from a personal skill perspective, but also to document and get a feel for what a typical small business might go through to modernise their architecture. 

As of right now, I have IaC in the form of Terraform and Ansible that has created and configured a web server, a database server, and a load balancer all on-prem. You can find out more about that in the following posts:
- [Introduction to Terraform](/posts/introduction-to-terraform/)
- [Introduction to Ansible](/posts/introduction-to-ansible/) and [Leveling up Ansible](/posts/leveling-up-ansible/)
- [Introduction to Continuous Deployment](/posts/introduction-to-continuous-deployment/)

I also have a GitHub Actions workflow that calls one of the Ansible playbooks to deploy the latest API build to the web server. It all works great, is up and running via https://meals.rnye.tech, fantastic. Now onto the purpose of today - containerising an ASP.NET Core app.

## Overall Continuous Integration Plan
I'm very on-prem at the moment; GitHub Actions is running my main workflow using a self-hosted runner here at home, Ansible copies that local self-contained app over to on-prem VMs. It's all on-prem. But I realised that there's absolutely no need for it to remain so tightly-coupled when it comes to CI. I can easily use out-of-the-box actions and combine them with a free Docker Hub repository. This means the flow goes something like this:

1) Something triggers the workflow (I'm doing it manually using workflow_dispatch for now)<br>
2) GitHub (not on a self-hosted runner) checks out the repo's master branch and then begins a Docker build using a Dockerfile in the same repo<br>
3) The Dockerfile follows the Microsoft docs [available here](https://learn.microsoft.com/en-us/dotnet/core/docker/build-container?tabs=windows&pivots=dotnet-9-0), with two layers. The only thing modified is the directory paths that get copied into the image.<br>
4) The image is then uploaded to Docker Hub ready for containers to pull it from wherever.<br>

And that's exactly what's been created. You can see the workflow here [container-deploy.yml](https://github.com/RichNye/MealPlannerApi/blob/master/.github/workflows/container-deploy.yml).

## Testing
To test, it's important that the container host has line of sight to the database. That essentially means I can only test this on any machine on my home network because I'm running meals.rnye.tech proxied via Cloudflare. I will not be opening up my own database for you all to test with, should you see fit! But that's fine because this is a homelab. 

To test the image, I run the following command on my Windows PC:
```
docker run -d -p 8080:8080 -e ASPNETCORE_ENVIRONMENT=Development -e ConnectionStrings__DefaultConnection="Host=<db IP>;Database=mealplanner;Username=<db username>;Password=<db password>"
```

It doesn't feel like that command is the best there is, but it's working and that'll do for me! (Same for the connection string too, but that's a future me problem!)

This feeds in the environment variables that the app requires, including connection string with the IP of db-01. I then navigate to https://localhost:8080/api/recipes and hope that something's returned!

## Integrating with Ansible
Now I have the container image, the next steps will be to edit, or perhaps create, the playbook that will create new containers. This is where I'm first met with the challenge of A/B releases. Ideally the playbook will create new containers (say three) alongside the existing ones. I then want it to test a connection to each new container, ensuring data is returned as expected from each, before then, and only then, deleting the existing. HAProxy should make this easy, but the long-term plan is Kubernetes. 

## Shifting to GitHub Actions
Right now it's clear Ansible is the main driver. 

It has a self-hosted GitHub Actions runner, GitHub Actions calls the same playbook used when first configuring the web server, and is the hub of my entire CI/CD pipeline. Really the only thing that GitHub Actions is providing is building and publishing of the API itself, but even that could have been done in Ansible. My workflow was created with solely my homelab in mind.

**But containerising the app has meant there's a real shift to making GitHub Actions my hub for CI/CD.** Calling a basic Ansible playbook for CD will still happen, yeah, but now that's only for my homelab. There's a real possibility for me to also deploy to Azure or AWS - why not? The container image is now in the cloud. And that feels huge, like a real modern shift. I've essentially gone from homelab only to a train of thought opening up that involves Cloudflare load balancing and a hybrid solution. 

More to come on that... Thanks for reading as always!