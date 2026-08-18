<!-- Base should be `staging` unless this is a staging -> main promotion. See docs/BRANCHING.md. -->

## What changed

<!-- One or two sentences. Link the doc/issue if there is one. -->

## How it was tested

<!-- Commands run, plus anything checked by hand. CI runs `guard-promotion` only — this repo
     has no packages and no test suites, so nothing else is checked automatically. -->

## Checklist

- [ ] Base branch is correct (`staging` for features, `main` only for a promotion from `staging`)
- [ ] CI is green
- [ ] Approved by someone other than the author

<!-- Promotions only (staging -> main): merge with a MERGE COMMIT, not squash. -->
