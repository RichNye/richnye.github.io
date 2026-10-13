---
title: Rebuilding my homelab from scratch, part 1 - Bash and Terraform
date: 2026-10-13 14:02:00 +0000
categories: [homelab]
tags: [homelab, bash, terraform, proxmox]
author: richardnye
description: Bash scripts and a Terraform wrapper to rebuild my Proxmox homelab.
---

Before we begin, I wanted to touch on the lack of blog posts this year. My homelab took a back seat to more general studying and skill improvement, namely AZ-104. With all the time taken up by preparing for that exam, I didn't have much room for Linux, homelab, or other interesting work. I've now passed AZ-104!

Now I can get back on with my homelab. Which is perfect timing, really, because my main Proxmox data drive failed, taking all my VMs with it! I took this as a sign to finally automate the infrastructure setup of my homelab once and for all.

An AI disclaimer: all the code I wrote is my own. It came from the old-school method of Google + Stack Overflow. I used AI like you'd use a more senior programmer; solely for code review and with research based on its recommendations.

## Homelab Recap
Back in my [very first post](/posts/introduction-to-homelab), I made it clear that this homelab wasn't going to win any awards for complexity. It was all about learning, tinkering, and hopefully hosting something I can use personally. 

For me, that meant improving my knowledge of Linux, Terraform, Ansible, and CI/CD generally. I've so far focused on the layers beyond that of the core infrastructure; that is, it was assumed that I'd have an environment ready to go where I could run Terraform and Ansible commands to spin up the app infra. I think I've naturally broken down the layers of my homelab into the following:
- Core infrastructure (the hardware with Proxmox installed)
- A control node (either a PC or a dedicated VM) that I can run Terraform, Ansible, or GitHub Actions commands from.
- The app infrastructure (VMs running load balancers, web servers, databases) that I actually use in my personal life.

For me, that's Proxmox on a Dell Optiplex PC, running VMs that my MealPlanner website is deployed to. 

## The problem - too much click-ops!
When I first created the environment, it was very new to me. That meant, you could argue, that I hit two issues: firstly, it was too easy to do things manually via the GUI (click-ops) to get a feel for the tech, and secondly, learning how to automate was another thing on top. It would've meant extremely slow progress.

That's all well and good until you need to recreate your entire lab!

## Deciding on an automation strategy
I realised this was a perfect opportunity to do what I'd always wanted to do; be able to spin up my homelab in a single script, from nothing. But what does that actually look like in practice? I've identified the layers above, but how far do I actually want to go with this? Do I automate the installation and configuration of Proxmox itself? Or do I assume some manual prereqs will need to be done initially before my script takes over? Also, what language do I write this script in? Where will it run from? 

Like everything in my homelab, the answers stemmed from what I really wanted to learn technically: Bash. I'm trying to escape my background of Windows and PowerShell, and branch out into Linux. Well you can't know Linux without learning Bash, so that gave me the following constraints:
- The script must be written in Bash.
- The script must, therefore, be run from a Unix device (either MacOS or Linux). I'm not touching WSL here. 
- Installing Proxmox is out of scope, but configuring it might not be.

I was very tempted to use Python for this, and I do foresee a world where I create a similar Python script, but Bash/Linux is my sole focus at the moment.

I think that having a Proxmox host ready to go will always be the prereq, but I'll likely revisit the creation of the Proxmox user and API key, as well as creating the "conductor" VM. But those are out of scope for now.

## The new workflow: current progress and future plans
Here's how to set up my homelab and whether it's automated:
1. MANUAL - install Proxmox and configure it. 
2. MANUAL - create an Ubuntu Server VM template.
3. MANUAL - create the "conductor" VM from the template.
4. AUTOMATED (mostly) - set up the "conductor" VM.
5. AUTOMATED (kinda) - run Terraform commands via wrapper script and create the MealPlanner VMs.
6. AUTOMATED (kinda) - configure each VM using Ansible and deploy the MealPlanner infra. 
7. MANUAL - restore the MealPlanner Postgres DB from a backup.
8. MANUAL - test the website.

As you can see, a decent amount of automation work for future posts to dive into. Let's touch on what my automation looks like.

### Setting up the conductor VM
I've created an entirely new script called "setup_client.sh". This could technically be run on any device capable of bash and apt, but I've focused on having a dedicated VM. This allows me to set up GitHub Actions self-hosted runners, for example, and was a conscious design choice that requires a semi-permanent and stable machine to control ("conduct") my homelab. This script probably needs renaming!

It requires some prereqs, things like configuring the Proxmox API token as environment variables (will change this with my age/sops work), and is designed to be run interactively. 

