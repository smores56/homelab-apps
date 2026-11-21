# homelab-apps

Typed application definitions for the `apps` tenant on `smortress`. The same
repository is the one place an agent may merge without a human: containment is
structural, not reviewed.

This repository contains no Quadlets, unit files, scripts, or anything else
smortress executes. It contains `apps/<name>/app.toml` and, when an app needs
secrets, `secrets.sops.json`.

## A definition

```toml
image = "docker.io/traefik/whoami@sha256:c4717a…"  # pinned by digest, no tag
port = 80                                          # the container's port
egress = "none"                                    # or "internet"
quota = "1G"                                       # optional, 50G ceiling
secrets = ["db-password"]                          # optional, own section only
env = { TZ = "UTC" }                               # optional
```

That is the whole vocabulary. There is deliberately no field for a host path, an
extra mount, a device, a capability, privileged mode, a socket, a host or custom
network, a published port, a user or an SELinux option, and unknown keys are
errors. Changing what an app may do is a `homelab-platform` change — the schema,
the renderer, the network policy and the tenant account all live there.

## The check

`apps-schema` runs `scripts/apps-schema.py` from `homelab-platform` at the SHA
pinned in `.github/workflows/ci.yml`, never a vendored copy: a copy an agent
could edit would make the check agree with whatever it checked. Both
repositories are public, so the checkout is anonymous: the workflow holds no
secret, and a PR that edits it has nothing to exfiltrate.

The validator reads `secrets.sops.json` still encrypted — SOPS encrypts values only,
so key names are what it checks and no identity is needed in CI.

The check renders rather than only validating: the host renders before it
applies, so a tree that validates but cannot render must fail in CI, not on
smortress with the previous revision still running. The smortress deploy still
does not trust this check — it re-runs the same validator from its own image
against the fetched commit before rendering.

## Secrets

`secrets.sops.json` is a SOPS file over the platform recipients in `.sops.yaml`:
the operator key, the shared machine identity and the three YubiKeys. Values are
encrypted; only the structure is plaintext.

```json
{ "apps": { "smoke": { "db-password": "…" } } }
```

A definition may reference only secrets declared under its own `apps.<name>`
section. The deploy creates each referenced value as a Podman secret in the
tenant, named `app-<name>-<secret>`.

The design originally called for a tenant-only age recipient whose identity
decrypts nothing else. It is not used: the apply step runs as root and already
holds the machine identity, so a second key would add a secret to protect
without narrowing any path. Recorded as a delta on SDLC #726.

## Merging

`main` requires a PR, the `apps-schema` check and code-owner review where
`CODEOWNERS` names an owner — which is only `.github/**`, `CODEOWNERS` itself and
`.sops.yaml`, the file that decides who can decrypt the tenant's secrets. An
app-only PR therefore needs no human approval and auto-merges once the check
passes; a PR that edits the check or the recipient list cannot.

Signed commits are not required: the agent merges unattended, so a signature
would attest to a key on the machine, not to human approval. What is still
missing is the broker's ability to open a PR here — its GitHub App holds
`contents: write` for this repository but no `pull_requests: write` — so until
that is granted the agent cannot open the PR the rule requires.
