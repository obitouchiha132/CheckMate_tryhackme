# CheckMate_tryhackme
TryHackMe Operation Checkmate writeup documenting password attacks, targeted wordlist generation, OSINT-based password profiling, SHA-256 hash cracking, and SSH access using Hydra, CeWL, CUPP, and Hashcat.

# Operation Checkmate — TryHackMe Writeup

> **Room:** Operation Checkmate  
> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Password Attacks / Web Security / Credential Attacks  
> **Status:** Completed

---

## Overview

**Operation Checkmate** is a hands-on TryHackMe challenge focused on identifying and exploiting weak password practices across multiple internal services.

The challenge simulates a security assessment of systems belonging to an employee named **Marco Bianchi**. Instead of relying on blind brute-force attacks, the room provides clues that can be used to build targeted password wordlists and understand the target's password-generation habits.

Throughout the assessment, I used techniques including:

- Service enumeration
- Default credential testing
- Online password attacks with **Hydra**
- Targeted wordlist generation with **CeWL**
- Password profiling with **CUPP**
- SHA-256 hash cracking with **Hashcat**
- SSH authentication
- OSINT-based password pattern analysis

> **Note:** This writeup intentionally does **not** reveal discovered passwords or the original filename from Level 4. The goal is to document the methodology while still requiring readers to perform the challenge themselves.

---

## Lab Information

| Information | Details |
|---|---|
| Platform | TryHackMe |
| Room | Operation Checkmate |
| Target | Marco Bianchi's internal services |
| Main Application | Port `5000` |
| FirewallOS | Port `5001` |
| Employee Portal | Port `5002` |
| Social Platform | Port `5003` |
| Final Access | SSH |
| Username | `marco` |

---

# Methodology

The overall attack chain was:

```text
                    ┌─────────────────────┐
                    │   Target Machine    │
                    │   10.49.x.x         │
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
        Port 5001                         Port 5002
        FirewallOS                    Employee Portal
              │                                 │
      Default Credentials                Company Keywords
              │                                 │
            Hydra                              │
              │                              CeWL
              │                                 │
              └──────────────┬──────────────────┘
                             │
                             ▼
                     Marco's Information
                             │
                             ▼
                       Port 5003
                     Social Platform
                             │
                    ┌────────┴────────┐
                    │                 │
                  CUPP          Password Pattern
                    │                 │
                    └────────┬────────┘
                             │
                             ▼
                    Targeted Credentials
                             │
                             ▼
                       SSH as marco
                             │
                             ▼
                       System Access
```

# Level 1 — Default Credentials

## Objective
The first clue revealed that Marco had deployed a firewall management interface on:
```
firewall.thm:5001
```
The clue also indicated that the firewall was using default credentials.
![level 1](screenshot/level1.png)

## Step 1 — Access the Firewall
I accessed the service on port 5001:
```
http://10.49.177.247:5001
```
The application presented a FirewallOS login page.

![level 1](screenshot/5001_login.png)

The username was identified as:
```
admin
```
## Step 2 — Password Attack with Hydra
Since the room specifically indicated that default credentials were being used, I performed a targeted online password attack against the login form.
```
hydra -l admin \
-P /usr/share/wordlists/rockyou.txt \
-s 5001 \
TARGET_IP \
http-post-form \
"/login:username=^USER^&password=^PASS^:F=Invalid credentials"
```
Explanation
- -l admin → specifies the username
- -P → uses the RockYou password list
- -s 5001 → targets port 5001
- http-post-form → attacks an HTTP POST login form
- F=Invalid credentials → tells Hydra what indicates a failed login
Hydra identified a valid credential.

![level 1](screenshot/5001_bruteforce.png)
 
## Step 3 — Firewall Dashboard
After authentication, I reached the FirewallOS dashboard.
The dashboard exposed information such as:
- Firewall appliance name
- Management port
- WAN status
- Threat blocks
- IPS status
- Firewall policies
- System configuration information
  
 ![level 1](screenshot/5001_bypass.png)
 
# Finding

>The service was vulnerable because the administrator account was protected using a `default/weak credential`.
