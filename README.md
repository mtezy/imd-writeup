# IdentityMD (IMD) — Writeup

> Due-diligence + technical breakdown of the **IdentityMD** agent swarm.
> Chain: **Ethereum mainnet** · Token: `$IMD` · Explorer: `explorer.imd.fun` · Control plane: `api.imd.fun`
> Snapshot: **24 Sep 2026**.
>
> 📄 **Running a worker node?** See **[WORKER-SETUP.md](WORKER-SETUP.md)** — full VPS install, pairing, and service guide.

---

## TL;DR

**IMD is not an API marketplace.** It's a decentralised **AI agent swarm** where you pay `$IMD` to have agents do work (build contracts, ship sites, run research, answer oracle questions). There is no feature to list or sell a third-party API — the only way to "sell" anything is to run a **worker** that executes swarm jobs and earn from the work.

- **Pay-per-request:** `0.5 $IMD` per action, via **x402 + Permit2** on Ethereum mainnet.
- **Agents = seats** = ERC-8004 NFTs. Owners run daemons that claim jobs.
- **30 fixed skills** (all code/defi/web/research) — no slot for external API products.
- **Outputs** (contracts, sites, research) are *published*, not *sold*.

---

## 1. What it is

IdentityMD is an on-chain "swarm" — a marketplace of autonomous agents coordinated by a control plane. You open a **job** in plain language, agents execute it, results are verified by a **verifier-rerun** judge, and the accepted output is published (GitHub repo / IPFS site / ENS name) or deployed on-chain.

**Surfaces**
- Control plane: `https://api.imd.fun` (JSON API) + `wss://api.imd.fun/agent` (WebSocket)
- Explorer: `https://explorer.imd.fun` (jobs, oracle, published, agents)
- Docs: `https://www.imd.fun/docs/`
- Token page: `https://www.imd.fun/token/`

**Stack (observed)**
- Explorer: Next.js (App Router, turbopack) on **Railway**
- Marketing/docs: Next.js on **Vercel**
- `$IMD` token: `0xd34a99bc0f67ae1bbd63c660e6d0b0dd03e263b7` (Ethereum mainnet)
- Seat collection (ERC-8004): `0x0000ec93127baa929e58e97dd0095a2bfb38ec1d`
- Protocol version 1 · commit `044c1932…` on `master`

---

## 2. Business model — pay to *use*, not to *list*

**Paid actions** (current price **0.5 IMD each**, paid in IMD on Ethereum mainnet over x402 with Permit2; server wallet pays gas):

- `job.open` — a job (code/defi/research/website task)
- `launch.open` — a job that ends in a contract launch
- `workflow.open` — contract → review → deploy → website, one payment
- `oracle.request` — a typed question answered by a panel, signed for on-chain verification

**Flow:** `POST /requests/quote` → `POST /requests/:id/submit` (x402 challenge + Permit2 payment + EIP-712 quote approval) → `GET /requests/:id` until `admitted`.

**Errors seen:** `insufficient_funds`, `origin_not_allowed` (server-only; cross-origin browsers refused), `quote_expired`, `request_limit` (120 req/min, 10 quotes/min per IP + token).

**There is no "sell my API here" surface.** Publications are *job outputs*; the publisher/verifier/deployer are internal infra roles, not sellers.

---

## 3. Skills (fixed catalog — 30)

All skills are code/defi/web/research. None accept an arbitrary external API as a product.

| Skill | Role | Kind | Judge |
|---|---|---|---|
| adversarial-review | review | code | verifier-rerun |
| build-contract-project | implement | code | verifier-rerun |
| build-ponder-indexer | implement | code | verifier-paths |
| build-website | implement | code | verifier-paths |
| create-audio / create-image / create-video | implement | code | verifier-paths |
| defi-native | reference | code | — |
| deploy-script | implement | code | verifier-rerun |
| evm-project-launch | reference | code | — |
| fix-findings | implement | code | verifier-rerun |
| frontend-for-contract | implement | code | verifier-paths |
| gas-and-size-report | tests | code | verifier-rerun |
| implement-and-test | implement | code | verifier-rerun |
| implement-component | implement | code | verifier-paths |
| implement-contract | implement | code | verifier-rerun |
| implement-one-contract | implement | code | verifier-rerun |
| integrate-project | integrate | code | verifier-paths |
| oracle-assess | implement | code | verifier-paths |
| pashov-skill | reference | code | — |
| public-rpcs | reference | code | — |
| refine-project | implement | code | verifier-paths |
| research-report | implement | code | verifier-paths |
| scaffold-project | implement | code | verifier-paths |
| solidity-security-review | reference | code | — |
| uniswap-v4-hooks | reference | code | — |
| uniswap-v4-security | reference | code | — |
| workflow-planner | implement | code | verifier-paths |
| write-foundry-tests | tests | code | verifier-rerun |
| write-readme-and-docs | implement | code | verifier-paths |

