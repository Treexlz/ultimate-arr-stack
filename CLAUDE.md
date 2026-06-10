# Claude Code Instructions

## NAS Access

SSH credentials are in `.claude/config.local.md`. Read it before running any NAS commands.

## Project Structure

Docker media stack for Ugreen NAS. Edit NAS files (like `pihole/dnsmasq.d/02-local-dns.conf`) **on the NAS**, not locally.

- **Local dev repo**: `/Users/adamknowles/dev/ultimate-arr-stack/`
- **NAS deploy path**: `/volume1/docker/arr-stack/`

## Cross-Stack: Therapy Stack

A separate `therapy-stack` runs at `/volume1/docker/therapy-stack/` on its own network (`therapy-net`, 172.21.0.0/24). Baserow is also on the `arr-stack` network (static IP 172.20.0.20) so Traefik can route to it.

**Files referencing therapy-stack:** `pihole/dnsmasq.d/02-local-dns.conf`, `traefik/dynamic/therapy.local.yml`

**IMPORTANT:** Baserow's static IP (172.20.0.20) is critical. Without it, Docker can assign Gluetun's IP (172.20.0.3) to Baserow on reboot, breaking the VPN stack. The `ip_range: 172.20.0.128/25` in `docker-compose.traefik.yml` confines dynamic IPs to 128-255.

Therapy-stack local repo: `/Users/adamknowles/dev/n8n Therapybot/Git repo/`

## Deploying to the NAS

Deploy via git only — never SCP files, never edit compose on the NAS directly, never `docker stop` + ad-hoc `docker run` to "pre-test" against live state.

**Classify the change first — this determines the flow:**

- **Trivial patch image bump** (e.g. cloudflared point release): commit + push to `main` → on NAS `git pull --ff-only` → `docker compose -f <file>.yml up -d` → verify.
- **Risky change** — anything that can fail or migrate data: **minor/major version bumps (especially with a DB migration, e.g. Seerr 3.2 → 3.3), env / network / volume changes.** Verify on the NAS *before* it reaches `main`:
  1. Commit to a **feature branch**, push the branch (not `main`).
  2. On NAS: `git fetch && git checkout <branch>`, then recreate the affected service.
  3. **Verify it's actually healthy on the NAS** (container healthy, API responds, migration clean).
  4. Only then fast-forward `main` and push `main`; check `main` out again on the NAS.

The push to `main` comes **after** the NAS proves the change works — not before. If unsure which bucket a change is in, treat it as risky and branch-test first. Back up the service's config volume before any version bump with a migration (`docker run --rm -v <vol>:/src:ro -v <dir>:/bak alpine tar czf /bak/<svc>-config-backup-<stamp>.tgz -C /src .`). Roll back with `git revert` (or `git checkout main`) → pull → recreate.

## E2E Tests

Run `npm run test:e2e` after any change to Docker Compose files, service config, networks, or ports. All 13 tests must pass. They screenshot every service UI and verify API responses.
