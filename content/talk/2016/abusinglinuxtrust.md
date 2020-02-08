---
title: "Thotcon 2016 - Abusing Linux Trust Relationships"
talkTitle: "Abusing Linux Trust Relationships: Authentication Back Alleys and Forgotten Features"
conference: "Thotcon"
location: "Chicago, IL"
date: 2016-05-06
summary: ""
type: "talk"
talkSummary: 'Passwords are weak, and generally speaking, the less a company relies on them, the better. Instead of using password authentication for multiple services and sending passwords (or hashes) all over the network, companies have started trying to adopt more password-less authentication mechanisms to secure their infrastructure. From SSH bastion hosts to Kerberos and 2FA, there are many controls that attempt to limit attacker mobility in the event that a single account or password is compromised. This session will be a ""walking tour"" of bypass techniques that allow a small compromise to pivot widely and undetectably across a network using and abusing built in authentication features and common tools. Starting with a simple compromise of an unprivileged account (e.g. through phishing), this session will discuss techniques that pentesters and attackers use to gain footholds in networks and abuse trust relationships in shared computing resources and ""jumphosts"". The session will demo common tricks to elevate privileges, impersonate other users, steal additional credentials, and pivot around networks using SSH. The presentation will culminate with a discussion of 2FA for SSH access, and how compromises elsewhere in a network can be exploited to completely bypass it. Since these tricks and techniques utilize only built-in Linux commands, they are extremely difficult to detect as they look like normal usage. The demo environment will mimic a segmented network that uses Kerberos and two-factor authentication on SSH jump hosts. '
---

## Slides
{{< speakerdeck 1941f2cf309a46f9b92e88ec284ec37f >}}

## Supplemental
Demo Video:
{{< vimeo 165653886 >}}