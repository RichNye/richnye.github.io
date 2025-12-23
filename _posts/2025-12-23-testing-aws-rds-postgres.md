---
title: Testing AWS RDS with PostgreSQL Restore
date: 2025-12-23 13:48:00 +0000
categories: [aws]
tags: [aws,rds,postgres]
author: richardnye
description: Testing AWS RDS with PostgreSQL and documenting the basic test environment setup and errors faced.
---

## Introduction
I created a new AWS free tier account a few months ago and the expiry date is rapidly approaching. I've been busy learning other tech (see basically any other blog post here), doing my actual day job, and life gets in the way. However, now that I have the groundwork laid with container images and a functional Kubernetes cluster, I wanted to deviate a bit and pick up some AWS again. I've decided to start with my Postgres database, because RDS is one of the fundamental AWS offerings, and figured I'd document my journey here.

Disclaimer: this is extremely quick and dirty. It's basically a mini how-to using the AWS GUI and pgAdmin, to get *something* spun up to test with as opposed to a full-on migration. Don't follow this if you're wanting a real production-grade migration path. 

## Migration strategies
From what I've seen, the preferred migration strategy is to actually create an HA pair with your RDS instance as the standby. This makes a lot of sense and is something I'll definitely be trying in future. 

What I've gone with, though, is a simple GUI-driven backup/restore using pgAdmin. I've gone with this because my app doesn't actually have any data manipulation traffic; it's simple reads only at the moment. This means I don't need a cutover window because there's no deltas that I'll be missing. 

## Creating the RDS instance
I used the easy creation route, letting AWS dictate most settings. I'm not doing anything special here, this is quick and dirty. 

One thing I did fall foul to though is needing to open my RDS instance up to the public, in a secure way. Now I might struggle with securing secrets in IaC, but when it comes to networking I find it fairly straightforward. I have a static IP at home so this was fairly easy. I did the following to modify my created instance:
1) I let AWS generate a password for my db user, and stored it in my private password manager.
2) I modified the instance settings to allow public access.
3) I attempted to connect to the instance using the provided URL (*.rds.amazonaws.com) with pgAdmin and received a connection timeout.

## Fixing the connection timeout
This was textbook firewall blocking, although in the AWS world it was VPC security group inbound rules. Now the docs do specifically say you need to do this, but I did figure it out on my own with the following:
1) Go to your RDS instance > Connectivity & Security tab
2) Click on your VPC Security Group to be taken to the Security Group config. Click on your Security group ID (I only had one listed here)
3) Edit inbound rules
4) Add rule
5) Type = PostgreSQL (note: I actually went for TCP custom and selected port 5432, AWS has then changed the type after I applied), source = My IP (which just populates the IP, AWS changes it to custom after) and for description I entered 'Home public access only'
6) Save rules

I then connected again using pgAdmin and was instantly hit with a different, scary error that referenced pg_hba.conf and no encryption.
```"no pg_hba.conf entry for host '<ip>', user '<user>', database 'database', no encryption"``` 
I've configured pg_hba.conf for my homelab, so was somewhat familiar here, but I found it odd that I'd have to configure something so granularly for a managed service. So I googled it and all sorts of stuff came up about creating a parameter group to turn off SSL encryption, but with public access configured this screamed a huge red flag to me. But it did get me thinking - if SSL/TLS is required, had it been configured in my pgAdmin connection?

Short answer: kinda, but not quite. 

Long answer: the SSL mode paramter was set to prefer, not required. Changing that in pgAdmin to required fixed my issue and satisfied AWS. 

## Restoring the DB
Great, so we're connected using pgAdmin - let's restore the DB. I simply used the GUI on my homelab, took a basic custom backup of my db, nothing special here. Then I went to restore it to my AWS server and realised a couple of things:
1) I don't have a database ready for this to restore to
2) I don't have a custom user created to own this db.

Cue more errors and fixing them...

### Fixing DB Creation Errors
Now creating the user was simple in pgAdmin, google that if you don't know how, and again I used a secure password knowing that this was public (albeit to my IP specifically). 

So I then went to create the database, chose the new user, and was instantly hit with ```ERROR: must be member of role '<user name>```. Now my user was the same as my database name which confused things a bit, but again Google to the rescue. Basically the user I connected to the databse instance needed to be a member of the new user role. Right-click your db instance user in pgAdmin > properties > Membership tab > add a new membership to the user (admin wasn't needed here) > Save.

That simple, in the end. Creating the new database with the new user as the owner then worked. Just a quick of Postgres here, I guess. 

The restore then worked and everything was happy. 

## Testing the AWS RDS Instance
My MealPlanner app is containerised, so that means I can just spin up an API container, feed in the right environment variables as part of the Docker run command and then test it in the browser. And voila! I have data being returned. 
