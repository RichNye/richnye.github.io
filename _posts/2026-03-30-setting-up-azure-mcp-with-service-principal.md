---
title: Setting up Azure MCP Server with Service Principal
date: 2026-03-30 11:48:00 +0000
categories: [azure]
tags: [azure,mcp,ai,devops]
author: richardnye
description: How to configure Azure MCP Server with Claude Desktop using a service principal for least-privilege access, including app registration and RBAC.
---
# Introduction
Today I'm going to focus on how to set up Azure MCP Server, but more specifically how to use a service principal to authenticate. During my setup, and I've done this with both MacOS using npm and Windows using Docker, I really struggled to find a simple guide about how to use Azure MCP with a service principal so thought I'd document my own setup to hopefully help the community.

# Why Azure MCP?
I don't know about you, but our Azure estate is vast. In fact it doesn't take long for an Azure estate to become difficult to analyse, in my opinion. Yes there's Azure Advisor, but that takes time and sometimes feels like a bit of an upsell - yes I'm aware my VMs are running as B-series SKUs, I did that for a reason! We're seeing all these articles and buzz about speeding up app development using MCP, and for good reason, but what about Infrastructure? 

## Potential Usage
The world really is your oyster here. Personally I've used Azure MCP to highlight inefficiencies within our environment and I'm excited by the potential. For example, I wanted it to act as an Azure architect and FinOps expert to highlight cost-saving, and it did just that. It found a VM that looked fine on the surface but was failing to deallocate, numerous potential orphaned Veeam backup resources, and inefficient use of public IPs on VMs that were behind an Azure firewall. All basic stuff, but really useful when you have a team full of varying Azure experience and skill levels. 

It wasn't perfect, though, and there's the usual disclaimer when it comes to AI - it's a tool that should never drive you. Every recommendation it's outputted (in a lovely dedicated markdown doc, no less) will be manually checked and confirmed before I go deleting resources. But the speed at which it analysed, and the thorough report it produced, has left me excited to see what's next. One glimpse at the Azure MCP Server docs and the tools it offers should be enough to get the creative juices flowing for anyone reading this!

Where I personally want to go next is getting it to analyse log analytics, triaging alerts before escalating to a human (with context provided by Confluence docs and the Azure estate), and the like. I think it's worth creating a plan, with phases, as part of the PoC here. Document what business challenges you're trying to solve, and how much money and time this might help you save. What are your pain points as a team?

But it can go beyond that. We have two Azure subscriptions that are currently not infrastructure-as-code, and it was going to be a daunting task creating Terraform that not only encompasses all resources that already exist, but also follows our other subscriptions in IaC style. What if I combine access to the IaC repository with Azure MCP? Suddenly this task potentially becomes *weeks* quicker. I'm yet to test that but I'm excited to. 


# Prerequisites
First, a brief overview of a variety of things. Now I won't be covering what MCP is here, I feel like there's already a plentiful supply of learning resources online, but I will be covering the why and how.

## Why a service principal?
Now I've noticed that Microsoft documentation [like this](https://github.com/microsoft/mcp/blob/main/servers/Azure.Mcp.Server/README.md#package-manager) has you doing an interactive sign-in via Azure CLI or PowerShell first, things like ```az login``` for instance. But this didn't sit well with me for a couple of reasons:
- I use a separate admin account which has permission to create, edit, and delete resources, but I wanted this PoC to be strictly read-only.
- I want to start treating AI like a unique identity. 
- For this PoC I wanted to make authentication a bit more portable so that other team members can try it. It's read-only after all, and limited to one subscription.

So I created a new app registration in Entra, created a client token/secret, and provided that app registration with the permission I wanted. I felt an entirely new identity was best here because I could guarantee it wouldn't have existing permission elsewhere. I wanted a good guardrail in place because this is unknown territory - yes the AI model usually asks for permission before acting, but a single misclick or lapse in concentration could give it permission to modify resources; a huge no for what I was attempting here. The service principal provides another guardrail at a higher level.

