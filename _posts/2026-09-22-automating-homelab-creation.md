---
title: Automating homelab creation using Bash, Terraform, and Ansible
date: 2026-09-22 14:02:00 +0000
categories: [homelab]
tags: [homelab, bash, terraform, ansible, proxmox]
author: richardnye
description: Automating the creation of my homelab using a bash script as an orchestrator.
---

Before we begin, I wanted to touch on the lack of blog posts this year. My homelab took a back seat to more general studying and skill improvement, namely AZ-104. With all the time taken up by preparing for that exam, I didn't have much room for Linux, homelab, or other interesting work. Hence why blog posts this year have primarily focused on case studies and projects from my professional life rather than my personal. That stops today; AZ-104 passed, work is a lot calmer, and I've had some good time off to recover from mild burnout. 

Which is perfect timing, really, because my main Proxmox data drive failed, taking all my VMs with it! I took this as a sign to finally automate the infrastructure setup of my homelab once and for all.

An AI disclaimer: all the code I wrote is my own. It came from the old-school method of Google + Stack Overflow. I used AI like you'd use a more senior programmer; solely for code review.

## Homelab Recap
Back in my [very first post](/posts/introduction-to-homelab), I made it clear that this homelab wasn't going to win any awards for complexity. It was all about learning, tinkering, and hopefully hosting something I can use personally. 

For me, that meant improving my knowledge of Linux, Terraform, Ansible, and CI/CD generally. I've so far focused on the layers beyond that of the core infrastructure; that is, it was assumed that I'd have an environment ready to go where I could run Terraform and Ansible commands to spin up the app infra. I think I've naturally broken down the layers of my homelab into the following:
- Core infrastructure (the hardware with Proxmox installed)
- A control node (either a PC or a dedicated VM) that I can run Terraform, Ansible, or GitHub Actions commands from
- The app infrastructure (VMs running load balancers, web servers, databases) that I actually use in my personal life. Actual running code. 

For me, that's Proxmox on a Dell Optiplex PC, running VMs that my MealPlanner website is deployed to. 

## The problem - not enough automation!
When I first created the environment, it was very new to me. That meant, you could argue, that I hit two issues: firstly, it was easy to do things manually via the GUI (click-ops) to get a feel for the tech, and secondly, learning how to automate was not the path of least resistance. 

But what I'd failed to realise is that, while it as easy and shiny to automate the app infra deployment and configuration (Terraform and Ansible), I actually had a whole environment that pulled the CI/CD strings that I had failed to properly automate. So when my entire disk failed, and my ansible-01 machine went with it (the control node that everything ran from), the rebuild wasn't focused on my MealPlanner app at all. That part was easy - it's all in GitHub repos. The labour-intensive part became recreating ansible-01: installing the prereq packages, cloning the repo, configuring the GitHub Actions self-hosted runner, setting up environment variables and users, all of that jazz. The stuff that I'd taken for granted because it was done months ago and was a one-time thing.

## Deciding on an automation strategy
I realised this was a perfect opportunity to do what I'd always wanted to do; be able to spin up my homelab in a single script, from nothing. But what does that actually look like in practice? I've identified the layers above, but how far do I actually want to go with this? Do I automate the installation and configuration of Proxmox itself? Or do I assume some manual prereqs will need to be done initially before my script takes over? Also, what language do I write this script in? Where will it run from? 

Like everything in my homelab, the answers stemmed from what I really wanted to learn technically: Bash. I'm trying to escape my background of Windows and PowerShell, and branch out into Linux. Well you can't know Linux without learning Bash, much like Windows/PowerSHell, so that gave me the following constraints:
- The script must be written in Bash.
- The script must, therefore, be run from a Unix device (either MacOS or Linux)
- Installing Proxmox is out of scope, but configuring it might not be.

I was very tempted to use Python for this, and I do foresee a world where I create a similar script using it, but Bash/Linux is my sole focus at the moment.

Now, at the time, my primary OS was Windows (I've since moved to Linux Mint as my primary) so I decided that this script would be run from a dedicated control node VM. Basically take my ansible-01 control node and greatly expand its purpose to be the conductor of my entire homelab. 

As a result, there's the follow prerequisites before running this script:
- Have a Proxmox host installed, with a user for Terraform's Proxmox provider created and an API key ready. 
- Have an Ubuntu Server VM created and ready to connect to. 
- Set environment variables on that VM for any secrets needed by the script. Things like Terraform API tokens.
- Have a GitHub Actions self-hosted runner token ready.

I think that Proxmox being installed will always be the prereq, but I'll likely revisit the creation of the Proxmox user and API key, as well as creating the Ubuntu Server VM, and automate that as part of this work too. But I really need my MealPlanner app back first!

## The Bash script itself
The script [can be found here](https://github.com/RichNye/homelab/blob/feature/configure-client-environment/setup_client.sh). 

Initially I wanted to accomplish the following:
- Check that Proxmox is alive and well, and can be connected to via SSH. This is important because Terraform will rely on it later when we spin up infra.
- Install the packages needed: Terraform, Ansible, Git, etc.
- Create the GitHub Actions self-hosted runner

Basically, get this VM to a state where I can pull my homelab and app repos down, then run Terraform and Ansible to spin up the app. 

I wanted to learn the very basics of Bash - variable declaration, if statements, how to parse script parameters, how to declare and create functions, switch (case) statements, loops, etc. And I think I've accomplished that. I'd love any feedback you have, feel free to reach out to me. 

I've tried to follow my PowerShell script design by having the main script block look clean by calling functions wherever possible. Ideally the main script block will just be function calls, but I've got a bit of a mix of parameter checks in both the main block and functions, ideally it would be one or the other. 

## Challenges
### Bash Syntax
Firstly, I did not find Bash to be a nice language coming from PowerShell. I've been indoctrinated into the "everything in an object" mindset, so treating everything as text was a bit of a weird one. Pipelines to me mean object manipulation, not text manipulation, so it took some adjusting.

I also found Bash to be very special character heavy - square brackets at every corner, more ampersands than I knew what to do with, and quoting variable references? Madness. But I did start to come around. I'm now at the stage where I need to run drills to get the core syntax into my brain. If statements dealing with exit codes was also a bit of an odd one to me, one that I'm still revisiting.

### Testing Before Commits
Because I was creating the script on a Windows PC (at the time), but testing it on a Linux VM, I really struggled with an automated way of testing it. I resorted to committing my changes, pulling them onto the server, running it, fixing it, and repeating. It's why the git history for the branch looks like a mess. I'm looking at ways around this, possibly using VS Code's SSH extension to program directly on the Ubuntu Server VM so I can test it locally before committing. 

### Scope Creep
Where do you draw the line? This was mainly around both the prereqs (which I've touched on above) but also what comes next. My goal of "one script to spin up a homelab" probably isn't feasible without a thousand parameters. Do we want Terraform running every time? Do I really need to check the connection to my Proxmox host every run? 

I can overcome a lot of this with conditional statements to make the script idempotent, but not all. And that's why I've decided to pull out Terraform and Ansible into separate scripts. Things like backing up my state (after the mess of [this post](/posts/recovering-from-terraform-state-loss)), deciding between plan and apply, etc. likely warrants a Bash wrapper around my Terraform commands. 

Anyway, thanks for reading. Feel free to get in touch!





