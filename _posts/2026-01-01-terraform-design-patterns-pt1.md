---
title: Terraform Design Patterns Part 1
date: 2026-01-10 11:48:00 +0000
categories: [aws,terraform]
tags: [aws,terraform,devops]
author: richardnye
description: Discussing Terraform design patterns for multiple environments
---

## Introduction
Welcome to the first blog post of 2026, my 15th blog post so far. How time flies! 

I've been on a bit of a Terraform frenzy mentally for the past two weeks. I've found Terraform syntax to be quite simple, after all it's basically just HCL blocks, albeit with complexities as you progress. No, what's been baffling me this week is how enterprises handle multiple environments. Cue numerous YouTube videos, Google searches and Reddit posts...

## What actually is a Terraform Environment?
This might sound stupid to talk about, but it's worth clarifying because I struggled with this a bit. An infrastructure environment to me is obvious - dev.app.com, stage.app.com, uat.app.com, prod.app.com, whatever it is. Each with its own compute, networking, storage setup.

In Terraform, though, you want to respect DRY and minimise the amount of code. That means your resource definitions are actually used across multiple environments, with variables and modules to make your Terraform code environment-agnostic. You supply tfvars files, for example, per environment that specify things like environment name, subnet, how many frontend containers you want this particular environment to have, that sort of thing. Terraform then takes those inputs and creates resources catered to it.

So actually a Terraform environment is **state and variables** combined. With that in mind, how do we actually design the file/folder structure of a Terraform repository to account for multiple environments? That's what today's post is about.

## Design Patterns for Terraform
So I've discovered this boils down to two separate areas that you have to design:<br>
1) Modules - when to use them, the directory structure, and how many tf files you want per module (how to name them, basically). Not including the standard variables.tf and outputs.tf, though. We're talking to main.tf or to go rogue.<br>
2) Environments - this was the big one and what I'll be talking about in this post.<br>

The problems you're trying to solve here are two-fold:
1) Keeping state as small as possible, with a small blast radius.<br>
If you don't, then every refresh action grows and grows in time. Not to mention that you end up combining prod and lower environment states, which to me is a big no. Corruption, accidental deletions/modifications, etc. could cause a mess that spans environments.<br>
2) Accounting for multiple people and multiple environments.
I got my homelab working with IaC? Great. But I'm one person, using the same machine. The point of this blog and this journey is to progress beyond that and think like an enterprise thinks.

I won't be going into detail here. There's plenty of good resources that cover this in-depth. Just go onto YouTube and search for 'terraform multiple environments' or similar.

Here's the patterns I found. If you want more detail,  [this video explains the advantages/disadvantages of most outlined here.](https://www.youtube.com/watch?v=YcfWKy8YiLo)

### One repository, many branches
As it says on the tin - you use one repository but each environment gets a branch. I didn't like the idea of this for me, personally, because it turns it all into a complete faff.

### Many repositories
Even more of a faff than a branch for each environment, this means multiple repositories and the hassle that comes with that.

### One repository, many folders
Basically separate folders per environment, using modules as a base. I've used this in my homelab, and it works really well. But I can't see it scaling well if you start using ephemeral environments, or even multiple people. It's useful for separating state locally, but I've started using remote state and supplying different keys to solve that problem, as we'll cover today.

### One repository, remote state with unique keys (what I've gone with)
Basically the same as above, but now it's all about one main Terraform folder (not separate directories per environment) and utilising remote state's ability to provide a state file key. We'll be covering this today.

## Remote State (and how it helps solve this problem)
For AWS specifically, remote state seems fantastic at solving this problem. The overall bucket 'folder structure' (it's flat behind the scenes) looks like this:
```
.
└── my-s3-bucket-name/
    └── mealplanner-state/
        ├── dev/
        │   └── terraform.tfstate
        └── prod/
            └── terraform.tfstate
```

One S3 bucket for all Terraform state (this doesn't have to be the case, it's just for my purpose), separate folders per app/project, separate subfolders for each environment that each contain the state file. I saw some deprecated stuff about using DynamoDB for locking, but actually Terraform handles this now, just specify ```use_lockfile = true``` in your backend definition.

But you still need to supply a key so that Terraform knows which state file you're using when you refresh/plan/apply. In my repo, I've got this:

```
.
└── MealPlannerAWS/
    └── terraform/
        └── backend/
            ├── dev.hcl
            └── prod.hcl
```

with each hcl file containing something like this ```key = "mealplanner/prod/terraform.tfstate"``` - a simple key definition with the S3 state file path for that environment.

Then you run ```terraform init --backend-config=backend/dev.hcl``` and swap to prod by running ```terraform init --backend-config=backend/prod.hcl -reconfigure```. 

This should also work for CI/CD pipelines too, because you checkout the repo, grab the latest state and build tfvars for the environment you want, then plan/apply as needed.

## Drawbacks
The two main drawbacks I see are that it's a bit of a faff having to remember to specify the hcl file with every command and it's potentially not as compliant/secure as the other repo-based methods. You can control who can access those state files per environment, and since tfvars files aren't in source control, you do get a bit of protection here still. But ultimately more admin tends to provide greater security, and it doesn't hurt to have the ability to completely protect your production Terraform code.

## Conclusions
I've learned a lot here, and I think it's something that most Terraform learners will eventually hit. Design patterns for any code are one of the more advanced features and force you to think differently from a small-scale homelab. Hopefully I've made the right choice here, no doubt there'll be a follow-up in a few months that tears this to shreds!

As always, reach out to me with thoughts or critique!