## Choosing how to run MCP Server
The documentation gives a variety of options, mostly centred around using some form of package manager: NuGet, NPM, PyPI, are the three [listed here](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/get-started). But I've been playing around learning MCP by taking advantage of the Docker Desktop MCP Catalog of MCP Servers, and I really wanted consistency in how I'm running MCP with Claude Desktop. [Microsoft also handily document exactly that scenario here.](https://github.com/microsoft/mcp/blob/main/servers/Azure.Mcp.Server/README.md#docker). 

Note that I did also have good experiences with MacOS and NPM, so really it's up to you how you run this. What I will say is that make sure you're familiar with the generic Azure SDK authentication because the environment variables available to you stem from that; they're not unique or created for Azure MCP.

## Choosing an MCP Host
This is ultimately a personal decision. I've personally got this working with both Claude Desktop (my preferred MCP host) and VS Code using the GitHub Copilot extension. What I've noticed is that the flow seems fairly similar - you'll setup the MCP server, plan how you'll pass in environment variables at time of Azure authentication, and modify a json file to configure the MCP host to use the MCP server.

# Setting up Azure MCP Server with Service Principal
## Creating the Service Principal
I won't go into great detail here, this isn't a tutorial on creating a service principal, but you use Entra to first create an app registration, configure how you'll authenticate (I went for a client secret with a short expiry, but certificates are a good option here too), and then assign the app registration/service principal some Azure permissions. It's worth getting this done first because you need the tenant ID, client/app ID of this app registration, and the client secret or cert later on.

## Configuring Azure RBAC permissions
As mentioned, I strictly wanted this PoC to be read-only. And the best guardrail I could think of here was Azure RBAC. I wanted something at a layer above the AI so that even if it attempted to make a change it considered worthwhile then it wouldn't be able to. But it's not just about that, it's also about the scope of the PoC, and this will be personal choice. I personally wanted to restrict it to one subscription, but the subscription in its entirety, so I went for the Reader role. For additional security, you could also choose to time-limit this assignment, which when combined with a certificate or short-dated token, could prove effective.

## Configuring the environment
### Docker Desktop
**Basic Docker Setup**<br>
Obviously you need Docker Desktop installed and in your Path environment variable. You should be able to run docker commands in the terminal. If you want to speed up the first run, considering pulling the Azure MCP latest build [found here](https://mcr.microsoft.com/en-us/artifact/mar/azure-sdk/azure-mcp/tag/latest) to guarantee the image is available locally.

**Authentication**<br>
Next up is authentication. With Docker, Microsoft recommends a .env file containing AZURE_TENANT_ID, AZURE_CLIENT_ID, and AZURE_CLIENT_SECRET variables, though I assume other variables for certificates like AZURE_CLIENT_CERTIFICATE_PATH and AZURE_CLIENT_CERTIFICATE_PASSWORD will also work here. Though it's worth noting that if you do go the certificate route then you'll need to pass the cert file into the container as part of the Docker run config you'll do in the next step, you could use a bind mount for that but I've not tested it. I went with a client secret.

It's also worth noting that you're storing the client secret in plain text here, and you can also inspect the container to see it. Not ideal and well worth spending some time investigating better and more secure methods. This was acceptable for me given the guardrail in place of Azure RBAC, plus it was all local on my machine, but well worth assessing the risk before proceeding.

**Configuring Claude Desktop**<br>
This was thankfully quite simple thanks to Microsoft. They have a great example MCP config file and I worked with Claude to get mine working. Here it is:
```
    "Azure MCP Server": {
      "command": "docker",
      "args": [
        "run",
        "-i",
        "--rm",
        "--env-file",
        <ENV FILE PATH HERE>,
        "mcr.microsoft.com/azure-sdk/azure-mcp:latest"
      ]
    }
```
I've redacted my .env file path, simply replace it with something like ```"C:\\path\\to\\.env"``` and note the need for escaped backslashes! Claude Desktop stores its env file by default in ```%APPDATA%\Claude\claude_desktop_config.json``` but you can also go to Settings > Developer to find it in Claude Desktop.

**Testing**<br>
Simply reboot Claude Desktop and it should launch a Docker container. Copy your subscription ID and do a test prompt like ```Please test my Azure MCP Server connection by pulling back all resource groups. The subscription ID is 1111111-fffffff-bbbbbb-ggggggg and you should have access.``` - if you get resource group names returned then setup complete!

# Lessons Learned
I'll start with the negatives before moving onto the exciting potential. Firstly, I don't like the idea of having these environment variables in plaintext on my machine, and I think it's worthwhile seeing exactly how I can take advantage of something like Azure Key Vault or the myriad of secret management solutions available going forwards. Even if that means some sort of script to launch this MCP server. I need to spend time strengthening this side of it.

Secondly, the recommendations weren't always fantastic. It lacked context of the wider estate, so don't expect this to be a silver bullet. Like with everything AI, you really do need to put the work into not only crafting a great prompt but also making sure it has the context it needs to make informed decisions. For me, that looks like configuring Confluence, repository access, and who knows what else. It needs thought.

Lastly, you really need to consider data governance here. You're opening up an Azure estate to an LLM. The use of a business/enterprise plan should go without saying, and ensuring the LLM won't be using your Azure estate to train future models is critical.

Now onto the exciting stuff! I found great benefit in going the service principal route rather than interactive auth. I really didn't like the idea of this AI agent acting as me, with all the permissions of my account. This is precisely the problem that service principals are designed to solve; it's a third-party app accessing Entra/Azure resources, treat it as such. The fact it's Azure SDK under the hood really enables you to tweak it to your needs, and you can take some solace in the fact this is using battle-hardened tech underneath.

It's also given me several recommendations to take forwards. Yes, I might have moaned about how they weren't always correct (and that's mostly on me anyway), but it only took a couple of minutes to completely analyse my estate, understand how Veeam for 365/Azure works, realise that's what we were using and cater suggestions to it. It found resources we've not been aware needed cleanup (because as a team we're simply too busy), and would even help me craft a change request if I asked it to. That task is easily several hours, if not more considering the breadth of tech we often use in this industry. 

I'm really excited to see how it can solve our other problems.
