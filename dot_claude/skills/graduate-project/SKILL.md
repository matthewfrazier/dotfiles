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

## 4. Own repo + provision the CT
- Create `matthewfrazier/<project>` (private), commit the extracted+adapted source + a `CLAUDE.md`
  documenting deploy target, architecture, and host-specific gotchas.
- Provision the dedicated CT via `local-network-deploy` (sizing, `--features nesting,keyctl` for
  docker, GPU/kvm passthrough, `--onboot 1`). Deploy the repo, build, verify.

## 5. Reboot-resilience (do NOT skip — hard-won)
A dedicated CT is worthless if it silently dies in the next maintenance reboot (this happened to the
emulator at the 04:00 pve upgrade — the container had `restart: unless-stopped` but was GONE after
reboot; homelab #16). Belt AND suspenders:
- `onboot: 1` on the CT (so the LXC restarts).
- A **systemd boot service** that `docker compose up -d`s the stack (recreates it even if the
  container was removed, not just stopped). `After=docker.service`, `Type=oneshot`,
  `RemainAfterExit=yes`.
- If it hosts a long-lived agent, also a boot service that launches it (see the agent-session pattern).

## 6. Cut consumers over — VERIFY the endpoint live first
- A wrong host:port **fails silently** (tests just fail). Verify the endpoint from OUTSIDE the CT
  (as a real consumer connects) BEFORE handing it out — e.g. `adb connect <ip>:5556` returns a booted
  device.
- Hand the endpoint to consumers/owners via the bus **in the message BODY** (`hub send <to>
  "<subject>" - <<EOF ...EOF` — subjects cap ~180 chars and silently truncate).
- Add a contention gate if it's now multi-tenant (an ssh+flock lease so two consumers don't drive one
  stateful device at once).

## 7. Retire the old instance
- Stop the old copy (keep it as rollback, don't delete yet). If it's a peer agent's host, do this
  JOINTLY.
- Once consumers are green on the new CT, remove the old container + its image/compose from the shared
  host, and strip any now-unused passthrough (e.g. the shared host's iGPU mapping) from its config.

## Related
- `local-network-deploy` — provision the capped LXC (the mechanics of step 4).
- `migrate-container` — faithful state move (pet-vs-cattle, docker commit, transfer mechanics) when
  the project has heavy volume state.
