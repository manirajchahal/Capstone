# Setup Guide — OpenClaw CoT Exploitation Agent

This guide covers installing and configuring OpenClaw on Kali Linux, loading the CoT skill file, and verifying everything works before your first run.

Tested on: Kali Linux 2024.1 (rolling) running as a VM under Proxmox.

---

## Prerequisites

Make sure the following are installed and working before starting.

**System packages:**

```bash
sudo apt update
sudo apt install -y nmap nikto metasploit-framework hydra gobuster python3 python3-pip
```

**Verify key tools:**

```bash
nmap --version
msfconsole --version
nikto -Version
hydra -h | head -5
gobuster version
```

If `msfconsole` is missing, install it via:

```bash
sudo apt install metasploit-framework
```

Kali includes Metasploit by default in most installations, but a fresh minimal install may not have it.

---

## Step 1 — Get an Anthropic API Key

The agent uses Claude Sonnet as its reasoning engine. You need an Anthropic API key with access to that model.

1. Go to [https://console.anthropic.com](https://console.anthropic.com)
2. Create an account or log in
3. Navigate to **API Keys** and generate a new key
4. Copy the key — you will need it in Step 3

---

## Step 2 — Install OpenClaw

Install OpenClaw via pip:

```bash
pip3 install openclaw
```

Verify the installation:

```bash
openclaw --version
```

If the command is not found after installation, your pip bin directory may not be in your `PATH`. Fix it:

```bash
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## Step 3 — Set Your API Key

OpenClaw passes your Anthropic API key to the Claude API for each reasoning step. Set it as an environment variable:

```bash
export ANTHROPIC_API_KEY=your_key_here
```

To persist it across sessions, add it to your `.bashrc` or `.zshrc`:

```bash
echo 'export ANTHROPIC_API_KEY=your_key_here' >> ~/.bashrc
source ~/.bashrc
```

**Do not hardcode your key into any config file that will be committed to git.** The `.gitignore` in this repo already excludes common credential file patterns, but environment variables are the safest approach.

---

## Step 4 — Start the OpenClaw Gateway

The OpenClaw gateway is the local server that the agent interface communicates through. Start it:

```bash
openclaw serve --port 18789
```

You should see output like:

```
[OpenClaw] Gateway started on http://localhost:18789
[OpenClaw] Waiting for connections...
```

Leave this running in a terminal. Open a new terminal for the next steps.

**Note:** If port 18789 is in use, pick any available port and update your prompts accordingly.

---

## Step 5 — Clone This Repo

```bash
git clone https://github.com/yourusername/openclaw-cot-agent.git
cd openclaw-cot-agent
```

---

## Step 6 — Load the CoT Skill File

Skills in OpenClaw are markdown files that are injected as system-level context into the LLM. Load the pentest skill:

```bash
openclaw skill load skills/pentest-cot.md
```

Confirm it loaded:

```bash
openclaw skill list
```

You should see `pentest-cot` in the output.

---

## Step 7 — Verify the Setup

Run a quick connectivity check to make sure the gateway is reachable and the API key is valid:

```bash
openclaw ping
```

Expected output:

```
[OpenClaw] Gateway: OK (localhost:18789)
[OpenClaw] LLM: OK (claude-sonnet-4-5)
[OpenClaw] Skills loaded: pentest-cot
```

If the LLM check fails, confirm your `ANTHROPIC_API_KEY` is set correctly in the current shell.

---

## Step 8 — Confirm Target Reachability

Before running the agent, confirm your target VM is up:

```bash
ping -c 3 192.168.8.230
```

If you're using a different target IP, update your prompts accordingly. Metasploitable 2 should respond to ping by default.

---

## Tailscale Setup (Optional — for Team Access)

If you need teammates to access the OpenClaw gateway remotely, you can expose it through a Tailscale overlay network instead of opening firewall rules.

```bash
# Install Tailscale on Kali
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Get your Tailscale IP
tailscale ip -4
```

Share your Tailscale IP with teammates. They can reach the OpenClaw gateway at `http://<tailscale-ip>:18789` once they've joined your Tailscale network (via your admin console at [https://login.tailscale.com](https://login.tailscale.com)).

---

## Troubleshooting

**Gateway won't start:**
- Check if the port is already in use: `ss -tlnp | grep 18789`
- Kill the conflicting process or change the port

**`openclaw skill load` fails:**
- Make sure you're running the command from the repo root (where `skills/` is)
- Check that the skill file is valid markdown with no syntax errors

**LLM returns errors mid-chain:**
- Usually a rate limit or API key issue. Check your Anthropic console for quota.
- Add a small delay between steps: `openclaw serve --step-delay 5`

**Metasploit module not found:**
- Run `msfdb init` to initialize the Metasploit database, then retry
- Run `msfupdate` to pull the latest modules
