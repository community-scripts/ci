# ci

Shared CI for [ProxmoxVE](https://github.com/community-scripts/ProxmoxVE) and
[DevScripts](https://github.com/community-scripts/DevScripts).

## What is here

| Path | What |
|---|---|
| `.github/workflows/pr-script-test.yml` | Reusable workflow: builds the `ct/` and `vm/` scripts a PR touches on the Proxmox and Incus lab hosts (and ARM64), reports on the PR |
| `.github/workflows/arm-script-test.yml` | Reusable workflow: one `ct/` script on GitHub's arm64 runner, by hand |
| `.github/workflows/screenshots-cleanup.yml` | Weekly: deletes the screenshots of closed PRs and issues |
| `.github/workflows/update-script-timestamp.yml` | Reusable workflow: stamps the PocketBase record of scripts changed on main (`mode: main` for ProxmoxVE, `dev` for DevScripts) |
| `.github/workflows/pr-test-command.yml` | Reusable workflow: posts a copy-paste test command on PRs |
| `scripts/` | The harness the test jobs run (`pve-run-script.sh`, `pve-run-vm.sh`, `arm-runner-setup.sh`) |
| `scripts/host/` | Setup of the lab: test hosts (`pve-ci-host-setup.sh`, `incus-ci-host-setup.sh`) and the single-use controller on prox1 (`ci-controller-setup.sh`, `ci-vm-controller.sh`, `gh-dispatch.sh`). Copied to the hosts by hand, never run by a workflow |
| `screenshots/` | Pictures for the PR comments, as `screenshots/<repository>/<pr\|issue>-<number>/`, written by the report job |

## How the repositories use it

Each repository keeps a small `pr-script-test.yml` with the triggers
(`pull_request_target`, `/test` comments, the "Ready For Testing" label,
`workflow_dispatch`) and the per-PR concurrency queue, and calls:

```yaml
uses: community-scripts/ci/.github/workflows/pr-script-test.yml@main
```

A called workflow runs in the caller's context: its `github.token`, its
self-hosted runners (`proxmox-test`, `incus-test`), its PRs and checks.
`github.repository` decides which URLs get rewritten and which container ID
range is used, as before.

## `/test` options

A `/test` comment (write access needed) can be followed by, in any order:

- a full 40-character SHA: build exactly that commit, while it is the PR head
- app names or `ct/<name>.sh` / `vm/<name>.sh` paths: build only those
- platforms `pve`, `incus`, `arm`: build only there
- `var_<name>=<value>` settings for the app, e.g.
  `/test n8n var_n8n_image_tag=3.3.0 var_force_breaking_update=yes`. They are
  set for the install and for both updates (the engine carries `var_*` into the
  container on Proxmox and Incus). Names are lower case; values use letters,
  digits and `. _ : / @ + , = -`. What the test decides itself (`var_ctid`,
  `var_hostname`, `var_os`, `var_arm64`, storage, network, ...) is refused.
  `vm/` builds ignore them.

A run on only some platforms or with settings gets its own check name, never
approves a PR and never lifts an earlier "changes requested".

**A change on `main` here goes live in both repositories at once.** Whoever
can push here can change what runs with their tokens, so this repository
needs the same people and the same branch protection as they have.

## Screenshots

The report job writes them with a GitHub App token limited to this
repository (`contents: write`). Each caller needs:

- variable `GHAPP_SCREENSHOTS_CLIENT_ID`: the App's client ID (`Iv23…`)
- secret `GHAPP_SCREENSHOTS_PRIVATE_KEY`: its private key

(organization-level works for both). Without them the report still goes
out, just without pictures. The test jobs never see the token.

They are committed straight to `main`, so pull before you push here. The
weekly cleanup deletes the folders of closed PRs and issues (their pictures
disappear from the old comments then); the history keeps them, `main` is never
force-pushed. If `main` gets a ruleset, the App and `github-actions[bot]` need
to be allowed to bypass it.
