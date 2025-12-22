---
title: First Kubernetes Deployment of an ASP.NET Core App
date: 2025-12-22 14:00:00 +0000
categories: [Kubernetes]
tags: [kubernetes,devops,docker]
author: richardnye
description: Deploying an ASP.NET Core application to Kubernetes as a beginner
---

## Introduction
This week is all about Kubernetes and deploying an actual application/platform. In my [previous post](/posts/venturing-into-kubernetes), I'd spoken about spinning up a cluster using k3s and now it's all about actually hosting something meaningful in that cluster. Today we'll cover the process, mistakes I made, and where I envisage going next. 

First, though, let me give some context into the app I'll be running inside Kubernetes.

## Background - what app am I running?
I've covered this in numerous posts before, namely [containerising ASP.NET Core applications](/posts/containerising-aspnet-core-applications), so I'll keep this brief. My app consists of the following architecture currently:

1) HAProxy load balancer - this terminates TLS, routes depending on URL, does health checks on backend server instances, all the typical load balancer goodness. 
2) Frontend - this is literally a one page static site, served by nginx. 
3) API - this is an ASP.NET Core MVC API, using kestrel as the web server to keep things standard.
4) PostgreSQL database - runs on its own dedicated VM. Typical connection string used to connect and return rows.

The current architecture is dedicated VMs for each tier.

## Moving to Kubernetes
This is the end goal, purely from a career perspective. I'll be completely upfront that this feels like total overkill for such a simple three-tier web application. My app is extremely static, very simple, and doesn't see much traffic. But the goal of this blog is to map that transition from typical VM infrastructure to something more modern, possibly cloud-based. Kubernetes is the go-to that many companies require knowledge of, hence the move.

Kubernetes shifts the architecture in many ways. Here's a few:
1) It's no longer VMs based on app tier - what I mean by that is that it's not a case of web content on this VM, HAProxy on another, DB lives here. Now my app's VMs will become one of two things: part of the Kubernetes control plane, or a worker node that hosts containers. 
2) HAProxy gets replaced by Traefik - at least for now. I'm nowhere near advanced enough to debate the differences. I'm just following out-of-the-box common decisions, because I trust Kubernetes/k3s engineers more than myself here.
3) My entire CD pipeline needs to change - CI should remain fairly static in that we merge changes, test the code, build the container again, and upload it to Docker Hub. But CD changes completely, and I don't know how that looks currently. All I know is it'll no longer be Ansible copying artifacts to web servers.

## Quick Learning Check
Here's what I learned while studying the theory:
1) Pods = the actual containers being run.
2) ReplicaSets = ensure a specified number of the same pod are running at any given time.
3) Services = stable networking configuration in an unstable world. Pods are designed to be short-lived, IPs change frequently, and you can't guarantee a certain route will always be there. Services ensure certain networking config *is* always there. 
4) Deployments = a higher-level resource. We don't care about ReplicaSets or pods at this level. We tell Kubernetes what containers to run, how many we want, and the config we need to feed into them. Kubernetes makes it happen by utilising ReplicaSets and pods. Deployments can be modified and Kubernetes will automatically roll out those changes.
5) Ingress = opening up Services to the world, but only HTTP. Or more specifically, opening up Services externally to the cluster. Requires an Ingress Controller - k3s configures Traefik by default.

## Folder Structure
While this doesn't matter to Kubernetes, it does matter to me. My initial thoughts were to mimic the per-environment folder structure I've used everywhere else but then I realised this made no sense. I'll be using the same Kubernetes files everywhere, and instead utilising git branches when I need to test. So I went for the following tree:
```
├── api
│   ├── api-deployment.yaml
│   ├── configmap.yaml
│   ├── secrets-example.yaml
│   ├── secrets.yaml
│   └── service.yaml
├── frontend
│   ├── frontend-deployment.yaml
│   └── service.yaml
├── ingress
│   ├── mealplanner-api.yaml
│   └── mealplanner-frontend.yaml
└── namespaces
    └── mealplanner.yaml
```

I pulled out Ingress into its own folder. I went for this design choice because, to me at least, networking crosses app tier boundaries in a way that the other object types don't. I could have placed ingress.yaml files in each of the directories that needed to be exposed, but I opted against it. Now if I have an issue with ingress, for whatever reason, I have a central folder to check first. Then again, if I have an issue with ingress it'll likely be per-app anyway, so I'm torn here.

