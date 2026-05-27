---
title: Using AI as an Azure FinOps agent to reduce Azure spend
date: 2026-05-27 20:28:00 +0000
categories: [azure]
tags: [azure,mcp,ai,devops]
author: richardnye
description: Using AI as an Azure FinOps agent to find under-utilised Azure resources, misconfigurations, and inefficiencies to reduce Azure annual cost by £2k.
---

## What is Azure FinOps and how can AI help?
As mentioned in my [previous post](/posts/setting-up-azure-mcp-with-service-principal/), I've been investigating the use of Azure MCP to improve cloud operations. That could involve analysing the architecture against best practice, finding security vulnerabilities, among many others. In today's post, however, I want to discuss a recent successful PoC I did that utilised Claude Sonnet 4.6, with GitHub Copilot and Azure CLI/MCP, to find financial inefficiencies in an Azure subscription. 

Now the concept of FinOps is a fairly simple one; it's about analysing cloud spend, attributing cost to the right business areas, and ensuring every pound spent is worthwhile to the business. It's not solely about cost-saving but that will inevitably be a benefit to a streamlined and efficient cloud environment. Simply put, I wanted to know that the resources and configuration we had were serving a purpose, correctly sized, and being utilised. This meant analysing every resource, every configuration setting of said resources, and cross-referencing that with Azure Advisor and general Azure knowledge. It's not a small task, it can be repetitive, and can require online research and knowledge before acting. Simply put, I foresaw weeks of work just for one subscription, with a monthly cost of around $5k a month - not huge, but not small either. Enter AI.

AI (LLMs specifically) can be fantastic at quickly analysing data. I won't get into the arguments for/against using LLMs in a technical environment, but it can do it far quicker than I can manually, so it was worth a try. The goal here was to utilise the connection to Azure I'd configured in the previous post, have it assume the persona of an Azure FinOps expert, have it analyse the entire subscription (Reader access only!), and have it output its recommendations to markdown for me to review. From there, I would use my experience within the organisation to make informed choices with business context it didn't have. I also wanted it to estimate cost-saving for each recommendation so that I could track the PoC savings to see if it was worthwhile going forward to other subscriptions (spoiler: it was!)

## How to prompt an AI for Azure cost analysis
I purposely wanted to split up the prompts per main Azure service - i.e. computer analysis would be separate to network or storage analysis. Having analysed my subscription costs using the standard cost management tools in Azure, I identified the following core areas:
1. Compute
2. Storage
3. Network (including firewall, bastion, and public IPs, to name a few)
4. Log analytics and monitoring - which can get quite costly if the environment is noisy.
5. PaaS analysis (databases, app services, containers, etc.)

I essentially followed the same prompt for all, with obvious tweaks depending on area. Here's an example prompt structure I used, specifically for log analytics and monitoring analysis:
```
You are an Azure FinOps analyst. Analyse Log Analytics workspaces and monitoring costs in Azure subscription {SUBSCRIPTION_ID}.

Use whatever tools and data sources are available to you to gather the information needed.

ANALYSIS TASKS:
- Inventory all Log Analytics workspaces: pricing tier, retention setting, location
- Identify workspaces on Pay-per-GB where ingestion volume suggests a commitment tier would reduce cost (threshold: >10 GB/day)
- Identify the top ingestion sources by table — flag any that are high-volume but likely low-value (e.g. verbose diagnostic tables, excessive polling frequencies)
- Identify resources sending diagnostics to multiple workspaces
- Identify workspaces with retention set beyond 90 days where archiving to cold storage would be cheaper
- Determine whether any workspace has Microsoft Sentinel enabled and estimate its contribution to ingestion cost
- Identify diagnostic settings that send all log categories where only specific ones are needed

OUTPUT FORMAT — produce a markdown file named log_analytics_analysis.md with this structure:

# Log Analytics & Monitoring Analysis
**Subscription:** {SUBSCRIPTION_ID}
**Date:** {DATE}

## Workspace Inventory
[table: Workspace Name | Pricing Tier | Retention (days) | Location | Monthly Cost]

## Ingestion Volume by Table (Top 10)
[table: Table Name | GB (30d) | % of Total | Notes]

## Commitment Tier Assessment
[table: Workspace | Current Tier | Avg Daily GB | Recommended Tier | Est. Monthly Saving]

## Diagnostic Settings — Duplication & Noise
[table: Resource | Destination Workspace(s) | Flags]

## Sentinel
[If enabled: estimated ingestion cost contribution and data connector breakdown]
[If not detected: "Sentinel not detected in this subscription"]

## Retention Cost Optimisation
[table: Workspace | Current Retention | Est. Saving if Data Beyond 90d Archived]

## Cost Summary
[table: Meter | 30-day Spend]

## Gaps
[⚠️ Further analysis warranted items]

CONSTRAINTS:
- Do not reproduce Azure documentation or generic best-practice advice
- Use ⚠️ Further analysis warranted wherever data is insufficient
- Prefer tables; keep under 600 tokens
- If a section has no findings, write "No significant findings"

```
I actually also used AI to refine the prompt, and it performed well, I think. Prompt engineering is an entire skill in itself these days, but this appeared to work fine. Feel free to reach out with any suggestions.

## What AI found and how I tracked potential savings
Now this will be different for every environment, depending on a variety of factors from the cloud literacy of the team through to how overloaded with work the team are. My general rule in tech is people will always create, but very rarely clean up. This means that as an environment ages, the inefficiencies worsen as resources are created, often hidden deep in the menus of Azure. Log analytics, snapshots, backup policies for resources that are turned off but never decommissioned, the list is endless. 

At a high-level, it found the following:
1. An entire Azure Bastion resource that had no sessions for at least 30 days, likely more. It was costing a significant amount per month and was never cleaned up when the system it was utilised for had been decommissioned.
2. Numerous premium SSD disks that simply weren't required. Downsizing these to standard SSD saved around £8 per month per disk. 
3. NSG flow logs for every NSG - yet nobody was reviewing the data. A change request was found that showed these flow logs were enabled as part of troubleshooting a now resolved issue, but had never been turned off. 
4. The Dependency Agent on several Windows VMs - it's been deprecated, not replaced by Microsoft, and nobody was even reviewing the data, nor were alerts configured. 
5. 50+ disk snapshots, created by Veeam Backup but never cleaned up because of Veeam failures over the years. Totalling around £50 per month in total, dating back two years.

Before any changes were made, I used an Excel spreadsheet with columns such as estimated monthly saving, Jira ticket reference, actual cost savings (after a month of waiting), and any notes. I also grabbed screenshots of Azure Cost Management filters per recommendation/change for a fair comparison.

## Don't blindly trust AI recommendations!
It also recommended downsizing Azure Firewall, but this simply isn't practical for our environment. And that's where business context and not blindly trusting the findings are key - it's a useful first review, that can often find things humans might miss, but not the be all and end all. Use it smartly and reap the rewards! 

Also be aware that the LLM provider will gain valuable and sensitive data regarding the core infrastructure of a cloud environment. I cannot recommend enough having some sort of commercial agreement with the LLM provider - in our case it's GitHub Copilot licensing, with Microsoft assuring customers that it won't use their data to train models. Don't simply hook up to any LLM, consider compliance first.

## Summary
Using AI saved over £2k annually, on a single subscription, roughly 6-8%. It was also staggeringly correct in its cost-saving predictions, almost to the dollar in some cases. Keep track of recommendations and note how you'll measure cost-saving per change conducted, and see what it comes back with.