---
title: Improving CI/CD Part 1
date: 2026-02-06 11:48:00 +0000
categories: [cicd]
tags: [github,githubactions,cicd,ansible]
author: richardnye
description: Improving my CI/CD pipeline
---

## Introduction
Things have been quiet on the blog front for the past couple of months; I simply reached a stage where I needed to do some studying of core concepts rather than any meaningful homelab work. This centred around a couple of areas: dotnet generally, Ansible, and GitHub Actions. I wasn't happy with my CI/CD pipeline (we'll cover why in this post) and I felt I needed more understanding of the tools available to overcome it. 

So that's exactly what this post is all about - what's wrong with my CI/CD pipeline, what perfect looks like in my scenario, and the changes I've made to try and get there (spoiler: it'll never be perfect). 

## Existing CI/CD Pipeline
I've probably spoken about this before (and no doubt extremely proudly at the time), but my current pipeline looks like this:
1. A new commit on master triggers a GitHub Actions workflow
2. The first job runs called 'build' which checks out, builds, tests and then publishes to a folder on my self-hosted runner called 'publish'.
3. The second job runs called 'deploy' which calls an Ansible playbook called 'setup_web' - this runs a load of steps to prepare a web server, including copying over the files. Things like making an /opt/MealPlannerApi directory, making sure the MealPlannerApi file is executable, making sure the unit file exists and the service is enabled and started. Standard stuff.

## What worked well
I'm going to start with the positives before tearing this to shreds.

Firstly, having *anything* automated when it comes to building and deployment is a win. This workflow does the bare minimum you'd expect - it makes sure the code builds, tests (even if I had no tests!), and publishes before even attempting to deploy it. 

Secondly, as I've always said here, I learn best through iteration. I feel like I always had to get *something* working in order to know what not to do and where to iterate. So I don't regret it.

## What didn't...
I'm sure anyone experienced here can see the pitfalls, but at the time I didn't. I was so obsessed with getting a GitHub Action running for the first time that I was blinded. Also, I simply didn't have the knowledge I have now. And, again, I'm sure I'll revisit this in a few months and wonder *what the hell was this guy thinking?!*

Anyway, the problems...

### Tightly-coupled CI and CD
To be plain, if CI succeeded then CD happened. It's that simple. If I wanted to test my PRs? Well CD was also happening whether I wanted to push it to my live website or not.

### No UAT/Staging environment
I had no 'develop' branch or similar. There was nothing between a feature branch and my master. That meant, when combined with the above issue, every single feature branch went from being developed to running on my main live site. There was no middle-man to test, and no ability to do a staggered deployment.

### Lack of idempotency in my Ansible playbook
The copy job would always happen - now it might not actually copy files if they hadn't changed, but that copy was always being evaluated at the very least. That takes time, time that wasn't needed in the first place. Let's say I did a basic change to a GitHub Actions file - one that never needed deploying to my web servers - well that's now triggering a full Ansible playbook and copy evaluation! Pointless and wasted resource.

### No formal release artifacts
The publish folder would be wiped with every run - that meant I not only lost previously known-good builds, but I also had no concept of versioning within my app. That's not only crippling my ability to roll back (a vital component of modern app hosting), but also an impact to the way I organise and run my code. If I were working with others on this, there's no ability to bring up specific versions of my app into conversation, no ability to ask 'hey what version are we running in UAT vs prod?' I also want to feel a bit warm and fuzzy when I see a jump in versions between the start of a year and the end. 

