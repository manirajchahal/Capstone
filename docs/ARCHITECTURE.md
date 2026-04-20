# System Architecture — OpenClaw CoT Exploitation Agent

This document explains how the system is structured, how data flows between components, and the design decisions behind this implementation.

---

## Overview

The system is composed of four layers:

1. **Skill Layer** — the CoT prompt file that defines agent behavior
2. **Reasoning Layer** — Claude Sonnet, called via the Anthropic API
3. **Execution Layer** — OpenClaw's exec tool, which runs shell commands on the Kali host
4. **Tool Layer** — the actual offensive security tools (Nmap, Metasploit, etc.)

These layers communicate in a single direction: the skill instructs the LLM, the LLM calls the exec tool, the exec tool runs tools, and the output feeds back to the LLM for the next reasoning step.

---

## Component Breakdown

### OpenClaw Gateway

OpenClaw is an open-source AI agent framework that provides:

- A local HTTP gateway (in this project: `localhost:18789`) that acts as the interface between the operator and the LLM
- A skill loading system for injecting markdown files as system prompts
- An exec tool that allows the LLM to run arbitrary shell commands on the host machine

In this project, OpenClaw is running directly on the Kali Linux VM. This gives it native access to all installed tools (Nmap, Metasploit, Hydra, etc.) without any additional sandboxing or API abstraction.

### Claude Sonnet (Reasoning Engine)

Each reasoning step is a call to the Anthropic API using Claude Sonnet. The model receives:

- The full CoT skill file as a system prompt
- The accumulated conversation history from all prior steps (tool outputs, agent reasoning)
- A new user message containing the most recent tool output

Claude then returns a structured reasoning block following the format defined in the skill file, which includes the next action and the command to run.

This is a **synchronous, stateless** setup. Claude has no memory beyond what is in the current context window. The conversation history is maintained by OpenClaw and passed with each API call.

### pentest-cot.md (Skill File)

The skill file is the core contribution of this project. It defines:

- **Role**: the agent is a penetration tester on an authorized host
- **Rules**: step ordering, stop conditions, justification requirements
- **Output format**: a structured 4-part response for every step
- **Tool reference**: a table of available tools and example commands

The skill file does not contain any code. It is pure natural language instruction. The LLM interprets it and applies it during inference.

The format enforcement (requiring the agent to state step number, reasoning, and a stop-or-continue decision every time) is what makes the chain reliable. Without structure, the LLM would tend to run multiple steps in one response or skip justification.

### OpenClaw Exec Tool

OpenClaw's exec tool allows the LLM to run shell commands on the host machine. When the LLM decides on a next action, it emits a structured tool call that OpenClaw intercepts and executes. The output is captured and injected back into the conversation as the next user message.

This is the only tool the agent uses. All offensive operations (Nmap scans, Metasploit exploits, web scans) are run as shell commands through this single interface.

### Target — Metasploitable 2

Metasploitable 2 is a deliberately vulnerable Linux VM designed for practicing penetration testing. It runs at `192.168.8.230` on the isolated lab network. It is the only authorized target in this lab.

---

## Data Flow

```
Operator
  │
  │  Initial prompt (target IP + objective)
  ▼
OpenClaw Gateway (localhost:18789)
  │
  │  System prompt = pentest-cot.md + conversation history
  ▼
Claude Sonnet API (Anthropic)
  │
  │  Returns: Reasoning block + tool call (shell command)
  ▼
OpenClaw Exec Tool
  │
  │  Runs shell command on Kali VM
  ▼
Tool (Nmap / Metasploit / Hydra / etc.)
  │
  │  Returns: stdout output
  ▼
OpenClaw Gateway
  │
  │  Appends output to conversation history
  ▼
Claude Sonnet API (next step)
  │
  │  (loop continues until stop condition)
  ▼
Final Summary Output → Operator
```

---

## Design Decisions

### Why OpenClaw?

OpenClaw was chosen because it provides the exec tool out of the box and already handles conversation history, API calls, and skill loading. Building this on top of OpenClaw meant the project could focus on the CoT prompt design and testing rather than scaffolding.

### Why Claude Sonnet and not a smaller model?

The CoT chain depends on the model's ability to read raw tool output (Nmap scans, Metasploit responses) and make correct inferences about what is exploitable. Smaller models struggled with this during early testing — they either failed to parse Nmap output correctly or made unjustified jumps between steps. Sonnet handled all test cases correctly.

### Why a markdown skill file instead of code?

The CoT logic lives entirely in the skill file rather than in Python or another language. This keeps the system simple and makes the agent's behavior easy to inspect and modify without touching any code. It also means the "chain" is entirely the LLM's reasoning — there is no hardcoded conditional logic deciding what happens next. The model decides, following the rules in the skill file.

This is intentional for a proof of concept. A production system would likely want some code-level guardrails (e.g., blocking certain commands, enforcing rate limits, logging all tool calls).

### Why Netcat for the reverse shell payload?

Metasploit's `cmd/unix/reverse_netcat` payload was chosen because it is simple, reliable on Metasploitable 2, and produces clean shell output that is easy for the LLM to parse. Meterpreter would have produced output that required additional parsing logic.

---

## What This Is Not

This is not a replacement for a real penetration tester. The agent has no ability to:

- Adapt to targets it has not seen patterns of before
- Handle interactive shell sessions beyond basic command-response
- Perform any post-exploitation beyond confirming user context
- Recognize or bypass modern defenses (EDR, WAF, etc.)
- Manage long engagements (it has a 5-step cap and no persistent memory)

It is a proof of concept demonstrating that LLM-driven tool chaining is viable in a controlled lab environment.

---

## Network Diagram

```
┌──────────────────────────────────────────────────────────┐
│  Proxmox Host                                            │
│                                                          │
│  ┌─────────────────────────┐  ┌───────────────────────┐ │
│  │  Kali Linux VM          │  │  Metasploitable 2 VM  │ │
│  │                         │  │                       │ │
│  │  OpenClaw :18789        │  │  192.168.8.230        │ │
│  │  Nmap, MSF, Hydra, etc. │  │  (target)             │ │
│  │                         │  │                       │ │
│  │  ──── Tailscale ────────┼──┼───────────────────    │ │
│  │  (team overlay network) │  │                       │ │
│  └─────────────────────────┘  └───────────────────────┘ │
│                                                          │
└──────────────────────────────────────────────────────────┘
                        │
              Anthropic API (HTTPS)
                        │
                 Claude Sonnet
```
