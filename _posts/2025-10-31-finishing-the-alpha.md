---
title: Finishing the website alpha
date: 2025-10-31 21:28:00 +0000
categories: [homelab]
tags: [homelab,devops,website,dotnet,nginx]
author: richardnye
description: Hitting the first milestone in the project with haproxy, nginx, a web frontend to the ASP.NET Core API all being added.
---

## Finishing the website alpha
Hi, everyone, and welcome to what is basically a recap of this week's work on the homelab. This week saw the absolute calamity that was losing my Terraform state file (full write-up of that recovery [can be found here](/posts/recovering-from-terraform-state-loss)) and that frustration gave me a kick. I really wanted to have something tangible that I can deploy and show people. To recap, I have an ASP.NET Core MVC API; it's a simple CRUD app for me to dive back into C#, learn Entity Framework Core, dive into Postgres and the world of database design. But that's not fun to show people. Friends, family, non-technical people generally, don't exactly get excited when they see a load of JSON returned instead of a fancy looking list of meals! 

I'll reiterate what I've said countless times before on this blog - I am not, and will never be, a frontend dev. It's not where my passion lies, at this time anyway! Who knows? After this I might catch the JavaScript bug. What this is basically saying is that I leveraged AI to help me create a single HTML page that contained everything needed to query my API and display it nicely. 

And I did that for one simple reason - this project, as it currently stands, is all about hosting an app/website as if an in-house dev team was developing it. For me, it's not about the app itself, it's about the fact that I've (with the help of AI) have made something that I can continually develop. But more importantly it's about recreating the journey that a lot of businesses are facing today; moving from a traditional website setup (we'll touch upon that) on VMs all the way through to the first containerisation project and then moving to the cloud. 

## Adding a load balancer
I had the web server running the API (kestrel and a self-contained build published for Linux), I had the database server running Postgres, and I'd exported the initial migration SQL and created the MealPlanner database. Great! What I lacked was any form of ability to scale out. This is critical for a couple of reasons:
1) What business in the world only runs one web server? 
2) As I containerise the app, the web server is the easy place to start. I want to run containers alongside the web VM and eventually decom the VM.
3) I want the benefits of running a load balancer, particularly haproxy which is layer 7 aware. That means decrypting SSL there instead of every web server, being able to redirect to different servers (will become apparent later) based on the request, and health checks! 

I've gone with haproxy rather than nginx because I see nginx as a web server first and a load balancer second. If haproxy didn't exist, I'd probably use it; but it does, so I won't. So I made sure a load balancer-specific VM was created (thanks Terraform), and got to work creating the ansible playbook 'setup_lb.yaml'. [You can see that here.](https://github.com/RichNye/homelab/blob/master/ansible/playbooks/setup_lb.yaml) It was my first introduction to Jinja templating, and I'm now converted! The only thing I want to improve is adding an haproxy config check prior to copying it, but that can come later on.

## Adding a web frontend
I'll keep this brief - I used AI to create the index.html file, but I created the Ansible playbook that sets up nginx and the new website on existing web servers. Kestrel is great, and I could quite easily have configured my API project to include this file and have kestrel serve it. But I want to stick with the microservices theme here; in short, I want my frontend to be completely separate. 

So I created this playbook, ran it, and job done. You'll notice that my haproxy config redirects to kestrel if it's a /api request and nginx for everything else. A design I find really clean. This will allow me to scale the frontend separately from the API in time, once I've containerised everything. Of course, haproxy will likely eventually die here in favour of some sort of Kubernetes load balancing solution, but it's a great stop-gap until then. 

## Configuring the site
I wanted the ability to protect the site, not have to worry about SSL certs (this week at least) and gain the many quality features that Cloudflare provides. I'm not a salesman, I just love them as a homelabber. So I've configured my website to be served through Cloudflare and restricted it to solely the UK for now. DNS is proxied and points to my home IP, and they take care of the certs for me. I still obviously need a certificate between Cloudflare and me, but for now it's a nice, easy setup. I'll be leveraging Let's Encrypt soon.

I've gone with https://meals.rnye.tech and it works wonderfully. I did run into some CORS issues, but that was because I had originally created my index.html file to reference my VM name rather than the subdomain here. I also had CORS issues because the index.html file was calling HTTP rather than HTTPS - it turns out mobile browsers will redirect WhatsApp links to HTTPS automatically? Madness. 

## Future Plans
For now, I'm really happy. I have something I can actually demonstrate to people, and also use myself. It's so difficult to now split my time because I essentially have two projects; the devops config and the next stages of this (containers!) and now this entire app. I've had fantastic feedback. It seems to be solving a problem we all have - we're fed up of eating the same 10 meals over and over despite having multiple cookbooks. I'll do another post here about the app itself, the problem I was overcoming, and the numerous features I'd love to add. But for now, the architecture is essentially this:

**User request --> Cloudflare --> haproxy --> web server --> Postgres and back.** 

A classic architecture, and that's basically the first milestone reached. All with IaC. Now I want to perfect that IaC so that I can literally tear this all down and rebuild in less than 2-3 minutes. We'll see how that goes!

As always, I'd love to hear your feedback. Please do get in touch!

