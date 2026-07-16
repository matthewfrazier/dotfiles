---
name: graduate-project
description: >
  Graduate a project OUT of a shared/multi-purpose host into its OWN dedicated Proxmox CT, with its
  own repo and identity, then retire it from the shared host. Use when a sub-project has become a
  shared/contended resource, needs to survive independently, or its crash/resource profile threatens
  its host — e.g. "graduate <x>", "give <x> its own CT/LXC/box", "extract <x> from <host> to a
  dedicated container", "spin <x> off into its own project", "split <x> out of <host>", "<x> should
  be its own thing not part of <host>". Distilled from the android-emulator graduation (fd-dev CT105
  -> dedicated CT113 + repo matthewfrazier/android-emulator, 2026-07) and the ai-radio split
  (te-eval-38 -> ai-radio). Pairs with `local-network-deploy` (provisions the CT) and
  `migrate-container` (faithful state move). This skill is the DECISION + END-TO-END orchestration:
  extract, re-home, provision, cut over, retire.
---

# Graduate a project to its own dedicated CT

A sub-project born inside a shared host (a dev container, an eval box, an agent's workspace)
outgrows that home: it becomes a resource multiple things depend on, its crashes take down the
host, or it just deserves its own identity. Graduating = give it a dedicated CT, its own repo, a
name for what it IS, reboot-resilience, then retire the old instance. Do NOT confuse with
`migrate-container` (that's a faithful lift-and-shift of one container between hosts).

## 1. Decide (before touching anything)
- **Why graduate?** Shared-resource contention (two consumers want it), blast-radius (it crashes the
  host — the emulator's SwiftShader zombies took down fd-dev), or identity (it's a distinct product
  mis-filed under the host's namespace — `te-eval-38` was really `ai-radio`).
- **Name it for what it IS**, not the shared host's namespace. The rename is part of the graduation.
- **Own repo?** Almost always yes: `matthewfrazier/<project>` (private). Stop filing it as an issue in
  the parent's tracker.
- **Ownership boundary.** If the shared host is another agent's domain (fd-dev-IT, threadeval), the
  retirement is JOINT — coordinate, don't unilaterally stop their workload (the auto-classifier and
  operator both enforce this). Often the cleanest is: THEY hand you the source location; YOU extract
  + stand up the new CT; you retire the old TOGETHER.

## 2. Extract the source (+ state)
- Pull the project's build context / code from its **canonical repo** if it has one (clone at the
  exact commit); the shared host's working checkout is a fallback. For the emulator: source was in
  `flavordrake/mobissh docker/emulator/@<commit>`, also in CT105's docker volume.
- Identify STATE vs code: baked-into-image (nothing to move) vs volumes/config/DB (must copy — see
  `migrate-container` pet-vs-cattle). The emulator AVD was baked into the image (no volume). A radio's
  config/playlist is state.
- Cross-account footgun: do NOT grant the shared host's identity push access to the new repo. You
  hold both (as the operator's identity) — pull the source and commit it yourself.

## 3. Adapt for standalone
Strip the shared-host coupling and give it its own runtime:
- Remove sibling-container / same-host assumptions (mobissh's `test-sshd` bridge, in-host DNS names).
- Give it its own `docker-compose.yml` / config. For a network-reachable service, `network_mode: host`
  so it binds the LXC's own IP (reachable across LAN/tailnet) instead of a private docker net.
- **Host-specific values must be remapped** — GPU render/video gids (104/44), device paths. On pve,
  the `dev0/dev1 gid=` mapping makes the in-LXC gids deterministic regardless of the host's gids.

## 4. Own repo
- Create `matthewfrazier/<project>` (private). If the source is a fork of someone else's repo (the
  radio was a clone of `keltokhy/writ-fm`), **fork it** and keep `upstream` wired, rather than
  pushing their history into a fresh repo — decide this with the operator.
- Commit the extracted+adapted source + a `CLAUDE.md`/overlay note documenting deploy target,
  architecture, host gotchas. **Scrub secrets before the first push** (public forks especially):
  externalize hardcoded passwords to a gitignored env file, gitignore any generated file that embeds
  tokens (a Jellyfin playlist embedded 500 `api_key`s), ship `.example` templates. `git diff --cached
  | grep -iE 'password|api_key|token'` before pushing.

## 5. Provision: clone the pve `agent-template` (the default path)
Do NOT hand-provision — clone the golden template CT. It is a real PVE feature (`pct clone` of a
`template: 1` CT; LXC has no cloud-init, so post-clone file injection via `pct exec` is the standard
mechanism, not a workaround). The template (`agent-template`, currently CT199) already bakes the
parts that cost real time to discover:
- **Config-level** (carried by `pct clone`): `/dev/net/tun` passthrough + `nesting,keyctl` features
  so tailscale runs in **kernel mode** (userspace mode can `serve` inbound but can't reach the
  tailnet-only hub to enroll — see [[pve-lxc-agent-gotchas]]). No snapshot-section append trap.
- **Rootfs-level**: claude, docker (+log-rotation `daemon.json`), gh, tree, node, tailscale
  (logged-out), the reboot-safe agent-session trio, and `~/.claude/settings.json` that defaults the
  agent to **Sonnet** (`"model":"sonnet"`) + `skipDangerousModePermissionPrompt`. **No baked auth.**

One command does clone → identity → project → enrolled → booted (runs on the pve host; raserver
drives it): `scripts/graduate-clone.sh --id <newid> --name <hostname> [--repo <url>] [--workdir <path>] [--brief <file>]`.
It clones, sets `hostname`+`onboot:1`, copies claude OAuth creds from a live agent CT
(container-to-container, never over the wire), sets `AGENT_WORKDIR`+brief+repo, pre-accepts the
workdir trust, runs the hub enroll (registers on agent-mail under the hostname), and starts the agent.
CAVEAT on copied creds: OAuth refresh tokens ROTATE — hosts sharing a copied `.credentials.json`
invalidate each other on refresh (ai-radio silently logged out a day after graduation this way).
Copy-from-live works for bring-up; the durable fix is a per-host `claude setup-token` from the vault
(homelab #17). If a graduated session shows "Not logged in", re-copy from the freshest live host.

## 6. Give it its own agent session (identity + brief + remote control)
The graduated project's end state is a **Remote-Control-registered Claude session named
`<hostname>`** — visible in the operator's mobile/web Claude Code session list ("Connected ·
Remote control") — reachable locally by a bare **`tmux a`** on the host. The template's trio
delivers this reboot-safely (verified across a real `pct reboot`):
- `/root/run-agent.sh` — sources `/root/agent.env` (`AGENT_WORKDIR`), `cd`s there, `exec claude
  --dangerously-skip-permissions --remote-control "$(hostname)" "$(cat /root/agent-brief.md)"`.
  The `--remote-control` flag is what puts the session on the operator's phone — a graduation
  isn't done until the operator can see it there.
- `/usr/local/bin/launch-agent-tmux.sh` — `tmux new-session -s main` (plain name, so `tmux a`
  attaches with zero ceremony), sends `IS_SANDBOX=1 /root/run-agent.sh`.
- `agent-session.service` — oneshot, `RemainAfterExit`, enabled. (NB: `systemctl restart` kills the
  session via `KillMode=control-group` — that is NOT the boot path; test resilience with a real reboot.)
Write a real **brief** (`/root/agent-brief.md`): who it is, what it owns, boundaries (**it's a pve
guest — request pve/host ops from raserver via the bus**, don't self-drive), and conventions
(event-only bus monitoring, no heartbeats; **Sonnet default, escalate with `/model` on demand**;
secrets via the vault). Then `systemctl restart agent-session`.

## Credentials for a graduated agent
A fresh agent has claude auth (copied) but **no git-push credential or app secrets**. Route these
through the **Infisical vault** (agent-hub CT106, tailnet-only `https://agent-hub.tailbe5094.ts.net:8450`,
project `homelab-agent-secrets`) — the vault IS stood up (don't let a stale bus message convince you
otherwise). Flow: agent creates an empty entry via the `homelab` machine identity → operator commits
the value in the UI (prod env) → agent reads it via API. **Never** paste tokens in chat/bus/code, and
never repurpose a read-only/wrong-scope key. See [[hub-send-subject-body]] for delivering the request.

**GitHub access is baked into the template** (mechanism, not the secret): `gh-cred.sh` is a git
credential helper that picks the token by repo OWNER — `GH_TOKEN_MF` for `matthewfrazier/*` (the
agent's own project repos), `GH_TOKEN_FD` for `flavordrake/*` (devloop commit-back). Both are
fine-grained PATs pulled from the vault by `graduate-clone.sh` at clone time into
`/root/.config/gh-tokens.env` — the vault machine-identity lives once on the pve host
(`/root/.infisical-graduate.env`), only the PATs reach the clone. **Identity hygiene:** flavordrake is
the operator's pseudonymous PUBLISH identity — commits into the live devloop checkout
(`/root/.local/share/devloop`) are authored as the flavordrake persona (GitHub noreply) via a
`.gitconfig` `includeIf`, so committing enhancements back never links the matthewfrazier identity
publicly. The **devloop plugin is pulled live** (`devloop-sync.sh`), never baked, so agents run the
latest and can commit improvements straight back to `flavordrake/devloop`.

## Maintaining the `agent-template`
The template is read-only. To update it as we learn: `pct clone <tmpl-id> <tmp-id>`, start, patch the
rootfs, clear identity state (`rm /var/lib/tailscale/tailscaled.state`, any baked creds), stop,
`pct set --hostname agent-template`, `pct template <tmp-id>`, then destroy the old template CT.

## 7. Cut consumers over — VERIFY the endpoint live first
- A wrong host:port **fails silently** (tests just fail). Verify the endpoint from OUTSIDE the CT
  (as a real consumer connects) BEFORE handing it out — e.g. `adb connect <ip>:5556` returns a booted
  device.
- Hand the endpoint to consumers/owners via the bus **in the message BODY** (`hub send <to>
  "<subject>" - <<EOF ...EOF` — subjects cap ~180 chars and silently truncate).
- Add a contention gate if it's now multi-tenant (an ssh+flock lease so two consumers don't drive one
  stateful device at once).

## 8. Retire the old instance
- Stop the old copy (keep it as rollback, don't delete yet). If it's a peer agent's host, do this
  JOINTLY.
- Once consumers are green on the new CT, remove the old container + its image/compose from the shared
  host, and strip any now-unused passthrough (e.g. the shared host's iGPU mapping) from its config.

## Related
- `local-network-deploy` — provision the capped LXC (the mechanics of step 4).
- `migrate-container` — faithful state move (pet-vs-cattle, docker commit, transfer mechanics) when
  the project has heavy volume state.
