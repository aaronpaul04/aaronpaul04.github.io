---
title: "Nibbles"
date: 2026-04-01
tags: ["linux", "easy", "hackthebox"]
description: "HackTheBox Nibbles writeup"
---

# Nibbles

## Overview

- **OS:** Linux (Ubuntu)
- **IP:** 10.129.13.242
- **Difficulty:** Easy
- **Platform:** HackTheBox

### Summary

Nibbles runs a Nibbleblog 4.0.3 instance with an arbitrary file upload vulnerability. After discovering the blog through source code inspection and enumerating admin credentials, a PHP reverse shell was uploaded via the "My image" plugin to get a foothold as nibbler. Privilege escalation was achieved by abusing a sudo NOPASSWD entry on a writable shell script to get root.

## Enumeration

### Nmap Scan

Started with a service and version scan.

![](nmap-scan.jpg)

Two ports open: SSH (OpenSSH 7.2p2) on port 22 and Apache httpd 2.4.18 on port 80.

### Web Enumeration

Browsing to the web server shows a simple "Hello world!" page.

![](hello-world.jpg)

Viewing the page source reveals a hidden HTML comment pointing to `/nibbleblog/`.

![](page-source.jpg)

Navigating to `/nibbleblog/` shows a blog powered by Nibbleblog with the tagline "Yum yum".

![](nibbleblog-homepage.jpg)

### Directory Bruteforcing

Ran Gobuster against the `/nibbleblog` directory to find more content.

![](gobuster.jpg)

Found several interesting paths including `/admin`, `/admin.php`, `/content`, `/plugins`, `/README`, and more.

### Version Identification

The README file confirmed the version: **Nibbleblog v4.0.3** (Codename: Coffee).

![](readme-version.jpg)

### Directory Listing

The `/content/` directory had directory listing enabled, exposing `private/`, `public/`, and `tmp/` folders.

![](content-dir.jpg)

The `/admin/` directory also had listing enabled showing the application structure.

![](admin-dir.jpg)

### Information Disclosure

Browsing `/content/private/` exposed several XML configuration files.

![](private-dir.jpg)

The `users.xml` file revealed the admin username and showed blacklisted IPs from failed login attempts.

![](users-xml.jpg)

The `config.xml` file leaked site configuration details including email addresses and paths.

![](config-xml.jpg)

The `/content/public/` directory showed the upload structure.

![](public-dir.jpg)

### Admin Login

Found the admin login page at `/nibbleblog/admin.php`.

![](admin-login.jpg)

The username `admin` was confirmed from `users.xml`. After some guessing, the password turned out to be `nibbles` - got into the dashboard.

![](admin-dashboard.jpg)

## Vulnerability Research

Searched for known exploits for Nibbleblog 4.0.3.

![](searchsploit.jpg)

Found an arbitrary file upload vulnerability - Nibbleblog 4.0.3 keeps the original extension of uploaded files via the "My image" plugin without checking the file type, allowing PHP file uploads and code execution. Admin credentials are required.

![](packetstorm-vuln.jpg)

## Exploitation

### Metasploit Attempt (Failed)

First tried the Metasploit module.

![](msf-search.jpg)

![](msf-options.jpg)

Configured the module with the target, credentials, and path.

![](msf-username.jpg)

![](msf-password.jpg)

![](msf-targeturi.jpg)

The Metasploit module gave a Meterpreter session.

![](meterpreter-pwd.jpg)

Landing directory was `/var/www/html/nibbleblog/content/private/plugins/my_image`.

### Manual Exploitation (Also Performed)

Also explored the manual route. In the Nibbleblog admin panel, went to Plugins and found the "My image" plugin.

![](plugins-page.jpg)

Uploaded a PHP reverse shell via the "My image" plugin configuration page. The warnings about image processing can be ignored - the PHP file still gets uploaded.

![](my-image-plugin.jpg)

Confirmed the uploaded file at `/nibbleblog/content/private/plugins/my_image/image.php`.

![](my-image-uploaded.jpg)

Set up a netcat listener and triggered the reverse shell by visiting the uploaded file.

![](reverse-shell.jpg)

Got a shell as `nibbler`. Upgraded to an interactive TTY with Python.

![](pty-upgrade.jpg)

## User Flag

![](user-flag.jpg)

## Privilege Escalation

### Sudo Enumeration

Checked sudo permissions for the nibbler user.

![](sudo-l.jpg)

![](sudo-l-shell.jpg)

The user `nibbler` can run `/home/nibbler/personal/stuff/monitor.sh` as root with NOPASSWD. The script doesn't exist yet but the path is within nibbler's home directory.

### Examining the Script

Found `personal.zip` in nibbler's home directory. Unzipped it to get the `monitor.sh` script.

![](unzip-monitor.jpg)

![](monitor-script.jpg)

The script is a server monitoring tool (Tecmint_monitor.sh) but since we have write access to it, we can replace its contents.

### Exploiting the Script

Overwrote `monitor.sh` with a payload that copies `/bin/bash` to `/tmp/stef` and sets the SUID bit.

![](privesc-setup.jpg)

Ran the script with sudo.

![](sudo-monitor.jpg)

Executed the SUID bash binary to get a root shell.

![](root-shell.jpg)

## Root Flag

![](root-flag.jpg)

pwned

## Lessons Learned

- Always check page source for hidden comments - the `/nibbleblog/` directory was only discoverable through an HTML comment.
- Directory listing on web servers exposes sensitive files like XML configs containing usernames and site configuration.
- Nibbleblog 4.0.3 has no file type validation on plugin uploads, making it trivial to upload PHP shells once you have admin access.
- Sudo NOPASSWD entries on writable scripts are a classic and easy privilege escalation vector. If a user can modify a script that runs as root, it's game over.
- Both Metasploit and manual exploitation paths were explored - having multiple approaches is useful when one method doesn't work cleanly.
