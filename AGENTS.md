# renovate-config

The shared Renovate preset of the pravbeseda repositories. They reference it as
`github>pravbeseda/renovate-config`, or as `local>pravbeseda/renovate-config` where Renovate's
onboarding PR wrote the config, so every change to `default.json` reaches all of them on their next
Renovate run.

- `default.json` holds only the behaviour common to every repository. A rule that one repository
  needs lives in that repository's own Renovate config, after its `extends`.
- The preset is plain JSON: Renovate resolves either reference to `default.json` and never to
  `default.json5`. The reasoning goes into the `description` array, which Renovate shows in the
  onboarding summary.
- `.github/workflows/ci.yml` validates both configs; run its last command locally (Node 24) before
  pushing. `--no-global` matters: without it the validator checks a file passed by name as global
  config and accepts what a repository config may not contain.
- Why the preset behaves as it does, and how each repository was moved onto it:
  `.github/tasks/renovate-migration.md`.
