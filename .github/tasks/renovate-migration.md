# Shared Renovate preset and migration of all repositories

## Goal

One preset in `pravbeseda/renovate-config` that every repository extends, behaving like the
current Dependabot setup in `sleep-noise`:

- one run a week, Saturday 00:00–06:00 Moscow time;
- all minor and patch updates of one package manager in one pull request;
- every major in a pull request of its own, opened only when approved in the Dependency Dashboard;
- no automerge;
- a release is offered only once it is 7 days old;
- security fixes ignore the schedule (Renovate's `vulnerabilityAlerts`, on by default).

Dependabot version updates are removed wherever the preset takes over.

## Current state (2026-10-08)

| Repository | Branch | Dependabot | Renovate | Managers Renovate detects |
| --- | --- | --- | --- | --- |
| sleep-noise | main | `dependabot.yml`: gradle + actions, weekly, grouped; detekt major ignored | onboarding #146 | github-actions, gradle, gradle-wrapper |
| drevo-web | main | `dependabot.yml`: npm (2 dirs) + actions, **monthly**, many groups and ignores | onboarding #431 | github-actions, npm (2), nvm, pip_requirements |
| debt-islands | main | `dependabot.yml`: pub (workspace root) + actions, weekly, grouped | onboarding #63 | github-actions, pub (3), **gradle + gradle-wrapper in `app/android`** |
| invest-ng | master | no config; security PRs #1, #2 open | onboarding #3 | npm |
| drevo-yii | main | none | onboarding #317 | composer, github-actions, npm (incl. vendored `external/jquery-ui`) |
| lab-google-auth | main | none | onboarding #2 | npm |
| ansible-hosts | main | none | active, `config:recommended`, 7 open PRs (#144–#154) | ansible-galaxy, pip, github-actions |
| CurrencyEditText | main | none | active, own `renovate.json5`, **patch automerge** | gradle |
| drevo-app | main | none | active, own `renovate.json5`, **monthly, patch automerge** | gradle, custom regex |
| SpendControl | main | none | active, own `renovate.json5`, **patch automerge** | gradle, custom regex |

Added 2026-10-09 (step 8); invest-ng and lab-google-auth are archived since and need nothing:

| Repository | Branch | Dependabot | Renovate | Managers |
| --- | --- | --- | --- | --- |
| codex-usage | main | security updates on | onboarding #1, extends `local>pravbeseda/renovate-config` | github-actions, npm |
| monitor | main | security updates on | onboarding #70, extends `local>pravbeseda/renovate-config` | github-actions, gomod |
| telegram-molko-bot | main | security updates on; PR #1 (esbuild) open since 2026-06-14 | onboarding #2, extends `local>pravbeseda/renovate-config` | npm |
| kalugaman | main | security updates on; PR #45 (postcss) open since 2026-08-04 | no onboarding PR yet | github-actions, npm |
| ali-agent-kit | main | security updates on | no onboarding PR yet | github-actions, npm |
| home (private) | main | security updates on | no onboarding PR yet | npm |
| antispam | main | security updates on | no onboarding PR yet | github-actions |

## Preset design

`default.json` at the root of `pravbeseda/renovate-config`, referenced as
`github>pravbeseda/renovate-config`. Renovate resolves that reference to `default.json` only, so the
preset is plain JSON and explains itself in its `description`. In JSON5 notation:

```json5
{
  $schema: 'https://docs.renovatebot.com/renovate-schema.json',
  extends: [
    'config:recommended',
    ':enableVulnerabilityAlertsWithLabel(security)',
  ],
  schedule: ['* 0-5 * * 6'],
  timezone: 'Europe/Moscow',
  minimumReleaseAge: '7 days',
  labels: ['dependencies'],
  packageRules: [
    {
      matchUpdateTypes: ['minor', 'patch', 'digest'],
      groupName: '{{manager}} non-major',
    },
    {
      matchManagers: ['gradle-wrapper'],
      matchUpdateTypes: ['minor', 'patch'],
      groupName: 'gradle non-major',
    },
    {
      matchUpdateTypes: ['major'],
      dependencyDashboardApproval: true,
    },
  ],
}
```

- A major gets no branch, PR or CI run until its checkbox in the Dependency Dashboard issue is
  ticked. Security fixes are not held back: `vulnerabilityAlerts` does not require approval.
- `minimumReleaseAge: '7 days'` is the supply-chain guard: Renovate PRs, unlike Dependabot's, run
  CI with the repository's secrets, and a compromised release is usually pulled within days.
  Security fixes bypass it. Worst-case lag on the weekly schedule: 14 days.
- One non-major PR per manager (`groupName` accepts templates): a red Gradle or npm bump does not
  hold back the actions bump. The Gradle wrapper joins the `gradle` PR, as in Dependabot.
- The group rules come after `config:recommended`, so they override the monorepo groups for minor
  and patch; majors keep the monorepo grouping (e.g. all `@angular/*` majors in one PR).
- `automerge` stays at Renovate's default `false`; the preset does not mention it.
- Repository-specific rules live in each repository's `renovate.json5` after
  `extends: ['github>pravbeseda/renovate-config']`.
- After migration, in every repository: **Dependabot alerts on** (Renovate reads them),
  **Dependabot security updates off** (otherwise two PRs per advisory).

## Steps

Each repository is one branch `chore/renovate-preset` and one PR. Commit, push, PR creation and merge
each need explicit permission.

### 1. Preset repository

1. Create `pravbeseda/renovate-config` on GitHub as a public repository.
2. The local repo has no commits, so a PR has no base: bootstrap `main` with an empty initial commit
   (an explicit exception to the no-commit-to-main rule, to be approved), then the rest via a PR.
3. PR content: `default.json`, `AGENTS.md` + `CLAUDE.md` (`@AGENTS.md`), a CI workflow running
   `renovate-config-validator --strict --no-global` on both configs, and `renovate.json` extending
   the preset itself so the repo keeps its own actions and the pinned validator current.

Verify: CI green; the validator rejects a deliberately broken copy.

### 2. Pilot: ansible-hosts

1. `renovate.json`: replace `config:recommended` with `github>pravbeseda/renovate-config`.
2. After merge, on the next Saturday run: the minor/patch PRs (#144, #145, #148, #149, #151) are
   autoclosed and replaced by one `<manager> non-major` PR per manager (releases younger than 7 days
   wait for a later run); the majors #152 and #154 are
   listed under "Pending Approval" in the Dependency Dashboard. Close the two PRs if Renovate leaves
   them open.

Verify: Dependency Dashboard issue lists exactly that; no automerge happened.

### 3. sleep-noise (reference behaviour)

1. `renovate.json5` with overrides:
   - detekt major disabled:
     `{ matchPackageNames: ['/^io\\.gitlab\\.arturbosch\\.detekt/'], matchUpdateTypes: ['major'], enabled: false }`
     (regex because Gradle plugins are matched by their marker artifact);
   - commit types as today: `build` for `gradle` and `gradle-wrapper`, `ci` for `github-actions`
     (`semanticCommitType` per `matchManagers`).
2. Delete `.github/dependabot.yml`.
3. Update `AGENTS.md:171` and `AGENTS.md:193` (Dependabot → Renovate, "lift a pin in both places").
4. `.github/actions/google-services/action.yml:36–52`: comments and messages name Dependabot as a
   case without secrets. Renovate PRs come from same-repo branches and get the real secrets; reword
   so only forks fall back to the stub.
5. Close onboarding #146 if Renovate does not close it itself.

Expected: the `java-jdk` 25 major waits in the Dependency Dashboard.

### 4. debt-islands

1. `renovate.json5` with the preset and
   `{ matchManagers: ['gradle', 'gradle-wrapper'], enabled: false }`: the Android toolchain in
   `app/android` follows Flutter's supported range and moves with the manual Flutter upgrade.
2. Delete `.github/dependabot.yml`; update `AGENTS.md:150`.
3. Close onboarding #63.

Verify: the first grouped PR updates the root `pubspec.lock` (pub workspace) and CI passes. If
Renovate cannot resolve the workspace, stop and decide before going further.

### 5. drevo-web

1. `renovate.json5` with the preset plus:
   - `rangeStrategy: 'update-lockfile'` (equivalent of `increase-if-necessary`);
   - disabled: minor and major of `@angular/**`, `@angular-devkit/**`, `@schematics/angular`,
     `@angular-eslint/**`, `angular-eslint`, `@nx/**`, `nx` (they go through `nx migrate`);
     every update of `typescript`; majors of `@types/node`; majors of the `nvm` manager
     (`.nvmrc` follows the `engines` floor like `@types/node`);
   - `codemirror` group for majors of `@codemirror/**` (minor/patch already share one PR).
   - Dropped as unnecessary: `shared-lib-dependencies` (Renovate updates a dependency in both
     `package.json` files in one branch), separate `angular`/`nx` patch groups (the one non-major PR
     keeps them in lockstep).
2. Delete `.github/dependabot.yml`, `scripts/check-dependabot-groups.js`,
   `scripts/check-dependabot-groups.test.js`; remove the script from `lint:workflows` in
   `package.json:22`; update the comment in `scripts/check-node-types.js:5–7`, `AGENTS.md:76`,
   `.github/copilot-instructions.md:78`. Leave `.github/tasks/nx-migrate-23.2.md` as history.
3. Close onboarding #431.

Verify: `yarn lint:workflows` and `yarn test:scripts` pass; the Dependency Dashboard shows no
Angular/Nx minor, no TypeScript, no `@types/node` major; the `@angular/router` security PR from the
onboarding preview still opens (vulnerability PRs must survive `enabled: false`; if they do not,
narrow the disable rules).

### 6. invest-ng, drevo-yii, lab-google-auth

invest-ng and lab-google-auth were archived before their turn: skip their items.

1. `renovate.json5` extending the preset (for invest-ng the base branch `master` is picked up
   automatically).
2. drevo-yii: `{ matchFileNames: ['external/**'], enabled: false }` — vendored code is checked by
   `vendored-audit.yml`, not updated by a bot.
3. invest-ng: close Dependabot #1 and #2 once Renovate's security PRs for the same packages exist.
   Many majors (Angular 14 → current) wait in the Dependency Dashboard.
4. Close onboarding #3, #317, #2.

### 7. CurrencyEditText, drevo-app, SpendControl

Each `renovate.json5` extends the preset and drops `config:recommended`, the vulnerability-alert
presets, `labels`, `platformAutomerge`, every `automerge` rule, and the groups that only existed to
separate automerged patches from the rest (androidx, firebase, quality gates, toolchain, ci tooling,
ci scanners). CurrencyEditText also drops `minimumReleaseAge: '3 days'`, which would lower the
preset's 7 days. Kept as local overrides:

- **drevo-app:** `schedule:monthly` (private repo, billed Actions minutes, every merge ships a QA APK),
  `osvVulnerabilityAlerts`, the `com.autonomousapps` ceiling, the `billing` group with
  `needs-manual-testing`, the gitleaks `prBodyNotes`, all three custom managers.
- **SpendControl:** the `com.autonomousapps` and `com.github.triplet` ceilings, the `kotlin and ksp`
  group (the two must move together), the `billing` group with `needs-manual-testing`, both custom
  managers.

Update the comments and any docs that describe automerge (check `AGENTS.md` / `CLAUDE.md` in each).

Verify: Dependency Dashboard of each repo shows one non-major group per manager, separate `billing` and
`kotlin and ksp` groups, and majors under "Pending Approval".

### 8. Repositories outside the first wave

Onboarding for the preset's own owner already proposes `extends: ['local>pravbeseda/renovate-config']`,
the same preset as `github>`, so these repositories need no PR of their own.

1. Wait for the onboarding PRs of kalugaman, ali-agent-kit, home, antispam: the app already has
   every repository, and onboarding has reached a few more of them each day since 2026-10-07. If one is
   still missing on 2026-10-12, read that repository's job log in the Mend developer portal.
2. Check that each onboarding PR extends only `local>pravbeseda/renovate-config`, then merge it:
   codex-usage #1, monitor #70, telegram-molko-bot #2, and the four new ones.
3. Close Dependabot PRs telegram-molko-bot #1 and kalugaman #45 once Renovate's PRs for the same
   packages exist.
4. Update `AGENTS.md` here: repositories reference the preset as `github>` or `local>`.

Verify: each repository has a Dependency Dashboard listing one non-major group per manager.

### 9. Close-out

- Dependabot security updates off everywhere; on 2026-10-09 still on in drevo-web, ansible-hosts,
  drevo-app, SpendControl and every repository of step 8.
- Every repository not archived: one Dependency Dashboard issue, no open onboarding PR, no `dependabot.yml`.

## Out of scope

- Lock file maintenance, custom managers for versions in scripts (gitleaks,
  actionlint, ktlint in sleep-noise) — not part of today's Dependabot behaviour.
- Repositories with no dependencies: garmin-watchface-955, MakeDrevoDB, memory, molkobot.

## Decisions

0. **PRs immediately or on Dependency Dashboard approval?**
   **Decision:** minor and patch arrive as PRs once a week; majors only after approval in the
   Dependency Dashboard. A major almost always needs a manual migration, so an unrequested PR that
   cannot go green is noise and burns CI minutes; approving everything instead would let updates pile up.
1. **Repositories with their own config and patch automerge** (CurrencyEditText, drevo-app,
   SpendControl): move them to the preset and drop automerge, move them but keep their
   automerge and groups as local overrides, or leave them out?
   **Decision:** move all three to the preset and drop automerge; keep only the overrides with a
   reason of their own (step 7). Keeping patch automerge would mean splitting the preset's
   non-major groups again locally.
2. **One non-major PR across all managers or one per manager?** Dependabot in sleep-noise opens one
   for Gradle and one for actions.
   **Decision:** one per manager. A breaking Gradle or npm bump must not hold back the actions bump;
   the cost is a rebase or two a week under strict branch protection.
3. **Preset repository public or private?** The Mend app reads both; public needs no access setup
   and contains nothing secret.
   **Decision:** public. Nothing in it is secret, the `extends` in public repositories stays readable,
   and resolution does not depend on which repositories the app is granted.
4. **drevo-web schedule:** switch from monthly to the preset's weekly, or keep monthly as an override?
   **Decision:** weekly, no override. Actions minutes are free in this public repository, and a
   week of npm bumps is easier to bisect than a month.
5. **Secrets in Renovate PRs:** Dependabot PRs ran without secrets (sleep-noise falls back to a stub
   `google-services.json`); Renovate PRs get them. Accept, or keep secret-using jobs off
   Renovate branches?
   **Decision:** accept, and add `minimumReleaseAge: '7 days'` to the preset. Gating secrets in
   every workflow would protect only until the merge, after which `main` builds with secrets anyway;
   the release age guards both, and security fixes bypass it.
6. **debt-islands Android Gradle files** (`app/android`): Dependabot never touched them, and AGP/Kotlin
   there track the Flutter template. Enable under the preset or disable the `gradle` and
   `gradle-wrapper` managers?
   **Decision:** disable both managers in debt-islands. AGP, Gradle and Kotlin there are part of a
   Flutter upgrade, which stays manual (`AGENTS.md:150`), as under Dependabot.
7. **When does the weekly run happen?**
   **Decision:** Saturday 00:00–06:00 Moscow time (`schedule: ['* 0-5 * * 6']`,
   `timezone: 'Europe/Moscow'`). The Mend app does not run at exact times, so the window is several
   hours wide; without `timezone` the schedule would be read as UTC.