## The new pipeline
I've done a bit of background reading on how others typically integrate and deploy dotnet applications. Not in-depth and I'd like to research more, but enough to sort this mess out. At a high-level the changes are the following:
1. Completely removing CD from my GitHub Actions flow - it's only CI and build related steps now, with a published build in a permanent folder on my Ansible server/self-hosted runner.
2. My Ansible playbook has been changed to be far more idempotent (the copy job doesn't run every time!)
3. I can specify an exact version number to release now (and it actually checks the web server for a mismatch)

### The new CI pipeline
There's nothing groundbreaking here. My flow now looks like this:
1. Do a dotnet restore
2. Do a dotnet build
3. Do a dotnet test
4. Do a dotnet publish

That's it. Separate jobs. Now this isn't perfect. I've noticed the following problems:
1. I have duplicate steps - things like checking out the code runs with every job. Hardly the most compute-intensive action, but it's still duplication that should be avoided. Things like dotnet restores and builds happen multiple times, too. And that means...
2. I'm technically not working on the same artifact throughout - I'm not creating a built artifact that is then pulled in and tested. I'm not publishing the same artifact as has been tested. All a technicality, but there's no real link between the jobs other than the code at the time of workflow run. I've made this decision because GitHub doesn't exactly have a huge free-tier storage limit (500MB) for artifacts, and I'm not willing to pay for this project. Maybe I can utilise GitHub Releases or similar here, or maybe I can simply utilise retention and have GitHub remove my artifacts every day. 
3. I'm not publishing anywhere other than one server - this means I have a single point of failure. For my homelab? Not the worst thing in the world. Ultimately nothing is impacted if this Ansible server dies. But again, not the point here.

### Release versioning
Now I am not ready to go for a fully-fledged versioning system like semver. I'd love my version numbers to be calculated according to how much has changed in the code, and I'll get there one day. But do not let perfect be the enemy of good. I'd contemplated a few options:
1. Have me manually specify a version (not ideal - humans are error-prone)
2. Use a sha checksum (I have an entire folder and I don't zip it at any point...)
3. Use the commit checksum (not bad, but not exactly human-readable)
4. Use the GitHub Actions run number 

I've gone for option 4 - the run number of that specific flow. It's not exactly a good version number, but it *is* a number, it's human-readable, it increments with every run and will provide me with some form of progress I can track. 

That leads me onto how I'm using this in my playbook.

### New and improved setup_web Ansible playbook
I wanted to fix the following issues:
1. The copy job running every time in some capacity
2. The inability to specify a version and check each web server
3. Have more than one published version of my app ready to go if needed.

1 and 2 are actually heavily linked - in order to evaluate whether a copy is needed, I needed *something* that's easy to compare. I could've gone the checksum route of a file, maybe zipped my published build and compared checksums, I could've created a file and compared it. But actually to easily compare, I needed a version number somewhere on the server. 

I've gone with a combo of an environment variable on the web servers and a playbook variable fed in with all the others. I have a group_vars directory that separates out my variables per environment, this means I can now have a different version running on dev vs prod! Huge win. 

I've utilised the following Ansible concepts:
1. ```Ansible_env```'s ability to capture environment variables from the remote host for reference in the script. In my case an environment variable in /etc/environment called APP_RELEASE. I might rename this API_RELEASE so that I can have a corresponding FRONTEND_RELEASE in future.
2. ```set_fact```'s ability to set a variable value for reference by other playbook tasks. I've set app_release and then app_release_matches that simply compares the environment variable with my playbook desired release number. True if they're the same, false if they're not.
3. A new handler that sets the APP_RELEASE environment variable on remote hosts, i.e. if the release version is different in my playbook vs the server's own environment variable then we need to set it.
4. ```when: not app_release_matches``` and ```changed_when: true``` - basically only run this entire task if needed. And if the task does run (because my web servers' release version doesn't match what I want) then make sure the task shows as 'changed' so that the handlers run.

I also now get multiple release-### folders in /opt/MealPlannerApi on my Ansible server now. Whatever run number the GitHub Actions flow is on, that's what the folder gets called. Not ideal if I create an entirely new Actions file, but it'll do for now.

## Benefits
This setup now lets me deploy on my terms, in an idempotent way, as often as I'd like. I could in theory script this playbook to run and get true continuous deployment. 

I can also specify different inventory files, have Ansible then pick up different group_vars files automatically based on the environment folder of the inventory, and deploy different versions to environments whenever I like.

Basically I have far more control over deployment now.

## Future improvements
Other than wanting a real versioning system, the most finnicky part is having to change a yaml value every time I want a new version for an environment. Okay it's changing one number, but it can be forgotten too easily. I don't like manual work. I don't see an easy way around this other than capturing input from the playbook itself. I've not investigated whether that's possible, but a bash script could be. 
