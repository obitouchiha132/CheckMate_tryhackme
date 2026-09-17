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
 
> **Finding:** The service was vulnerable because the administrator account was protected using a `default/weak credential`.

# Level 2 — Targeted Wordlist with CeWL
## Objective
The next clue pointed to:
```
jobs.thm:5002
```
It stated that Marco had created an internal employee login panel and used common company keywords as passwords.

![level 1](screenshot/level2.png)

## Step 1 — Enumerating the Website
I accessed:
```
http://10.49.177.247:5002
```
The website was an Engineering Careers portal.

![level 1](screenshot/5002_main.png)
 
The website contained several useful keywords, including:
- innovation
- excellence
- security
- digital
- cloud
- future
- talent

These words were important because the challenge specifically mentioned company keywords.

## Step 2 — Employee Login

The website contained an Employee Login page.

![level 1](screenshot/5002_login.png)
 
The username was:`marco`

Instead of blindly using a huge password list, I created a targeted wordlist from the website itself.

## Step 3 — Generate Wordlist with CeWL
I used CeWL to crawl the website and extract words.
```
cewl -d 2 -m 3 --lowercase --with-numbers \
-w 5002_words.txt \
http://10.49.177.247:5002
```
Options Used

| Option | Purpose
|---|---|
| -d 2	             | Crawl up to depth 2
| -m 3	             | Minimum word length of 3
| --lowercase	       | Convert extracted words to lowercase
| --with-numbers	   | Include words containing numbers
| -w	               | Save output to a wordlist

This produced:
```
5002_words.txt
```

## Step 4 — Attack the Login

I then used Hydra with the generated wordlist:
```
hydra -l marco \
-P 5002_words.txt \
-s 5002 \
TARGET_IP \
http-post-form \
"/login:username=^USER^&password=^PASS^:F=Invalid credentials"
```
Hydra successfully identified a valid credential.

![level 1](screenshot/5002_bruteforce.png)
 
## Step 5 — Employee Profile
After successful authentication, I accessed Marco's employee profile.

![level 1](screenshot/5002_bypass.png) 

The profile exposed additional information about Marco.
This information became useful in the next stage of the attack.

# Level 3 — Password Profiling with CUPP

## Objective
The Level 3 clue directed me to:
social.thm:5003

![level 1](screenshot/level3.png)

The login page contained a hint indicating that information from the employee portal could be used to generate Marco's password.

![level 1](screenshot/5003_login.png)

## Step 1 — Gather Information

From the employee profile, I collected publicly available information such as:
```
First Name: Marco
Surname: Bianchi
Nickname: marky
Birthdate: 14021995
```
I also found Marco's social media profile.
 
His public information provided useful clues about his password habits.

## Step 2 — Generate a Targeted Wordlist with CUPP
Instead of manually creating hundreds of combinations, I used CUPP — Common User Passwords Profiler.
```
python3 cupp.py -i
```
I entered the information discovered during enumeration:

- First Name: Marco
- Surname: Bianchi
- Nickname: marky
- Birthdate: 14021995

The remaining fields were left blank because they were not required.
CUPP generated:
`marco.txt`
containing combinations based on the supplied information.

![level 1](screenshot/cupptool.png)

## Step 3 — Hydra Against the Social Login

I used the generated wordlist against the login form:
```
hydra -l marco \
-P marco.txt \
-s 5003 \
-f \
-t 4 \
social.thm \
http-post-form \
"/login:username=^USER^&password=^PASS^:F=invalid"
```
Important Options
- -l marco → target username
- -P marco.txt → CUPP-generated wordlist
- -s 5003 → target port
- -f → stop after finding a valid credential
- -t 4 → use four parallel tasks

Hydra successfully found a valid credential.

After authentication, I gained access to Marco's social media account.

# Level 4 — SHA-256 Filename Cracking
## Objective
The Level 4 clue revealed that Marco's uploaded profile picture had been renamed using the SHA-256 hash of the original filename.

![level 1](screenshot/level4.png)

The resulting file was stored in the following format:
```
SHA256_HASH.png
```
The task was to recover the original filename.

## Step 1 — Identify the SHA-256 Hash
I opened Marco's profile picture in a new browser tab.

The image URL contained a long SHA-256 hash:
```
d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b
```
![level 1](screenshot/sha256_hash.png)

I extracted the hash and stored it in a file:
**hashh.txt**

## Step 2 — Crack the Hash with Hashcat

I identified the hash as SHA-256 and used Hashcat mode 1400.
```
hashcat -m 1400 \
-a 0 \
hashh.txt \
/usr/share/wordlists/rockyou.txt \
--session=sha256
```
| Hashcat | Options
|--|--|
| Option	| Meaning
| -m 1400	| SHA2-256 hash mode
|-a 0	| Straight/dictionary attack
| hashh.txt	| File containing the target hash
| rockyou.txt	| Dictionary
| --session=sha256	| Names the Hashcat session


Hashcat reported:
Status: Cracked
Hash.Mode: 1400 (SHA2-256)
 
The original filename was successfully recovered.
 
![level 1](screenshot/cracking_hash.png)

# Level 5 — Password Pattern Analysis & SSH
## Objective
The final clue revealed that Marco had publicly disclosed his password-generation pattern.

The social media post explained that the password was based on: `tip`
```
Company keyword
        +
First letter capitalized
        +
2024
        +
!
```
This was the key to the final stage.

 ![level 1](screenshot/tip.png)

