---
title: The Four Things I did to Fix Alert Fatigue
date: 2026-03-27 17:48:00 +0000
categories: [monitoring]
tags: [azure,monitoring,sre,devops]
author: richardnye
description: How I tackled alert fatigue with four practical changes to monitoring in Azure
---

# The Problem and Setting the Scene
## The site
I'm supporting an eCommerce website, all hosted in Azure, with multiple internal and external integrations. There's nothing unique here, the world runs on this sort of stack.

## Death by email
What is unique is the alerting setup, and I don't mean that in a positive way. The team receives hundreds of Azure Monitor alerts daily. That's not an exaggeration and it's all driven from Azure App Insights alert rules. 

Why this is worth a blog post is because of the learnings I've found from it.

## Alert Blindness (aka Alert Fatigue)
For those that have never come across this term, it's basically where alerts fire so much that you become desensitised to them. The thought of "oh this has been firing for ages, and the last few times I investigated, it turned out to be false-positive," so you just... don't investigate. 

That's exactly what has happened to this team. Alerts may be investigated periodically, but it's already led to slow response to outages, or a complete miss entirely. Alerting shifts from a system telling you to waiting for a senior business manager to shout.

# How we got here
I wanted to answer the following questions:
- what are the alert definitions?
- what alerts are firing the most?
- why have these alerts been created?

Standard stuff, but it's led to some interesting findings. Ultimately it's a significant number of alerts that are quite wide in scope, mostly trying to be aware of any errors that are happening. 

It's important to remember that the intentions are clearly good; they want awareness of errors to identify bugs, service outages, etc. The issue is there are simply too many alerts that are firing too often. 

## Dev vs SRE alerts
There's been a battle between Ops and Dev - Ops want the alerts reduced, Devs want them to remain so that we're aware of issues as they happen. Emails will keep firing because there's resistance to stopping them and it's a hard argument to make because both sides have valid points. So don't; invest your energy into creating a new solution that works.

This has led me to an interesting realisation; there are two types of alert at play here: those for developers, and those for service uptime. To put it another way, there's alerts for functionality and there's alerts for availability. 

## Email as an alerting tool
It just shouldn't be. It's an asynchronous method of communication trying to serve a purpose that requires an instant response. The two don't add up. And yet people get into the habit of configuring alerting emails, because Outlook happens to show a notification when an email arrives on most platforms. Please stop.

What I've learned is that most of what a developer wants to be alerted about could be a dashboard, or a log query run ad-hoc or daily. It might be convenient to receive the odd alert, but when it's 20+ alert rules firing hundreds of times a week, that's not helping anyone. You shouldn't need to create multiple Outlook rules just to survive the day and remain productive.

# The Solution
## 1) Don't try to fix the current alerts
People are resistant to change, particularly when the situation has been going on for years, or the odd person might be using those email alerts, or there's a valid argument for why the emails come in.

So just don't try to fight the battle. The goal here isn't to stop emails, though that would be nice, the goal here should have the business in mind. So what's the goal? **Become aware of system availability issues before the business does.**

That's it. I want to know when the system coughs immediately. I don't mean a slight, pathetic cough. I mean a cough that impacts revenue. I want the business to have ultimate faith that their IT team are reliable. Even if there's nothing we can do because a vital third-party has an outage, it always looks better if business leaders have IT inform them, not the other way around.

## 2) Email is no longer viable
So use literally anything else. Alert blindness and fatigue has ruled out email entirely for alerting. That could mean going all out and using something like PagerDuty, but it could also be as simple and cost-effective by utilising something like a Teams channel, or even the Azure app. It just needs to be something completely new to the business.

For the new alerts, I'm experimenting with the Azure app. We don't use it currently and it's free. It also allows us to specific target users in the same way as email, keeping alerts specific. I want users to have somewhere completely new and uniquely for alerting. Whether that's the Azure App remains to be seen, but the point is it's significantly different from email.

## 3) Create new alerts
I know this sounds counter-intuitive, but bear with me. Create what I've been calling "sister alerts". Our current alerting isn't bad, but it'll be difficult to justify a change to developers because they're using these alerts. The issue is the alerts are too broad, things like throwing an error if any non-200 response is received from an external system. But in some instances, a 404 is a valid business response caused by our customers. 

So take the existing alert and refine it greatly. Target 5xx or timeouts. What does an actual outage look like in the logs? Make your thresholds high enough so that they don't trigger every 5 minutes. Here's an example of the old alert:
```
dependencies
| where target in (<redacted>)
| where success == false
| where resultCode !in ("200", "0")
| project timestamp, resultCode, target, operation_Id
| order by timestamp desc
```
You can clearly see how wide it is - basically any failed, non-200 response triggers it. The threshold is 10 log entries in 10 minutes. There's no percentage considered of failed vs success and it suffers during peak traffic periods.

The new alert:
```
dependencies
| where target in (<redacted>)
| where resultCode startswith "5"
    or isnull(resultCode)
    or resultCode == ""
| summarize 
    TotalCalls = count(),
    Failures = countif(success == false),
    FailureRate = round(100.0 * countif(success == false) / count(), 1)
| where TotalCalls >= 10  // ignore windows with negligible traffic
| where FailureRate >= 30
```
This handles it very differently and has already proven effective - the old alert has fired 4 times yet this one is yet to trigger. As mentioned, we only care about external server errors, things that result in that service being unavailable; 5xx and timeouts are the key here. Not only that, but pulling back the total and calculating the failure rate means we can now trigger the alert based on failure percentage. We can also only trigger the alert if we're receiving meaningful traffic. 3 logs at 2am isn't enough data, and 1 failure would equal 33% failure rate. So why bother? 

It requires a change in alert threshold - we're looking for 1 row to exist. This query either returns a row if the failure rate and traffic conditions are met, which should trigger an alert, or it returns nothing (so all is well).

## 4) Alerts should prompt action always
I want to hammer this home to the team. If an alert fires, we should have to act. It shouldn't be an option to simply ignore the alert, because then what was the point of the alert? Ideally this would initiate some sort of failover to a different external integration, but that's not always possible commercially or technically. At the very least it could be a status update, an email letting the business know, a message on the site or socials. Whatever suits your business and situation.

 For us, if these new sister alerts fire, then the following action should occur, without fail:
- A runbook is followed by the team.
- The alert should be tuned if it's a false-positive.

Without fail. Either action was required and the runbook was needed, or the alert was wrong and needs to be refined. Every firing of the alert should be taken seriously, always, and if it wasn't needed then apologise to the team, say what you've done to refine the alert, and make it a big deal. It's vital the team maintain confidence in this new alerting.

## Summary
Too many alerts and using email as an alerting tool is not the one. It's also worth remembering that devs and ops will care about different things sometimes, and alerting looks different for both. Make your alerts require action, and make sure they fire *only* when there's an outage. Ideally sparingly. 

I'll be creating a post on using AI to aid this problem, and some ideas I'll be investigating so keep an eye out.