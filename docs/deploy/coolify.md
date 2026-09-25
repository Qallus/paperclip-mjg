---
title: Coolify
summary: Deploy Paperclip to a Coolify-managed VPS on your own domain
---

Deploy Paperclip to a Coolify-managed VPS, reachable at your own subdomain over
HTTPS. Coolify runs Paperclip and Postgres together and terminates TLS at its
bundled Traefik proxy.

This guide uses `paperclip.michaeljgauthier.com` as the example domain.

## Pick a path

A Git repo is **optional**. Choose based on whether you intend to modify
Paperclip's source.

| | **Path A — prebuilt image** | **Path B — build from Git** |
|---|---|---|
| Git repo needed | No | Yes, your own |
| Compose file | [`docker-compose.coolify-image.yml`](../../docker-compose.coolify-image.yml) | [`docker-compose.coolify.yml`](../../docker-compose.coolify.yml) |
| Coolify resource | Docker Compose (Empty) — paste the file | Docker Compose — read from repo |
| VPS sizing | 1–2 GB RAM is fine | 4 GB minimum, 8 GB comfortable |
| First deploy | ~2 min (image pull) | ~10–20 min (full source build) |
| Deploy your own code changes | No | Yes |
| Auto-deploy on `git push` | No | Yes |

**Start with Path A.** It is the fastest way to a working instance, and you can
switch to Path B later without losing data — the volumes and environment
variables are identical between the two files.

Path A uses `ghcr.io/paperclipai/paperclip:latest`, which is public, needs no
registry login, and ships for both amd64 and arm64.

## Before you start

**Sizing (Path B only).** The Dockerfile is a full source build: it compiles the
Rust runner (`paperclip-runnerd`), installs the pnpm workspace, builds the
server and UI, and `npm install`s four agent CLI toolchains. Budget:

| Resource | Minimum | Comfortable |
|---|---|---|
| RAM | 4 GB | 8 GB |
| Disk | 30 GB free | 50 GB |
| Build time (cold) | ~20 min | ~10 min |

A 2 GB VPS will OOM during the Rust or UI build. Use Path A there.

**Security.** Paperclip runs agent CLIs that execute arbitrary code and shell
commands inside the container. Anyone who gets an account on a public instance
can run code on your VPS. Treat account access as root-equivalent: use a strong
password, and disable sign-up immediately after creating your first account
(step 6). If you do not need internet access, Coolify can keep the service on a
private network or behind its own basic auth instead.

## 1. Point DNS at the VPS

Create an `A` record with your DNS provider:

```
paperclip.michaeljgauthier.com.  A  <your-vps-public-ip>
```

Wait for it to resolve before creating the Coolify resource — Let's Encrypt
issuance fails if the record is not live yet.

```sh
dig +short paperclip.michaeljgauthier.com
```

Make sure ports 80 and 443 are open on the VPS firewall. Port 80 must stay open
for the ACME HTTP-01 challenge, even though all traffic ends up on 443.

## 2. Create the Coolify resource

### Path A — prebuilt image (no Git repo)

In Coolify: **Project → New Resource → Docker Compose (Empty)**.

Paste the entire contents of `docker-compose.coolify-image.yml` into the
compose editor and save. There is no repository, no branch, and no build step —
Coolify pulls the published image on deploy.

### Path B — build from Git

Coolify builds from a Git repo, not from your laptop, so your clone has to be
pushed somewhere Coolify can reach. The upstream `paperclipai/paperclip` repo
will not work: it does not contain `docker-compose.coolify.yml`, which is a
file added by this setup.

Create an empty repo under your own GitHub account (private is fine), then:

```sh
git remote add deploy git@github.com:<you>/paperclip.git
git add docker-compose.coolify.yml docs/deploy/coolify.md
git commit -m "Add Coolify deployment config"
git push deploy master
```

In Coolify: **Project → New Resource → Private Repository (with GitHub App)**,
or **Public Repository** if you made it public. A private repo needs Coolify's
GitHub App installed on it, or a deploy key — Coolify walks you through either.

| Field | Value |
|---|---|
| Repository | your repo |
| Branch | `master` |
| Build Pack | **Docker Compose** |
| Docker Compose Location | `/docker-compose.coolify.yml` |
| Base Directory | `/` |

Save. Do **not** deploy yet — the environment variables come first.

## 3. Set environment variables

Generate the secrets locally:

```sh
openssl rand -hex 24   # POSTGRES_PASSWORD
openssl rand -hex 32   # BETTER_AUTH_SECRET
openssl rand -hex 32   # PAPERCLIP_TOOL_ACTION_SIGNING_SECRET
```

Add them under **Environment Variables**, all marked as build-time-unneeded
runtime secrets:

| Variable | Value | Notes |
|---|---|---|
| `POSTGRES_PASSWORD` | generated | Changing this later orphans the existing database |
| `BETTER_AUTH_SECRET` | generated | Rotating it invalidates all sessions |
| `PAPERCLIP_TOOL_ACTION_SIGNING_SECRET` | generated | Signs tool-action approvals |
| `ANTHROPIC_API_KEY` | `sk-ant-…` | Optional, but agents need at least one provider key |
| `OPENAI_API_KEY` | `sk-…` | Optional |
| `PAPERCLIP_AUTH_DISABLE_SIGN_UP` | `false` | Flip to `true` after step 6 |
| `PAPERCLIP_PUBLIC_URL` | `https://paperclip.michaeljgauthier.com` | Full origin, scheme included, no trailing slash |
| `TRUST_PROXY` | `uniquelocal` | Already defaulted in the compose file; only change it if you front Coolify with another proxy |

