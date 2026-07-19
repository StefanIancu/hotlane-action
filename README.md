# hotlane-action

Deploy with [hotlane](https://github.com/StefanIancu/hotlane) from GitHub Actions: install the binary, run one command, get the verify verdict as job outputs and a summary. A push goes git delta → verified running fork → traffic flip, typically in ~1-2s on the daemon side.

## Deploy on every push to main

```yaml
name: deploy
on:
  push:
    branches: [main]

concurrency: deploy-production   # one deploy at a time

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0                 # the diff base must exist locally
      - uses: StefanIancu/hotlane-action@v1
        with:
          daemon: ${{ vars.HOTLANE_DAEMON }}
          token: ${{ secrets.HOTLANE_TOKEN }}
```

A rejected push fails the job with the fork's verify results and dying logs in the step output; a promoted one writes the timings and verify table to the job summary.

## Manual rollback

```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: version to roll back to (empty = previous)
        required: false
jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - uses: StefanIancu/hotlane-action@v1
        with:
          daemon: ${{ vars.HOTLANE_DAEMON }}
          token: ${{ secrets.HOTLANE_TOKEN }}
          command: rollback
          args: ${{ inputs.version }}
```

## Scheduled drift check

`hotlane drift` exits non-zero when the live container's behavior has diverged from the clean from-source build - a red scheduled run is your alarm:

```yaml
on:
  schedule: [{ cron: "0 6 * * *" }]
jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: StefanIancu/hotlane-action@v1
        with:
          daemon: ${{ vars.HOTLANE_DAEMON }}
          token: ${{ secrets.HOTLANE_TOKEN }}
          command: drift
```

## Inputs

| input | default | |
|---|---|---|
| `daemon` | - | daemon URL; required for everything except `version` |
| `token` | - | bearer token for the daemon API |
| `command` | `push` | `push`, `test`, `promote`, `discard`, `rollback`, `drift`, `status`, `version` |
| `args` | - | extra arguments (e.g. the version for `promote`/`rollback`) |
| `version` | `latest` | hotlane release to install, e.g. `v0.4.2` |
| `working-directory` | `.` | app checkout `push` computes its git delta from |

## Outputs

| output | |
|---|---|
| `version` | app version produced/acted on |
| `promoted` | `"true"` when a push was verified and promoted |
| `json` | the command's full `-json` response |

## Notes

- `push` needs the repository's git history: check out with `fetch-depth: 0`.
- The daemon runs on your own box ([where hotlane runs](https://github.com/StefanIancu/hotlane/blob/main/docs/ci.md#where-does-hotlane-run-and-not-run)); this action is just the client. If the daemon is on a private network, add your tunnel/VPN step before this one and point `daemon` accordingly.
- Full integration guide: [docs/ci.md](https://github.com/StefanIancu/hotlane/blob/main/docs/ci.md).
