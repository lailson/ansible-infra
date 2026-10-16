# Changelog

All notable changes to `fabbrito.infra`. The version here, the version in `galaxy.yml`, and the git tag move together —
consumers pin the tag, so a change that is not released is a change nobody gets.

A released tag is never repointed. 1.0.0 moved while the repo was private and nothing pinned it; going public ended
that, and the next correction is 1.0.1.

## 1.2.3

### Changed

- The rendered `/etc/caddy/Caddyfile` lands `0640 root:caddy` instead of `0644`. Defense in depth: route auth material
  (basic_auth hashes) is valid Caddyfile content a consumer's routes can carry, and the service reads the file through
  its group anyway — `caddy validate` and the reload run as root. A declared exception to ADR-0001's `no_log` rule: what
  a route can carry here is a bcrypt hash, never a plaintext, and censoring the task would censor the `caddy validate`
  stderr a broken render needs to fail with.

## 1.2.2

### Changed

- `caddy_version` is 2.11.7, the release current when the Cloudsmith repo went dark. 1.2.1 pinned 2.11.4, which was
  behind 2.11.6 (2026-10-01) and 2.11.7 (2026-10-03): a host that had taken either through apt before 2026-10-09 would
  have failed the install, because `apt deb:` refuses a downgrade — and `--check` hides that, since it skips the install
  on a host without the cache dir. A converged host sees Caddy upgraded and restarted by the package's postinst.
- ADR-0004 is amended: the install mechanism is the release `.deb`, upgrades are a `caddy_version` bump, and the
  `caddy upgrade` warning says what actually happens (the stray binary survives until the next bump, it is not clobbered
  by a re-run).

## 1.2.1

### Fixed

- `caddy` installs from the GitHub release `.deb`, verified against the release's sha512 (`caddy_version`,
  `caddy_deb_sha512`), instead of the Cloudsmith apt repo. Cloudsmith started answering 402 on 2026-10-09
  (caddyserver/caddy#8184), which failed every `caddy` converge at `apt update` — and, because it was an apt source,
  every other role's `apt update` on the same host. The role now removes that source and its key.
- Patch rather than minor: the installed package is the same build, and a converged host sees no Caddy change.

## 1.2.0

### Changed

- `os` no longer names the box unless asked to. `os_hostname` previously defaulted to a derivation over
  `os_hostname_domain`, so a consumer who set neither still got the bare inventory key written to the host — renaming a
  box already carrying an FQDN, and rewriting the `127.0.1.1` entry the cloud image ships, on a converge that reported
  green. Both vars now default to empty and the hostname tasks skip together, which is what the README already claimed
  the row did.
- Minor rather than major: nothing became required, and the only behaviour that moved is the case that was wrong. A
  fleet that wants the old bare-key naming asks for it explicitly with `os_hostname: "{{ inventory_hostname }}"`.
- `os_hostname` is now the per-host override rather than a derived value — set it or `os_hostname_domain`, not both.

## 1.1.0

### Changed

- `bootstrap` asserts `deploy_authorized_keys` is defined and non-empty, before it creates anything. Empty was the case
  worth closing: the key loop was a no-op over `[]`, so the play went on to grant passwordless sudo and exited
  **green**, leaving an account nobody holds a key for on a box that reports itself converged. Unset died at the key
  loop instead, after the account existed but before the sudoers write.
- Minor rather than major, deliberately: [ADR-0001](docs/adr/0001-secrets-in-vault-and-an-absent-secret-skips.md) and
  the README contract both already listed the var as required, so the contract did not move — only its enforcement. No
  converged fleet can be running with it empty either, because such a host was never reachable as `deploy_user` to begin
  with.

## 1.0.1

- Citations in rendered files are qualified with the collection. `# Source:` paths, ADR references and `docs/` pointers
  now read `fabbrito.infra …`; unqualified, they resolved against the consuming repo's own tree, where `ADR-0006` is
  somebody else's decision and `roles/caddy/…` is somebody else's role. Affects the bytes of every rendered file, so the
  goldens moved with it.
- A `v*` tag now fails CI unless it matches `galaxy.yml`'s version. An installed tree records no git ref, so the version
  a consumer compares their pin against is `MANIFEST.json`'s; a tag that disagrees reaches them as "pinned X, installed
  Y", which no reinstall can fix. It also means a moved tag installs clean and matches — undetectable downstream, so
  this side is the only one that can refuse it. `make tag-check TAG=v1.0.1` runs the same guard before you tag.
- ADR-0011 records that distribution is a git tag and there is no Galaxy publish, along with the three things that would
  reopen it. The argument that publishing was needed for a downstream collection to depend on this one is false and the
  ADR says so: git dependencies resolve transitively, tested.

## 1.0.0

Initial extraction from the consumer repo these roles grew in, so a second team can take the same baseline without
taking the fleet it was written for.

### Roles

- `os` — apt baseline, swap before the upgrade that would OOM without it, hostname/FQDN, timezone, unattended-upgrades
  with a per-host reboot minute.
- `ufw` — declared ingress and an SSH rate-limit opened **before** the deny-default flip, so a converge cannot lock the
  host out.
- `fail2ban` — sshd and recidive jails, banning through ufw.
- `docker` — engine, compose v2, an optional registry login, and a weekly prune timer scheduled clear of the backup
  window.
- `rclone` — pinned upstream `.deb` verified by checksum, plus an R2 remote rendered only when its credentials exist.
  The `r2crypt` wrapper is both-passwords-or-neither, asserted and enforced at render.
- `caddy` — the TLS edge from the official apt repo: route schema, Cloudflare origin certs, derived proxy trust, and URI
  redaction on both loggers.
- `monitoring` — Beszel hub and agent plus Dozzle, loopback-bound, reached over an SSH tunnel. Asserts an admin
  identity, since an admin-less hub answers unauthenticated, and asserts the agent's key whenever its token is set.

### Playbooks

- `baseline` — the five baseline roles in their load-bearing order; consumers `import_playbook` it.
- `bootstrap` — one-time creation of the deploy user on a fresh box. Addressed by `-e target=<host>`, not `-l`.
- `update` — serial apt upgrade with a reboot when required.

### Gates

Static only — this repo owns no inventory and reaches no host, so the dry-run and second-converge legs stay the
consumer's ([ADR-0003](docs/adr/0003-the-gate-is-make-check-and-a-second-converge.md)).

- `make check` — formatting, playbook syntax, `ansible-lint` at the production profile, `shellcheck`, collection build.
  The pre-commit hook runs it, so every commit leaves it green.
- `make sanity` — `ansible-test sanity`. CI only; a cold run builds a venv per supported Python.
- `make test` — golden renders. Fixtures go through the real templates and the bytes are diffed against checked-in
  expectations. It catches what the other two structurally cannot: a config that is _valid_ and says the wrong thing —
  proxy trust on a host nothing fronts, a body cap clamped by a matcher-less default, an agent rendered with an empty
  token.

### Licensing and identity

- **Apache-2.0.** The pre-release tree said "All rights reserved".
- **The namespace is `fabbrito`**, matching the GitHub account that owns the repo — Galaxy grants namespaces from that
  identity, so a mismatch would have blocked any future publish. Every FQCN is `fabbrito.infra.*`.

### Installing

The consuming repo must declare `collections_path` in its `ansible.cfg` before installing. An install into a path
Ansible does not search leaves every `fabbrito.infra.*` FQCN unresolvable. See README, "Install".

The repo is public and installs over `https`, so no deploy key is needed — which is what makes it fetchable from a CI
runner. `git+ssh` still works.
