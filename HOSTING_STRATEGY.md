# Hosting Strategy: Railway vs Vultr

_Last updated: 2026-07-02_

## 1. The apps in scope

| App | Current state | Stack shape |
|---|---|---|
| Alba Property AI | Running in **Railway** today | PaaS-hosted app + managed DB |
| M365 SecPosture (new) | Being built for **Railway** | PaaS-hosted app + managed DB |
| SecPosture (legacy) | Running on **Ubuntu 24.04 LTS VPS**, migrating to **Vultr** | Self-managed VM |

The question isn't "Railway or Vultr" in the abstract — it's **which workload goes where, on purpose**, plus a standard pattern so every future app lands on the right tier without re-litigating it each time.

## 2. Serverless (PaaS) vs VPS — what's actually being compared

Railway isn't "serverless" in the FaaS sense (no cold-start-per-request billing, no Lambda-style event model). It's a **PaaS**: you push a git repo, it builds a container, runs it, autoscales it, and gives you managed Postgres/Redis/MySQL as attached services. You never touch an OS.

Vultr is straight **IaaS**: you get a VM (or their own managed DB product), and you own the OS, patching, firewall, backup tooling, and deploy pipeline — or you bolt one on (Coolify, Dokku, CapRover, plain systemd + GitHub Actions).

