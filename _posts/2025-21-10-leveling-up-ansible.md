---
title: Leveling Up Ansible Part 1
date: 2025-10-21 20:21:00 +0100
categories: [ansible]
tags: [ansible,sysadmin,devops]
author: richardnye
description: Adding Ansible handlers, group variables, Postgresql improvements, and accounting for dev and prod environments.
---

## Leveling Up Ansible
As discussed [in the previous post](/posts/introduction-to-ansible/), I'd started creating Ansible playbooks to reasonably good effect. It worked well when I had one environment but was lacking in several areas as that post outlined. I've started dabbling in Ansible vault, but we won't be covering those efforts today - that'll come in part 2. 

## Ansible Handlers
Up first is one of my favourite changes. As I discussed in the intro post, I hadn't quite got in the true mindset of Ansible. I was very much in the declarative, almost scripting, mindset where I expected a set of steps to happen rather than declaring the ideal state and what to do if changes were needed. This meant, for example, that I was reloading systemd every time, restarting services when it wasn't warranted, and generally not accounting for a stable environment after the first playbook run.

Enter Ansible handlers.

I've gone from this:
```
  - name: Reload systemd to pick up new service
    ansible.builtin.systemd:
      daemon_reload: yes

  - name: Restart MealPlanner API service
    ansible.builtin.systemd:
      name: mealplannerapi
      state: restarted
```

To this much cleaner look:
```
  handlers:
  - name: reload systemd
    ansible.builtin.systemd:
      daemon_reload: yes

  - name: restart mealplannerapi
    ansible.builtin.systemd:
      name: mealplannerapi
      state: restarted
```

These handlers only fire if notified by tasks, which is perfect. I've kept their order the same because I want systemd reloading before restarting services. It's so much cleaner and the playbook is running better on servers that have already been deployed as a result. 

## Group Variables and Multi-Environment Setup
One environment is great but it didn't suit for a number of reasons. Firstly, it's not what enterprises are about. Secondly, I've hit a point where I'd like multiple builds of my API running, plus I'm planning on building a front-end in React and don't want that interfering with my lovely and stable prod environment. Thirdly, I'm actually using my meal planning tool to... plan meals! 

As a result, I knew I need to make these playbooks flexible and generic quickly. Traditionally, to me that means variables. It's what I'd done in Terraform, PowerShell, you name it, variables are typically the answer to reusable and flexible code. Ansible is no exception here. I went with the following directory structure:
```
.
└── inventory/
    ├── dev/
    │   ├── group_vars/
    │   │   └── all.yaml
    │   └── inventory.yaml
    └── prod/
        ├── group_vars/
        │   └── all.yaml
        └── inventory.yaml
```
This wasn't easy - I remember having a real battle trying to get the group_vars yaml file recognised when I specified the inventory in the ```ansible-playbook``` command. But this hierarchy works really well. To run it, I do the following command ```ansible-playbook playbooks/setup_db.yaml -i inventory/dev/inventory.yaml``` and just change the environment of the inventory file. 

I went with all.yaml initially because it's my playbook that filters the inventory for a given group of hosts. Perhaps I'd be better off splitting these out into something like db_servers.yaml and web_servers.yaml, but I need material for the next part in this series! 

## Postgres Improvements
So many changes to the setup_db.yaml playbook! So this was mainly about automating the final things I had to do when manually when setting up the new dev environment. Things like configuring pg_hba.conf to allow the mealplanner database user to actually sign in from the Ansible control node, configuring listening addresses so that the db is available to the web servers, and obviously sorting the handlers out. I finally merged all the package installation tasks into one too, what a world.

Once that's merged into my master branch, definitely check it out to see the improvement. 

## Future Work
I still need to secure my secrets. I started doing it as part of the same work above but I need to hit the videos and revisit Ansible vault. It definitely seems like the way to go without going the cloud route. I'm not opposed to that, but it does mean cost for very little benefit here. That'll be one for the roadmap once I'm happy "on-prem" is complete. 

