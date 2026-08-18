# Prime 100 releases

Hosts Prime 100 OS desktop app releases (auto-updater `latest.json`) and the whisper voice-dictation assets fetched by the companion on first run.

## Branches

`staging` is the integration branch; `main` is production. Cut every `feature/*` from
`staging`, PR it back into `staging` (squash), then promote `staging` → `main` with a merge
commit. Never push straight to either.

One-time setup in your clone:

```sh
git config core.hooksPath .githooks
```

Full details, including what is and is not actually enforced: [docs/BRANCHING.md](docs/BRANCHING.md).