> **Do not try to set `SERVICE_FQDN_PAPERCLIP_3100` by hand.** Coolify owns
> every `SERVICE_FQDN_*` name: it rewrites the value from the service's Domain
> field on each deploy and silently discards anything you enter. That is why
> `PAPERCLIP_PUBLIC_URL` is a separate variable rather than being derived from
> it.

Keep these out of the repo. The compose file refuses to start if the three
generated secrets are missing, rather than silently booting with a weak default.

## 4. Set the domain

On the **`paperclip` service**, set the domain to:

```
https://paperclip.michaeljgauthier.com
```

**Set the port to 3100**, not Coolify's default of 3000. The port is part of
the magic variable's name (`SERVICE_FQDN_PAPERCLIP_<port>`), so a mismatch
leaves the variable in the compose file unset *and* points the proxy at a port
nothing listens on. The symptom is a deploy log line reading
`"SERVICE_FQDN_PAPERCLIP_3100" variable is not set. Defaulting to a blank
string.` followed by "no server available" in the browser.

From that field Coolify generates the Traefik routing labels and requests the
Let's Encrypt certificate.

Leave the `db` service with no domain.

## 5. Deploy

Hit **Deploy** and watch the build logs. The first build is the slow one;
subsequent deploys reuse Docker layer cache.

Healthy startup looks like:

```
plugin job coordinator started
plugin-loader: loadAll complete
```

Verify from your machine:

```sh
curl -sf https://paperclip.michaeljgauthier.com/api/health
# -> {"status":"ok"}
```

## 6. Create your account, then close the door

Open `https://paperclip.michaeljgauthier.com`. The **first account to sign up
becomes the admin**, so do this immediately after the deploy goes healthy.

Then set `PAPERCLIP_AUTH_DISABLE_SIGN_UP=true` in Coolify and redeploy. After
that, add people through the in-app invite flow rather than open registration.

## Updating

**Path A:** hit **Deploy**. Coolify re-pulls `:latest` and restarts onto the
newer image. If you pinned a digest, bump it first.

**Path B:** push to your branch and hit **Deploy**, or enable Coolify's
auto-deploy webhook so pushes redeploy automatically.

Either way, database migrations run on boot.

A build that fails never replaces the running container. Whether the *swap*
itself is zero-downtime depends on Coolify's rolling-update setting for the
resource; with it off, expect a short gap while the new container boots and
passes the health check defined in the compose file.

## Backups

Two things hold state, and you need both:

- **Postgres** (`pgdata` volume) — all application data. Coolify can schedule
  automated Postgres backups to S3-compatible storage; turn that on.
- **`paperclip-data` volume** — uploads, agent workspaces, and the local
  secrets master key at `/paperclip/instances/default/secrets/master.key`.
  Losing this volume means stored secrets can no longer be decrypted, even
  with an intact database.

## Moving from Path A to Path B later

Nothing is thrown away. The two compose files declare the same service names,
the same volumes (`pgdata`, `paperclip-data`) and the same environment
variables — Path B only swaps the `image:` line for a `build:` block.

On the same Coolify resource, replace the compose contents with
`docker-compose.coolify.yml` and attach the Git repo, or create a new resource
and point it at the existing volumes. Your environment variables carry over
unchanged, so the database and secrets master key stay intact.

## Pinning a version

`:latest` moves on every stable release, which means a redeploy can pick up a
new version at a moment you did not choose. On anything you care about, pin the
digest instead:

```yaml
    image: ghcr.io/paperclipai/paperclip@sha256:a02ac35ac41df911af477422ea0e781cf41d2b2c600c66f0a5ac9d8c63f52c2c
```

Read the current digest for a tag with:

```sh
docker buildx imagetools inspect ghcr.io/paperclipai/paperclip:latest
```

The registry also publishes `:nightly`, `:beta` and `:canary` channels, plus a
`sha-<commit>` tag for every build on `master`.

## Troubleshooting

**Build is killed / exits 137 (Path B).** Out of memory. Add swap, resize the
VPS, or switch to Path A.

**Certificate never issues.** Check DNS actually resolves to the VPS, port 80 is
open, and no other service is bound to 80/443 ahead of Coolify's proxy.

**Redirect loop or wrong links in the UI.** `PAPERCLIP_PUBLIC_URL` does not
match the browser URL. It must be the exact external origin including `https://`
and no trailing slash.

**Server refuses to start citing exposure checks.** Public mode requires an
explicit non-loopback `PAPERCLIP_PUBLIC_URL`. Confirm the domain is set on the
`paperclip` service, not only on the project.

**Agents fail with missing prerequisites.** No provider key is set. Add
`ANTHROPIC_API_KEY` or `OPENAI_API_KEY` and redeploy.
