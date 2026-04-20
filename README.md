# OpenClaw CoT Exploitation Agent

**Capstone Project — Cybersecurity Program**

A proof-of-concept penetration testing agent that extends the [OpenClaw](https://github.com/openclaw) AI agent framework with a Chain-of-Thought (CoT) reasoning loop. The agent plans, executes, and chains offensive security tools logically against an authorized target — one step at a time — using Claude Sonnet as the reasoning engine.

> **Disclaimer:** This project was developed and tested exclusively in an isolated lab environment against Metasploitable 2, a deliberately vulnerable VM. It is intended for educational purposes only. Do not use against systems you do not own or have explicit written permission to test.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Lab Environment](#lab-environment)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [How to Run](#how-to-run)
- [Results Summary](#results-summary)
- [Project Structure](#project-structure)
- [Limitations](#limitations)

---

## Project Overview

Traditional penetration testing automation tools execute pre-scripted workflows. This project explores a different approach: giving an LLM the ability to reason about tool output and decide what to do next, iteratively, without hardcoded logic trees.

The agent uses a **Chain-of-Thought skill file** (`skills/pentest-cot.md`) that instructs Claude Sonnet to:

1. Reason out loud about what each tool result means
2. Decide what the next logical step is
3. Execute that step via OpenClaw's tool interface
4. Evaluate whether to continue or stop

This loop runs up to 5 chained steps and halts automatically when no new attack surface is found or when the objective (root access) is achieved.

The end-to-end test on Metasploitable 2 resulted in a root shell obtained in 4 steps, with no human input after the initial trigger.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Operator (You)                      │
│              issues initial target prompt               │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                 OpenClaw Gateway                        │
│              localhost:18789 (Kali VM)                  │
│                                                         │
│  ┌─────────────────┐    ┌──────────────────────────┐   │
│  │  pentest-cot.md │───▶│  Claude Sonnet (API)     │   │
│  │  (Skill File)   │    │  Reasoning Engine        │   │
│  └─────────────────┘    └────────────┬─────────────┘   │
│                                      │                  │
│                         ┌────────────▼─────────────┐   │
│                         │  OpenClaw Exec Tool       │   │
│                         │  (shell command runner)   │   │
│                         └────────────┬─────────────┘   │
└──────────────────────────────────────┼─────────────────┘
                                       │
               ┌───────────────────────┼──────────────────┐
               ▼                       ▼                   ▼
          ┌─────────┐           ┌──────────┐         ┌──────────┐
          │  Nmap   │           │ Metasploit│         │  Hydra   │
          └─────────┘           └──────────┘         └──────────┘
               ▼                       ▼                   ▼
          ┌─────────┐           ┌──────────┐         ┌──────────┐
          │  Nikto  │           │ Gobuster  │         │  (more)  │
          └─────────┘           └──────────┘         └──────────┘
                                       │
                                       ▼
                            ┌──────────────────┐
                            │  Metasploitable 2 │
                            │  192.168.8.230    │
                            └──────────────────┘
```

For a detailed design breakdown, see [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

---

## Lab Environment

| Component | Details |
|---|---|
| Attacker OS | Kali Linux (VM on Proxmox) |
| Hypervisor | Proxmox VE |
| Target | Metasploitable 2 — 192.168.8.230 |
| OpenClaw Gateway | localhost:18789 |
| LLM | Claude Sonnet (Anthropic API) |
| Network | Isolated lab + Tailscale overlay for team access |

---

## Prerequisites

- Kali Linux (bare metal or VM)
- Proxmox (if running VMs) — optional, any hypervisor works
- Python 3.10+
- An Anthropic API key with access to Claude Sonnet
- OpenClaw installed (see [SETUP.md](SETUP.md))
- Tools installed on Kali: `nmap`, `nikto`, `metasploit-framework`, `hydra`, `gobuster`

---

## Setup

See [SETUP.md](SETUP.md) for the full step-by-step installation guide.

Short version:

```bash
# Install OpenClaw
pip install openclaw

# Configure your API key
export ANTHROPIC_API_KEY=your_key_here

# Start the OpenClaw gateway
openclaw serve --port 18789

# Load the CoT skill
openclaw skill load skills/pentest-cot.md
```

---

## How to Run

With the OpenClaw gateway running and the skill loaded, send the agent its initial prompt. You can do this via the OpenClaw CLI or the web UI at `http://localhost:18789`.

**Example initial prompt:**

```
Target: 192.168.8.230
You are authorized to perform a full penetration test on this host.
Begin with reconnaissance and proceed logically using the pentest-cot skill.
```

The agent will then take over, running up to 5 chained steps autonomously. Watch the OpenClaw console for step-by-step reasoning and tool output.

**Stopping early:**

Press `Ctrl+C` in the OpenClaw console at any time, or set `MAX_STEPS=2` in your environment to limit the chain depth for testing.

---

## Results Summary

In the end-to-end test against Metasploitable 2, the agent completed 4 steps and achieved a root shell via a reverse Netcat connection.

| Step | Action | Outcome |
|---|---|---|
| 1 | Nmap scan (192.168.8.230) | Identified 23 open ports; flagged vsftpd 2.3.4, Samba 3.x, UnrealIRCd |
| 2 | Metasploit — vsftpd 2.3.4 backdoor | Shell obtained as `root` |
| 3 | Metasploit — Samba usermap_script | Shell obtained as `root` |
| 4 | Post-exploitation — whoami + id | Confirmed `uid=0(root)` |

**Agent stopped automatically** after step 4 — root access achieved, objective met.

Full details and agent reasoning transcripts are in [WRITEUP.md](WRITEUP.md).

---

## Project Structure

```
openclaw-cot-agent/
├── README.md               # This file
├── SETUP.md                # OpenClaw installation guide
├── WRITEUP.md              # End-to-end test writeup
├── .gitignore              # Excludes keys, tokens, raw results
├── skills/
│   └── pentest-cot.md      # The CoT skill file loaded into OpenClaw
├── docs/
│   └── ARCHITECTURE.md     # System design and component breakdown
└── results/
    └── sample-run.md       # Sanitized example of agent output
```

---

## Limitations

This is a proof-of-concept, not a production tool.

- The agent has no memory between sessions; each run starts from scratch.
- The 5-step cap is hardcoded in the skill file. More complex targets would require a longer chain or recursive planning.
- Tool selection is LLM-driven, which means it can make suboptimal choices if tool output is ambiguous.
- Tested only against Metasploitable 2. Results on other targets will vary.
- The Anthropic API introduces latency between steps (typically 3–8 seconds per reasoning cycle).
