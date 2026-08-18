# Prime 100 releases

Hosts Prime 100 OS desktop app releases (auto-updater `latest.json`) and the whisper voice-dictation assets fetched by the companion on first run.

## Branches

`staging` is the integration branch; `main` is production. Cut every `feature/*` from
`staging`, PR it back into `staging`, then promote `staging` → `main`. Both branches allow
**merge commits only** and both require a pull request — direct pushes, force-pushes, squash
merges and branch deletion are rejected by ruleset.

One-time setup in your clone:

```sh
git config core.hooksPath .githooks
```

Full details, including what is and is not actually enforced: [docs/BRANCHING.md](docs/BRANCHING.md).