## Step 1 — Analyze the Password Pattern

The relevant company keywords discovered earlier included:
```
security
excellence
innovation
digital
cloud
```
The pattern was:
<Keyword with first letter capitalized>2024!

Instead of using a large generic wordlist, I generated a small targeted wordlist based directly on the information disclosed by Marco.

Example structure:
```
Security2024!
Excellence2024!
Innovation2024!
Digital2024!
Cloud2024!
```
## Step 2 — SSH Access
I connected to the machine:
```
ssh marco@10.49.177.247
```
Authentication was successful.
 
I then verified the current user:
```
whoami
Output:marco
```
I also checked the user's identity and group membership:
```
id
Output confirmed:uid=1001(marco) gid=1001(marco) groups=1001(marco),100(users)
```

This confirmed successful remote access as the `marco` user.

 ![level 1](screenshot/ssh.png)

# Attack Chain Summary
The complete attack chain can be summarized as:
```
                    Reconnaissance
                         │
                         ▼
              ┌────────────────────┐
              │   Port 5001        │
              │   FirewallOS       │
              └─────────┬──────────┘
                        │
                 Default Credentials
                        │
                      Hydra
                        │
                        ▼
              ┌────────────────────┐
              │   Port 5002        │
              │ Employee Portal    │
              └─────────┬──────────┘
                        │
                 Company Keywords
                        │
                      CeWL
                        │
                      Hydra
                        │
                        ▼
              ┌────────────────────┐
              │   Port 5003        │
              │ Social Platform    │
              └─────────┬──────────┘
                        │
                 Personal Details
                        │
                       CUPP
                        │
                      Hydra
                        │
                        ▼
               Social Media Access
                        │
                        ▼
              SHA-256 Filename Hash
                        │
                     Hashcat
                        │
                        ▼
                Filename Recovered
                        │
                        ▼
               Password Pattern
                  discovered
                        │
              Targeted Wordlist
                        │
                      Hydra
                        │
                        ▼
                  SSH as marco
                        │
                        ▼
                 System Access
```
# Tools Used

| Tool | Purpose |
|---|---|
| **Nmap / Enumeration** | Service discovery and reconnaissance |
| **Hydra** | Online password attacks |
| **RockYou** | Password dictionary |
| **CeWL** | Website-based targeted wordlist generation |
| **CUPP** | Personal-information-based password profiling |
| **Hashcat** | SHA-256 offline password/hash cracking |
| **SSH** | Remote system access |
| **Bash** | Targeted wordlist generation |

Key Takeaways

## 1. Default Credentials Are Dangerous
The first stage demonstrated how dangerous default administrative credentials can be.
A service exposed to a network should never rely on vendor or deployment defaults.

## 2. Targeted Wordlists Can Be More Effective
The Level 2 attack did not require blindly trying enormous password lists.
Instead, information from the target website was extracted using:
```cewl```
This produced a smaller and more relevant wordlist.

## 3. Public Information Can Help Build Password Dictionaries
CUPP demonstrated how information such as:
```
Name
Surname
Nickname
Birthdate
```
can be transformed into password candidates.
This is why personal information should not be used directly in passwords.

## 4. Password Patterns Reduce Password Entropy
The final stage was particularly interesting because Marco had publicly revealed a predictable password-generation pattern.
Even though the password appeared to contain:
- Uppercase letters
- Lowercase letters
- Numbers
- Special characters
the predictable structure dramatically reduced the search space.

# Security Recommendations
From a defensive perspective, the issues demonstrated in this lab could be mitigated by:
## Disable Default Credentials
Administrative interfaces should require changing default credentials during initial deployment.
## Use Strong, Random Passwords
Passwords should not be based on:
```
Names
Birthdays
Company keywords
Predictable years
Publicly available information
```
## Implement MFA
Multi-factor authentication can significantly reduce the impact of a compromised password.
## Rate Limit Authentication
Login endpoints should implement:
- Rate limiting
- Account lockout policies
- Progressive delays
- Monitoring and alerting
## Avoid Predictable Password Policies
A password such as:
```
Keyword + Year + !
```
may satisfy complexity requirements while remaining highly predictable.
## Secure Sensitive Services
Management interfaces and SSH should not be unnecessarily exposed and should be protected using network controls and strong authentication.

# Lessons Learned
This room helped me practice a complete password-attack workflow rather than relying on a single tool.

The most important lesson was that context matters.

Instead of simply running a massive brute-force attack, I followed the clues provided by the application and progressively built a better understanding of the target:
```
Service Discovery
      ↓
Credential Enumeration
      ↓
Targeted Wordlist
      ↓
Password Profiling
      ↓
Hash Cracking
      ↓
Pattern Analysis
      ↓
SSH Authentication
```
The challenge also demonstrated how multiple small pieces of information can be combined into a successful attack chain.

# Conclusion

## Operation Checkmate was a practical exercise in password security assessment.

The challenge covered several important penetration-testing techniques:
- Default credential testing
- Online password attacks
- Targeted wordlist generation
- Password profiling
- OSINT-based credential analysis
- SHA-256 hash cracking
- Password pattern analysis
- SSH authentication

The biggest takeaway is that a password does not necessarily become secure just because it contains uppercase letters, numbers, and special characters.
If the underlying pattern is predictable, the effective password search space can become much smaller.

> Completed: Operation Checkmate