## Creating the Kubernetes Resources
I'll be referencing the yaml files inside my Kubernetes directory [here in my homelab repo](https://github.com/RichNye/homelab).

My overall flow was frontend > deployment then service > test. Once happy, I then tackled API > deployment > service > test. Then added ConfigMaps and Secrets.

### Frontend
I started here because it's the simpler of the two. I knew full well that my page would load but meals wouldn't, but just getting the page to load was the goal here.

**Deployment**<br>
I called it frontend-deployment.yaml because hopefully it stands out if I query many deployments. There's really not much to tell here, it's about as basic a deployment as you can get. The container needed no environment config, which made this a good beginner reference.

I'm trying to get better at memorising yaml syntax, not just here but for Ansible too, so that when interviews come around I won't look like an absolute moron. For this, I went for **AKMS** or apiVersion, kind, metadata, spec. Within spec, there's no hope yet. Far too much specified to make that easy to remember.

Anyway, I found deployments generally are split into two sections; defining the deployment and what pods to manage, then defining the pod workload in a way that looks similar to a Docker Compose. The app label in the former and container name in the latter must match.

**Service**<br>
Again, quite simple and follows **AKMS**. The app selector here must match what was specified in the deployment. I want my service to use the common HTTP port and bind that to the app-specific port I use (8090 for frontend) so I started that here. I could've also just bound 8090 > 8090 and made it more specific at the Ingress level, but I didn't.

**Ingress**
I opted to split my Ingress into separate files. The one in question here is 'mealplanner-frontend.yaml' if you want to take a look.

Again, extremely simple. Open up port 80 to all URLs. Similar to my HAProxy config currently where my frontend is extremely open, but my API route is more specific and takes precedence when /api/ is hit.

**Testing**<br>
I applied the config in the typical way ```kubectl apply -f filehere.yaml``` and watched as Kubernetes did its magic. One thing that I wasn't expecting was to have three available IPs to hit once the ingress was created. I later realised this was because Traefik has three instances and requests an IP for each. Neat. I'm assuming any load-balancing solution (even just DNS) in front of this takes all three to account for any possible node failures? One for me to research.

### API
I won't rehash my work again, the only difference here was the deployment and use of ConfigMaps and Secrets. I opted for Secrets for the DB connection string, adding secrets.yaml to .gitignore, and ConfigMaps for other ASP.NET Core variables like environment being set to Development. It turns out secrets.yaml was committed anyway, likely because I made the .gitignore change after. Lesson learned here - plan repo security before starting the work. I've deleted the file now and can clean up the commits... I really need to study proper git hygiene and flow.

For Secrets, I almost got caught out by Base64 encoding when I initially opted to follow the docs and use the ```data``` key instead of ```stringData``` which handles that for you. I'm not entirely happy with my knowledge of secure secret management in Kubernetes, and this has become a bit of a running theme throughout my homelab project. I did similar with Terraform, but at least I tried to consider security even a little ahead of the work.

I used ```valueFrom``` and ```configMapKeyRef/secretKeyRef``` here and it worked well to select the value and inject it into the pods as environment variables.

## Mistakes
I made a few. Let's list them here:

1) Forgetting what my container ports were when I built the image - this led to bad gateway errors, because the Service was trying to map port 80 > 80 instead of 80 > 8080/8090.
2) Forgetting to apply ConfigMaps and Secrets yaml. Yes, they need applying like any other Kubernetes object! 
3) Forgetting to specify the mealplanner namespace - my Ingress had no idea about the Service at one point because it wasn't in the same namespace.

I found ```kubectl describe``` extremely valuable here to see the actual events like a ConfigMap file not being found, or the Service not being found. 

## Future Work
I'll just rattle these off because if you've read this far then you're probably getting sick of this post by now:
1) Customise Traefik - currently it's all defaults. No TLS, no max connections, nothing. 
2) Logging - I need a similar request log as I get now from HAProxy. Not just Traefik, though, I also need logging from my pods, logging pod restarts so I can set up alerts, the lot. 
3) Resource limits - currently my frontend/API could battle each other and impact the other. I need to specify CPU/memory limits.
4) Add a host to my Ingress config - currently it'll accept any host, ideally I need to restrict this to something like kubernetes.meals.rnye.tech and either customise my hosts file or setup global DNS in Cloudflare.
5) Add various probes like readiness and health - currently we're flying blind a bit.

And generally do more research now that I've got my own resource declarations to compare to. 

As always, do reach out if you've got any feedback!




