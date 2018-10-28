---
layout: post
title: "Salesforce deployments with PowerShell"
author: Javier Garcia
description: "A script to deploy and test salesforce metadata/apex e2e."
category: salesforce
apprenticeship: false
tags: salesforce, powershell, automation, deployment
---

Recently I've been learning a little bit of [PowerShell](https://docs.microsoft.com/en-us/powershell/), initially for the sake of automating some tedious tasks I had to do at work like calling a REST endpoint 10-20 times to populate some DB for testing or simply pipelining some CLI commands.

As I got more comfortable with PS, I started to realize the huge amount of things that can be done with scripts  which make our day to day easier. The biggest one that came to my mind was Salesforce deployments.

Usually, once we've finished developing a certain feature in Salesforce, we run all tests, pull all our changes from the environment where we were developing (*In Salesforce, you develop remotely, not locally*), deploy to a staging environment and run again the tests there.

The issue is that that process involves a lot of manual steps, so I decided to try and automate it. For this, I looked into the [SFDX CLI](https://developer.salesforce.com/tools/sfdxcli) and tried to reuse most of its features: an authentication command, the actual command which calls the deployment endpoint, etc.

Eventually, after a few hours, the result was this:

<script src="https://gist.github.com/Manzanit0/3908ad6ba3136fcde328a4eccdd5613f.js"></script>

It's a little nifty script which prompts a tab in your default browser so you log into the destination org and pipelines the deployment and the running of tests with real-time status in the console. That I did simply by polling the API.

There are, of course, many things to be improved in the script but, as a starter, it's already saving me a lot of time! :)