# AGENTS.md

Guidance for agents working in `homelab-apps`. This repository is the agent's
deploy path into the `apps` tenant on `smortress`: a change here merges without
human review and is what the tenant runs.

## What may live here

- `apps/<name>/app.toml` — one typed definition per app.
- `secrets.sops.json` — the tenant SOPS file, values encrypted.
- `README.md`, `AGENTS.md`, and `.github/`-owned files (which need a human).

Never add a Quadlet, unit file, script, compose file, or manifest of any kind.
Nothing here is executed by the host; the platform renders definitions into
rootless Podman units.

## Classification

An app or secret change is a tenant change: no SDLC funnel, no worktree. Push a
branch, let `apps-schema` pass, enable auto-merge, and let it land — the merge is
the rollout record. A change to `.github/**` or `CODEOWNERS` is a platform
change, needs `@smores56`, and should not be attempted autonomously.

## The vocabulary is closed

`image`, `port`, `egress`, `quota`, `secrets`, `env`. Unknown keys are errors,
at every level, and so are host paths, extra mounts, devices, capabilities,
privileged mode, sockets, custom or host networks, published ports, users and
SELinux options. There is no field to widen containment, and adding one is a
`homelab-platform` change.

- `image` must be a repository path with `@sha256:<64 hex>` and no tag. Resolve
  a digest with the registry API, not by guessing.
- `port` is the container's own port; the platform assigns a loopback port and a
  Serve path `/apps/<name>`. An app that cannot run under a subpath is not
  deployable yet.
- `egress = "internet"` gets pasta networking behind the tenant's deny-first
  nftables table; `egress = "none"` gets the internal, routeless network.
- `secrets` may name only entries under its own `apps.<name>` section of
  `secrets.sops.json`.

## Adding an app

1. `mkdir apps/<name>` and write `app.toml`. Lower-case letters, digits and
   hyphens; the name becomes the Serve path, the data directory `/srv/apps/<name>`
   and the unit name.
2. Give it a `quota` if it stores data. It writes only under `/data`, which is
   that directory, mounted `:Z` into its own SELinux MCS pair.
3. Run the same check CI runs before pushing:

   ```sh
   python3 ../homelab-platform/main/scripts/apps-schema.py render \
     --root . --secrets secrets.sops.json --out /tmp/apps-render
   ```

4. Push a branch, open a PR, and enable auto-merge. That path needs the broker's
   GitHub App to hold `pull_requests: write` here, which it does not yet, and
   the PR rule forbids a direct push to `main`: until that permission lands,
   ask the operator to merge.

## Secrets

Edit `secrets.sops.json` with `sops` over the recipients in `.sops.yaml`; if you hold
no identity, ask the operator for the ciphertext rather than inventing one. The
file stays encrypted at every step, including in CI.

`.sops.yaml` is generated from the canonical policy in `dotfiles`:

```sh
python3 <dotfiles>/scripts/sops-policy.py render --profile apps --out .sops.yaml
```

The keyless recipient check runs from the pinned platform checkout once this
repository's platform pin includes it. Do not vendor a checker here: a copy an
agent could edit would agree with whatever it checked.

## When a check fails

`apps-schema` prints `apps/<name>/app.toml: <problem>`. Fix the definition; do
not edit the workflow, the schema in `homelab-platform`, or `CODEOWNERS`.