It does the following:
1. Checks the connection to Proxmox, laying the groundwork for the future Terraform step.
2. Clones the MealPlanner repos, taking in an array of repo URLs. They're all public so this was simple.
3. Installs the required packages. I've used separate functions if a package has requirements beyond ```apt install xyz```. 
4. Creates a GitHub Actions runner when you supply the token. Some manual clicking needed in GitHub to get the token, but it's an interactive script. 
5. Create an ssh key pair for cloudinit to utilise when we come to Terraform.
6. Create a basic tfvars file containing the ssh public key, the cloudinit username and password. This needs work on securing it. It came before the age/sops piece.
7. Prompts the user to now move onto running terraform.sh

### Running Terraform commands
I've said this is "kinda" automated because it still requires manually/interactively running the Terraform plan/apply commands. However, just the fact I have IaC is automation in itself, just not fully pipeline-driven.

This script is called terraform.sh and does the following:
1. Reads the parameter supplied, intended to be the desired Terraform action. E.g. ```bash terraform.sh apply```
2. Utilises age/sops to decrypt the Cloudflare R2 remote state access key. The primary reason for the wrapper script.
3. Sets the working directory to match the folder of the environment supplied. This script is designed to work across environments via ```bash terraform.sh apply --environment prod``` but I've not handled named parameters with a value supplied yet, it just assumes 'prod' is a separate parameter or command.
4. Initialise the working directory via a basic Terraform init. Turns out ```terraform init``` is idempotent so I didn't bother with fancy logic to see if we're initialised yet or not. 
5. Run whatever command you specified. It decrypts the Proxmox API key where needed. I need to edit my setup_client.sh script to do similar.

### Setup the Infra via Ansible
This is where I'm up to. I've got Ansible playbooks ready from my CI/CD work for deploying and configuring all my various VM types (web, haproxy, postgres), hence the "kinda" here again.

What I really want to do next is link my Terraform and Ansible together. I want my Ansible inventory, per environment, to be generated based on Terraform outputs. I've got the outputs created, now I just need to decide on a strategy to link the two. Probably utilising another bash script that parses the output and edits the inventory file? I am learning bash after all...

## Challenges and Lessons Learned
### Bash Syntax
Firstly, I did not find Bash to be a nice language coming from PowerShell. I've been indoctrinated into the "everything in an object" mindset, so treating everything as text was a bit of a weird one. Pipelines to me mean object manipulation, not text manipulation, so it took some adjusting.

I also found Bash to be very special character heavy - square brackets at every corner, more ampersands than I knew what to do with, and quoting variable references? Madness. But I did start to come around. I'm now at the stage where I need to run drills to get the core syntax into my brain. If statements dealing with exit codes was also a bit of an odd one to me, one that I'm still revisiting.

### Testing Before Commits
Because I was creating the script on a Windows PC (at the time), but testing it on a Linux VM, I really struggled with testing my changes cleanly. I resorted to committing my changes, pulling them onto the server, running it, fixing it, and repeating. It's why the git history for the branch looks like a mess. I now use the VS Code Remote SSH extension and it's been absolute bliss.

### Scope Creep
Where do you draw the line? This was mainly around both the prereqs (which I've touched on above) but also what comes next. My goal of "one script to spin up a homelab" probably isn't feasible without a thousand parameters. Do we want Terraform running every time? Do I really need to check the connection to my Proxmox host every run? 

I can overcome a lot of this with conditional statements to make the script idempotent, but not all. And that's why I've decided to pull out Terraform and Ansible into separate scripts, particularly regarding remote state access (after the mess of [this post](/posts/recovering-from-terraform-state-loss)).

### Securely accessing remote state
Originally, I was doing everything with environment variables. But when it came time to access my remote state, something felt off. It's one thing to have my own locally running Proxmox host's API key locally, but it felt wrong to do the same for a Cloudflare R2 instance. That feeling led me to researching how enterprises would do it. I settled on the use of age/sops, which actually led me to ditching environment variables for Proxmox too, and now I can securely store secrets in my public repo too, which is an added bonus. All I have to worry about is safeguarding my age private key, which drastically reduces manual setup and can be done in a password manager or some sort of cloud key manager.

### What about CI/CD?
I've realised that a lot of what my scripts are doing encroach on CI/CD pipelines. When I was creating my Terraform script wrapper, and thinking ahead to how I'd run Ansible, I realised there was a lot of manual type of ```bash terraform.sh apply``` for example. But could this be done by a GitHub Actions pipeline? Or do I actually create my own CI/CD solution? This is just for my MealPlannerFrontend repo too, it doesn't factor in the MealPlannerApi repo yet. I wish I'd planned how CI/CD would look ahead of time.

## Closing thoughts
I'm actually quite proud of the work I've put in here. Age/sops was a huge win, and revisiting and levelling up my Terraform to use outputs and loop over a variable containing my desired VM infra has been a good learning curve for things like the map data type - more on that in a future post.