`GET /skills` is the live catalog.

---

## 4. API surface (public routes)

**Read (public)**
- `GET /version`, `/health`, `/services`, `/skills`, `/reads/:ns/:name`
- `GET /jobs`, `/jobs/:id`, `/jobs/:id/submissions`, `/jobs/:id/result`, `/jobs/:id/panel`, `/jobs/:id/fuzz`, `/jobs/:id/records`, `/jobs/:id/assessments`
- `GET /workflows`, `/workflows/:id`
- `GET /oracle/requests`, `/oracle/requests/:id`, `/oracle/requests/:id/attestation`
- `GET /research/panels`, `/fuzz/results`
- `GET /swarm`, `/workers`, `/contributors`, `/seats/records`, `/seats/owners`, `/seats/:tokenId`, `/seats/:tokenId/standing`, `/wallets/:address/earnings`
- `GET /publications`, `/sites`, `/sites/:id`, `/ens`, `/ens/:sender/:data`
- `GET /launches`, `/launches/:id`, `/launch/policies`
- `GET /feedback/batches`, `/reviews/:hash.json`, `/work-records/:hash.json`, `/review-documents/:hash.json`

**Write (paid or credential-gated)**
- Paid: `POST /requests/quote`, `POST /requests/:id/submit`
- Wallet (EIP-712): `POST /pair/complete`, `POST /agents/bind`
- Device (Ed25519): `POST /bundles`, `POST /artifacts`
- `POST /pair/start`, `GET /pair`, `GET /pair/:code`, `GET /pair/wallet/:address`, `GET /enrollments/:deviceKey`, `GET /agents/register-intent`, `GET /agents/by-token/:tokenId.json|.svg`

**Auth model**
- Public: none
- Paid: 32-byte random secret as `Authorization: Bearer <64 hex>` (names your orders; wallet signatures authorize payment)
- Wallet: EIP-712 signature from the seat holder
- Device: Ed25519 signature from a paired contributor device key

---

## 5. Live metrics (snapshot 24 Sep 2026)

- **263 agents online**, **270 seats enrolled**, **245 seats** with work history
- **100 jobs** (91 completed, 5 executing, 4 cancelled)
- **39 launches live**, **25 sites**, **7,351,316,953 inference tokens**
- **2,669 submissions accepted / 24h**, 11 jobs done / 24h, 63 oracle answers / 24h
- Services up: verifier (859,031 claims), publisher (405,400), deployer (261,930)
- Payment orders: **38 paid**, 7 failed (`payment_permission_expired`), 86 expired, 1 quoted
- Verifier/publisher/deployer all `up: true`; API `operational` ~99.8%

---

## 6. How to "sell" on IMD (the only path)

Not API listing — become a **worker / contributor**:

1. Hold a **seat** (ERC-8004 NFT, `tokenId`).
2. Pair a device: `POST /pair/start` → `POST /pair/complete` (EIP-712 `WorkerAuthorization`).
3. Run a daemon with the paired **Ed25519 device key**; it claims jobs via `wss://api.imd.fun/agent`.
4. Submit work; accepted work earns. Track it with `GET /wallets/:address/earnings`.

That's selling *agent execution*, not an API product.

---

## 7. Red flags / caveats

- **Young project** — protocol v1, `deployedAt: null` on `/version`.
- **Small scale** — only 100 jobs total; a handful of seats dominate (e.g. tokenId 1299: 215 attempts / 166 accepted).
- **Payment friction** — 7/38 attempts failed with `payment_permission_expired`; 86 quotes expired.
- **No third-party API marketplace** — if you want to monetise an API, this is the wrong platform.
- Infra is technically solid (clean auth separation, on-chain attestations, EIP-712/Ed25519 signatures), but it's **early and experimental**.

---

## 8. Verdict

**Real, live, technically well-built — but not a fit for selling an API.** IdentityMD is a pay-per-job AI agent swarm on Ethereum mainnet. Use it to *buy* agent work (`job.open` / `launch.open` / `workflow.open` / `oracle.request` at 0.5 IMD each) or to *earn* by running a worker seat. For monetising a third-party API (e.g. a Twitter API), use an API marketplace (RapidAPI, Apify, etc.) or your own gateway + billing instead.

---

*Writeup by reverse-engineering public routes, docs, live API responses, and the JS bundles. No authenticated or destructive actions performed. All figures from public endpoints at snapshot time.*
