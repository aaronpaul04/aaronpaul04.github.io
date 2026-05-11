---
title: "Legacy"
date: 2026-03-16
tags: ["windows", "easy", "hackthebox"]
description: "HackTheBox Legacy writeup"
---

# Legacy

## Overview

- **OS:** Windows XP (Service Pack 3)
- **IP:** 10.129.227.181
- **Difficulty:** Easy
- **Platform:** HackTheBox

### Summary

Windows XP box with only SMB ports open. Exploited via MS08-067 for immediate SYSTEM shell.

## Enumeration

Started with an nmap service and version scan.

![](nmap-scan.png)

Only SMB-related ports open (135, 139, 445). OS identified as Windows XP with SMB2 negotiation failing, confirming a legacy system. The narrow attack surface makes MS08-067 the obvious candidate.

## Vulnerabilities

**PORT 445/tcp**

Windows XP microsoft-ds

**MS08-067 (CVE-2008-4250)** - critical vulnerability in the Windows Server service (netapi32.dll) that allows remote code execution through a specially crafted RPC request. Affects Windows XP, 2000, and Server 2003.

## Exploitation

Used the ms08_067_netapi Metasploit module. Set LHOST to VPN tunnel address and RHOSTS to target.

![](ms08-067-options.png)

Metasploit fingerprinted the target as Windows XP SP3 English (AlwaysOn NX) and delivered a Meterpreter payload.

![](meterpreter-session.png)

SYSTEM access confirmed via getuid. No privilege escalation needed.

## Post-Exploitation

Meterpreter cd command had issues with "Documents and Settings" path. Dropped into native Windows shell to navigate.

![](filesystem-listing.png)

![](cmd-navigation.png)

## Flags

### User Flag

![](user-flag.png)

### Root Flag

![](admin-desktop.png)

![](root-flag.png)

pwned
