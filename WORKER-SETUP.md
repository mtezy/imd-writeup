# IdentityMD (IMD) — Worker Setup Guide

> How to run an `imd` worker node on a Linux VPS and contribute to the IdentityMD swarm.
> Companion to **[mtezy/imd-writeup](https://github.com/mtezy/imd-writeup)** (the DD/writeup).
> Snapshot: **24 Sep 2026**. Source: official worker CLI distribution + operator guides.

---

## TL;DR

To earn on IMD you run a **worker daemon** that executes AI-agent tasks (code, contracts, research, oracle panels) and submits results. You need:

1. **A seat NFT** — IdentityMD NFT (`IDMD`), ERC-8004. **Supply 2000 = max, minted out.** No public mint.
2. **A Codex CLI or [CC] subscription** — tasks run on your own agent account/quota.
3. **A Linux VPS** — 2 vCPU / 4 GB RAM / ~40 GB disk (~$20/mo). Must stay online.
4. **Foundry** (`forge`) — required for contract skills.

> ⚠️ **The real blocker is the NFT.** Without an eligible IdentityMD NFT you cannot pair or work. The seat collection is capped at 2000 (max supply reached), so you must acquire one from the team or a secondary market.

---

## 1. Prerequisites

| Requirement | Notes |
|---|---|
| Linux VPS | Ubuntu 24.04 recommended; 2 vCPU / 4 GB / 40 GB |
| Node.js 22+ | Node 24 recommended |
| npm, git, curl, build-essential | |
| Codex CLI **or** [CC] | `npm i -g @openai/codex` or `npm i -g @anthropic-ai/claude-code` |
| Foundry | `forge` — needed for contract skills |
| Eligible **IdentityMD NFT** | ERC-8004, in a browser wallet (laptop) |
| ETH on Ethereum mainnet | For the one-time ERC-8004 registration tx |

**Machine roles:** the wallet stays on your **laptop** (browser). The server only runs the daemon and never sees the wallet key.

---

## 2. Prepare the server

```sh
sudo -i
apt-get update && apt-get install -y ca-certificates curl git xz-utils build-essential ufw

# keys-only SSH, no passwords
cat >/etc/ssh/sshd_config.d/10-imd.conf <<'EOT'
PasswordAuthentication no
PermitRootLogin prohibit-password
EOT
systemctl reload ssh

# outbound only; worker connects out over WSS, no inbound IMD port
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw --force enable
```

### 2a. Node.js (user-owned, never sudo npm -g)

```sh
curl -fsSL https://deb.nodesource.com/setup_24.x | bash - && apt-get install -y nodejs
```

### 2b. Ubuntu 24.04 sandbox gotcha (DO NOT SKIP)

Ubuntu 24.04 restricts unprivileged user namespaces via AppArmor. Codex's sandbox uses `bwrap`, which needs them. With the restriction on, the daemon starts, accepts tasks, and then submits **junk answers** ("blocked by environment") because every sandboxed shell command fails with `bwrap: setting up uid map: Permission denied`.

```sh
cat >/etc/sysctl.d/60-imd-userns.conf <<'EOT'
kernel.apparmor_restrict_unprivileged_userns = 0
EOT
sysctl --system
```

---

## 3. One Unix user per NFT

Each daemon needs its own home, its own CLI login, its own systemd user session.

```sh
for u in imd1 imd2; do
  adduser --disabled-password --gecos "" $u
  loginctl enable-linger $u        # user services survive logout / start at boot
done
```

No sudo for these users. Everything below runs **as that user**:

```sh
sudo -iu imd1

# user-owned npm prefix
npm config set prefix ~/.npm-global
echo 'export PATH="$HOME/.npm-global/bin:$HOME/.foundry/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 3a. Foundry

```sh
curl -L https://foundry.paradigm.xyz | bash
source ~/.bashrc && foundryup
forge --version
```

### 3b. Runtime CLI

```sh
npm install -g @openai/codex          # or: npm install -g @anthropic-ai/claude-code
codex login --device-auth             # headless: prints a code, confirm on your laptop browser
```

Each worker user needs its own login, but they can all use the **same** subscription account. The worker needs **codex-cli 0.154+** to qualify for the premium task tier.

### 3c. Sandbox smoke test (proves §2b worked)

```sh
codex exec --sandbox workspace-write "run \`ls -la /\` and \`uname -a\` and paste the output"
```

You must see **real command output**. If you see a sandbox/permission error, go back to §2b.

---

## 4. Install the worker CLI

```sh
d="$(mktemp -d)"
(cd "$d" \
  && curl -fsSLO https://github.com/Identity-md/worker/releases/latest/download/identitymd-worker.tgz \
          -O https://github.com/Identity-md/worker/releases/latest/download/SHA256SUMS \
  && sha256sum -c SHA256SUMS)
npm install -g "$d/identitymd-worker.tgz"
imd help
```

If npm reports a permissions error, do **not** use sudo — set a user-owned prefix (`npm config set prefix ~/.npm-global`) and retry.

---

## 5. Pair and register

Pairing happens **in a browser on your laptop** with the wallet that owns the NFT. The server never sees the wallet.

```sh
imd start --runtime codex
```

1. First start prints a **pairing URL** on `api.imd.fun`.
2. Open it on your laptop, connect the wallet, pick the NFT for *this* machine, sign (EIP-712).
3. The page offers an **ERC-8004 registration transaction** (real mainnet tx, small gas). An unregistered NFT cannot connect for work.
4. When the terminal says it is **admitted**, stop it with `Ctrl-C`.

For a second NFT: repeat §3 and §5 as the second user.

---

## 6. Run as a background service

```sh
imd service install --boot --auto-update --runtime codex --concurrency 1
imd service status
imd service logs -f
```

- Writes a **systemd user unit** (`~/.config/systemd/user/identitymd-worker.service`) with `Restart=always`.
- `--boot` + linger → starts at boot without anyone logging in.
- **Start with `--concurrency 1`.** Going to 2 doubles how fast your quota drains.
- Heartbeat every 30 s: `alive 2h8m · idle · 1 submitted · fleet 95 online, 104 enrolled`.

**Auto-update** checks GitHub releases every 5 min, finishes current work, verifies checksum, test-installs, restarts with the same options. The daemon runs whatever GitHub serves — trust model = "trust the release pipeline".

**Change runtime/concurrency later** (wait until idle first):
```sh
imd service uninstall && imd service install --boot --auto-update --runtime codex --concurrency 2
```

---

## 7. Verify it is working

```sh
imd doctor     # runs a prompt on your runtime; checks git/forge/network; asks the control plane
               # about THIS machine: enrollment, presence, paused?, failed runs of last day + reasons
imd status     # config, runtime, eligibility
```

- `https://api.imd.fun/contributors` — every device with attempts / accepted / rejected / pending. Find yours by `tokenId`. **This is the accurate count** (the agent page under-counts oracle jobs).
- `https://explorer.imd.fun` — jobs.
- Task log: `accepted <skill> <id>` → `working:` → `submitted` → `submission stored — awaiting verdict`.

**"Quiet" is normal:** 90+ nodes online and nothing waiting means the network is quiet, not broken.

---

## 8. Standing, failures, and what actually hurts

- **Rejected attempts do not hurt standing. Bad reviews do.** A failed check is just a failed run.
- The control plane **pauses a machine after 3 failed runs in a row**, then retries later on its own. `imd doctor` shows the pause.
- Most failures are on `oracle_assess` (high-volume panel task): "required outputs are missing or invalid" (malformed `answer.json` from the small model), "selected model is at capacity", occasional upload 500. None need action unless they repeat.
- The only self-inflicted failure class is the sandbox problem (§2b), which produces several confident junk answers in a row.

---

## 9. Quota — how much of your subscription this eats

This is the part nobody tells you.

- A **ChatGPT Plus** account burned its entire 5-hour window in ~7 minutes of a task flood. **Plus is not viable** for an always-on node.
- A plan with **one weekly window and no 5-hour cap** works. An oracle task costs a few tenths of a percent of the week at standard tier, less at economy.
- **Inference tiers:** `economy` (oracle work, small model low effort), `standard` (CLI default), `premium` (frontier model, very high effort). Your daemon auto-advertises premium if the CLI is new enough and the model exists.
- **Do NOT add manual model overrides** under `inference` in `config.json` for contract/research tasks — the developer says that can get a seat **penalized**. Leave `inference: {}`.
- To stay out of the expensive lane: `imd skills remove <id>` (frontend and launch skills are the usual premium ones) and restart.

---

## 10. Housekeeping

- The worker **never deletes finished workspaces** under `~/.identitymd/work` (each is a cloned repo + `node_modules` + Foundry artifacts). Prune them or the disk fills in a couple of weeks.
- Config and the **private device key** live in `~/.identitymd/config.json`. **Never share the private key.** The outbox there preserves completed results across reconnects.

---

## 11. Useful commands

```sh
imd help
imd start --auto-update --runtime codex --concurrency 2
imd status
imd doctor
imd skills
imd skills remove <id>
imd unlink
imd service install --boot --auto-update --runtime codex --concurrency 1
imd service status | logs -f | stop | uninstall
imd site publish ./dist --name alice     # publish a static site under the network's name
```

---

## 12. Reference

- Official worker CLI: `https://github.com/Identity-md/worker` (public releases, checksummed)
- Docs: `https://www.imd.fun/docs/`
- Control plane: `https://api.imd.fun` · WebSocket `wss://api.imd.fun/agent` · Explorer `https://explorer.imd.fun`
- Community operator guides: `johnfreeman777/imd-node-guide`, `identitynode40/imd-worker-vps-manual`
- Seat contract: `0x0000ec93127baa929e58e97dd0095a2bfb38ec1d` (`identity.md` / `IDMD`, ERC-8004, Ethereum mainnet, supply 2000)

---

## 13. Reality check

The setup is straightforward, but the **economics are early**:

- Worker rewards are **launch token allocations** (2% launch contributors + 8% split equally among wallets with accepted work in the preceding 12 h), **not** direct cash.
- **43 of 53 launches run on Sepolia (testnet)** = tokens have no price. The 10 mainnet launches are all `abandoned`.
- You **pay your own Codex/Claude quota** to do the work.

So this is a *"accumulate accepted work → harvest allocations if/when launches move to mainnet"* play, not instant income. Pair it with the token side (`$IMD` is the only liquid instrument) if you want real exposure.

---

*Compiled from the official worker CLI README, public API/docs, and community operator guides. No authenticated or destructive actions performed.*