| Dimension | Railway (PaaS) | Vultr (VPS/IaaS) |
|---|---|---|
| OS patching | Not your problem | Your problem (unattended-upgrades, cron, or manual) |
| Git → deploy | Native, first-class | You build it (Actions + SSH/rsync, or a PaaS layer like Coolify) |
| Managed DB | Built-in Postgres/MySQL/Redis as Railway services | Vultr Managed Databases (separate product, same account) |
| DB backups | Snapshot-based today; **PITR (point-in-time recovery, 7-day window) is in rollout for Pro plans** — [Railway Backups docs](https://docs.railway.com/reference/backups), [PITR announcement](https://station.railway.com/feedback/point-in-time-recovery-for-postgre-sql-d8c063e2) | Automated **daily backups with true PITR down to the minute**, up to 14-day retention on 3-node clusters, AES-256/LUKS2 encryption at rest, 99.99% SLA with automatic failover — [Vultr Managed Databases](https://www.vultr.com/products/managed-databases/postgresql/) |
| Scaling model | Autoscale within plan limits, no capacity planning | You size the VM; scale = resize or add nodes yourself |
| Networking control | Limited — Railway-managed edge, private networking between services in a project (up to 100 Gbps) | Full control — VPC, firewall rules, WireGuard, Cloudflare Tunnel, whatever you want |
| Cloudflare Tunnel fit | Works as an **outbound sidecar process** inside your service container (cloudflared just needs outbound HTTPS) — no inbound ports needed either way | Native and first-class — `cloudflared` as a systemd service, full control over what's exposed |
| Operational skill required | Low — you don't need sysadmin skills | Medium/high — you already have this, which changes the calculus for you specifically |
| Blast radius of a mistake | Contained by the platform | Yours to contain (backups, snapshots, firewall discipline) |
| Cost model | Usage-based, scales with the app, no idle-server cost | Fixed VM cost regardless of load, cheaper at steady, predictable load |

**The honest framing for you specifically:** the "VPS is harder" argument is weaker in your case because you already run Ubuntu in production and know how to operate it. The real trade-off isn't skill, it's **who owns business-continuity risk for the database** — and right now Vultr's managed DB backup story (true PITR, 14-day retention, 99.99% SLA) is more mature than Railway's (snapshot-based, PITR still rolling out to Pro tier). That's a meaningful data point given you said "Railway backups are rock solid and that just works for me" is your ideal — as of today, that statement is **aspirational, not yet fully true**, though Railway is actively closing the gap.

## 3. Decision framework

Score each app against these questions, in order:

1. **Does it need a real database with strict RPO/RTO (business continuity)?**
   If yes and the recovery point matters to the minute (compliance data, security posture findings, client-facing SLAs) → weight toward Vultr Managed Database or self-hosted Postgres with your own WAL-archiving/pgBackRest setup, regardless of where the app compute lives.
2. **Is the team optimizing for shipping velocity over infra control?**
   If yes → Railway. Git push → deploy, zero YAML, zero OS.
3. **Does the workload have spiky/idle-heavy traffic (dev tools, internal apps, low-traffic SaaS)?**
   If yes → Railway's usage billing wins on cost.
4. **Does the workload have steady, predictable load, or need specific OS-level control (custom agents, kernel tuning, non-standard ports, long-running background workers with heavy local state)?**
   If yes → Vultr VPS.
5. **Is this a security-posture / compliance-relevant app where you need to show an auditor exactly what's patched, what's exposed, and what the backup/retention policy is?**
   If yes → Vultr, because you can produce the artifact (systemd units, firewall rules, backup logs, Cloudflare Zero Trust access logs) directly instead of pointing at a vendor's shared responsibility page.

## 4. App-by-app read

**Alba Property AI (Railway, already running)** — Leave it. It's working, no BC-critical compliance angle described, and re-platforming a running app for a marginal backup improvement isn't worth the churn. Do enable Railway's PITR the moment it's GA on your plan, and add an independent nightly `pg_dump` to off-platform storage (S3/R2) as a belt-and-braces measure — cheap insurance against relying on one vendor's backup subsystem alone.

**M365 SecPosture (new build, targeting Railway)** — This is a security-posture tool touching Microsoft 365 tenant data. That's exactly the category where "can I prove my RPO/RTO and access controls to a customer or auditor" matters. Two reasonable paths:
   - Keep compute on Railway for deploy velocity, but point it at a **Vultr Managed Postgres** instance (or any DB with real PITR) instead of Railway's bundled DB, for the backup guarantees. Railway supports connecting to external databases via connection string — this decouples "where the app runs" from "where the data of record lives."
   - Or, if cross-cloud latency/complexity isn't worth it, stay all-Railway and layer your own `pg_dump` → R2/S3 cron as the backstop until Railway PITR is GA and proven.

**Legacy SecPosture (Ubuntu 24.04 LTS VPS → Vultr)** — This is already a VPS workload; moving it to Vultr is a lift-and-shift, not a re-architecture. This is the natural home for your standards-based VPS pattern (below). Good candidate to also front with Cloudflare Tunnel to kill the public-facing attack surface entirely — no open port 443 to the internet, just outbound `cloudflared`.

## 5. Standards-based build pattern going forward

Adopt this so every new app is a decision, not a debate:

**Tier A — Railway (default for new apps)**
- Use when: fast iteration, no bespoke OS needs, DB backup requirements are "good enough" (or offloaded to an external managed DB).
- Git-based auto-deploy from `main`/`master` with PR preview environments for anything client-facing.
- Attach Railway Postgres for low-stakes data; use an external managed Postgres (Vultr) for anything with real compliance/BC requirements.
- Add an independent off-platform backup job (`pg_dump` → object storage, scheduled) as standard practice regardless of which DB backend is used — never rely on a single vendor's backup mechanism as your only copy.
- Secrets via Railway's environment variables; never commit `.env` files (verify `.gitignore` on every new repo).

**Tier B — Vultr VPS (for compliance-sensitive, steady-load, or OS-control workloads)**
- Base image: Ubuntu 24.04 LTS, standardized across all VPS apps (matches what you already run — one patching/hardening playbook for everything).
- Provisioning: cloud-init or a small Ansible playbook, checked into git, so a box can be rebuilt from scratch in minutes — this is your actual DR plan, more than backups alone.
- Auto-deploy from git: GitHub Actions workflow that SSHs in and runs a deploy script, or self-hosted Coolify/Dokku if you want a Railway-like git-push experience on your own VM. Pick one pattern and reuse it across all Vultr apps rather than hand-rolling per app.
- Database: Vultr Managed Database (Postgres/MySQL) rather than self-hosting Postgres on the same VM as the app — keeps the DB's backup/failover/PITR story managed and separates the app's blast radius from the data's.
- Firewall: default-deny inbound, Vultr Cloud Firewall + `ufw` as belt-and-braces, nothing exposed except SSH (ideally via Tailscale/Cloudflare Access instead of public SSH) and whatever Cloudflare Tunnel needs.
- Patching: unattended-upgrades for security patches, scheduled reboot window, monitored.

**Cloudflare Tunnel — apply to both tiers**
- On Vultr: install `cloudflared` as a systemd service, tunnel to `localhost:<app-port>`, zero inbound firewall rules for web traffic, Cloudflare Access policies (SSO/MFA) in front of anything admin-facing.
- On Railway: `cloudflared` can run as a lightweight sidecar/second service in the same project, tunneling to the app's private-network hostname. It's an outbound-only connection either way (cloudflared reaches out to Cloudflare's edge — see [Cloudflare Tunnel docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)), so this works identically in spirit on both platforms, though it's more natural on a VM you control.
- Standardize: every app, regardless of host, sits behind a Cloudflare Tunnel + Access policy rather than a bare public origin. This is the one piece of the pattern that should be non-negotiable across both tiers.

**Backup policy — apply to both tiers**
- Minimum bar for any app holding real data: automated daily backups, retention ≥ 7 days, at least one copy off the hosting provider (S3/R2), and a documented/tested restore procedure — not just "backups exist," but "we've restored one and timed it."
- For compliance-relevant apps (M365 SecPosture, legacy SecPosture): PITR-capable DB (Vultr Managed DB today; Railway once PITR is GA and proven), documented RPO/RTO, and quarterly restore drills.

## 6. Bottom line

- **Don't force a single-vendor answer.** Railway for velocity, Vultr for control and stronger managed-DB backup guarantees today. Mixing (Railway compute + Vultr-managed Postgres) is a legitimate, low-friction pattern if backup rigor matters more than architectural purity for a given app.
- **Cloudflare Tunnel works on both** — make it standard everywhere, not just on Vultr.
- **Re-check Railway PITR status before the next app decision** — once it's GA and battle-tested, the backup-driven reason to reach for Vultr weakens and Railway becomes the default for nearly everything except apps that need genuine OS-level control.
- **Legacy SecPosture → Vultr** is straightforward; use it as the reference implementation of the Tier B pattern (Ubuntu 24.04 LTS baseline, cloud-init/Ansible provisioning, Vultr Managed DB, Cloudflare Tunnel, unattended-upgrades) so the next VPS app is a copy-paste, not a redesign.

---

### Sources
- [Railway — Backups docs](https://docs.railway.com/reference/backups)
- [Railway — Point-in-Time Recovery announcement](https://station.railway.com/feedback/point-in-time-recovery-for-postgre-sql-d8c063e2)
- [Railway — Automated PostgreSQL Backups](https://blog.railway.com/p/automated-postgresql-backups)
- [Vultr — Managed PostgreSQL](https://www.vultr.com/products/managed-databases/postgresql/)
- [Vultr — Managed Databases overview](https://www.vultr.com/products/managed-databases/)
- [Cloudflare — Cloudflare Tunnel docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)
- [Cloudflare — Private networks with cloudflared](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/private-net/cloudflared/)
