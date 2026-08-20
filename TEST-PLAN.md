# Monorepo integration test plan

This test plan covers all the major features from the monorepo implementation. Each phase is a separate push to `main`, and we go through them one by one.

## Prerequisite

Before any of these tests work, the canary build of `Pandoriux/zephyr-release` must contain the latest fixes (especially the working branch name resolution fix).

1. Push all pending fixes to `zephyr-release` main.
2. Wait for the `Update Canary Branch` workflow to succeed.
3. Verify at https://github.com/Pandoriux/zephyr-release/actions

## Repo structure

```
test-monorepo/
├── .github/workflows/zephyr-release.yml
├── packages/
│   ├── core/
│   │   ├── package.json    (version: 0.0.0)
│   │   └── index.ts
│   └── cli/
│       ├── package.json    (version: 0.0.0)
│       └── index.ts
├── package.json            (root, private, no version tracking)
└── zephyr-release-config.json
```

---

## Phase 1: Initial setup + both workspaces affected

**Goal:** Verify basic monorepo detection, path-based filtering, and that both workspaces get version bumps, tags, and releases.

### What I (agent) do
- [x] Create the repo structure above with two workspaces.
- [x] Create `zephyr-release-config.json` with `review` mode (default) and two workspaces.
- [x] Create `.github/workflows/zephyr-release.yml` using `@canary`.
- [x] Commit: `feat: set up monorepo with core and cli workspaces`

### What you verify
After pushing to main:
- [ ] CI runs without errors.
- [ ] Working branch name is `zephyr-release/main` (not `zephyr-release/` with trailing slash).
- [ ] Both `core` and `cli` are detected as affected workspaces.
- [ ] A PR is created with changes for both workspaces.
- [ ] PR title mentions both workspace releases (e.g. `chore: release core-v0.1.0, cli-v0.1.0`).
- [ ] Each workspace's `package.json` version is bumped to `0.1.0` in the PR diff.
- [ ] Each workspace gets a `CHANGELOG.md` generated inside its own directory.

After merging the PR:
- [ ] Two tags created: `core-v0.1.0` and `cli-v0.1.0`.
- [ ] Two GitHub Releases created with correct release notes.

---

## Phase 2: Single workspace affected (path-based filtering)

**Goal:** Verify that commits touching only one workspace's files only bump that workspace.

### What I do
- [ ] Commit: `feat(core): add subtract function` (only changes `packages/core/index.ts`)

### What you verify
After pushing:
- [ ] Only `core` detected as affected.
- [ ] `cli` is skipped (no version bump, no tag, no release entry).
- [ ] PR only contains changes for `core` (version bump to `0.2.0`).
- [ ] After merge: only `core-v0.2.0` tag and release created. `cli` stays at `0.1.0`.

---

## Phase 3: Cross-workspace commit

**Goal:** Verify a commit touching files in both workspaces bumps both independently.

### What I do
- [ ] Commit: `fix: update shared logic` (changes files in both `packages/core/` and `packages/cli/`)

### What you verify
- [ ] Both workspaces detected as affected.
- [ ] `core` bumps to `0.2.1` (patch for `fix`).
- [ ] `cli` bumps to `0.1.1` (patch for `fix`).
- [ ] Single PR with changes for both.

---

## Phase 4: `Release-As` footer (forced version)

**Goal:** Test the monorepo-specific `Release-As: name@version` syntax.

### What I do
- [ ] Commit with footer:
  ```
  feat: major rewrite

  Release-As: core@2.0.0
  ```
  Only touches `packages/core/`.

### What you verify
- [ ] `core` version is forced to `2.0.0`, ignoring normal bump rules.
- [ ] `cli` is not affected (commit only touches core).
- [ ] Tag created: `core-v2.0.0`.

---

## Phase 5: Stdout config override

**Goal:** Test that a command hook can print a config override to stdout and it gets applied.

### What I do
- [ ] Add a `pre-calculate-version` hook that prints a bump strategy override between the `ZR_CONFIG_OVERRIDE_START` / `ZR_CONFIG_OVERRIDE_END` markers.
- [ ] Commit: `fix: minor change in core` (touches only `packages/core/`)

### What you verify
- [ ] The hook runs and the override is captured from stdout.
- [ ] The bump strategy change takes effect (e.g. forcing a minor bump instead of patch).
- [ ] CI logs show `Resolve runtime config override (preCalculateVersion)` with the applied override content.

---

## Phase 6: Per-workspace config overrides

**Goal:** Verify that workspace-level config overrides work (custom tag template, custom changelog path).

### What I do
- [ ] Update `zephyr-release-config.json` to give `cli` a custom tag template (`cli/v{{ nextVersion }}`).
- [ ] Commit: `feat: add cli feature` (touches `packages/cli/`)

### What you verify
- [ ] `cli` tag uses the custom template: `cli/v0.2.0` (slash separator, not dash).
- [ ] `core` still uses default: `core-v2.1.0` (dash separator).
- [ ] Both changelogs are still written to the correct workspace directories.

---

## Phase 7: Environment variables and outputs

**Goal:** Verify that per-workspace and global env vars are correctly set.

### What I do
- [ ] Add a `post-release` hook that prints env vars: `ZR_NAME`, `ZR_NEXT_VERSION`, `ZR__core__NEXT_VERSION`, `ZR__cli__NEXT_VERSION`, `ZR_WORKSPACES`.

### What you verify (from CI logs)
- [ ] During `core`'s `postRelease`: `ZR_NAME=core`, `ZR_NEXT_VERSION=<core version>`.
- [ ] During `cli`'s `postRelease`: `ZR_NAME=cli`, `ZR_NEXT_VERSION=<cli version>`.
- [ ] `ZR__core__NEXT_VERSION` and `ZR__cli__NEXT_VERSION` are set in both hooks.
- [ ] `ZR_WORKSPACES` contains a JSON array with both workspace entries.

---

## Edge cases to test later

These can be tested ad-hoc once the main phases pass:

- **Global `Release-As: 3.0.0`** (without `@`): should force all affected workspaces to that version.
- **Commit with no allowed type** (e.g. `chore: cleanup`): should still pick up older unreleased `feat`/`fix` commits for affected workspaces.
- **Empty workspace** (commit touches no workspace paths): verify clean skip with no errors.
- **`tag.matchPatterns`** for migration: add `matchPatterns: ["v*"]` to pick up old single-repo tags.

---

## Notes

- The workflow uses `GITHUB_TOKEN` (no app token). PRs created by the action won't re-trigger workflows. You'll need to manually trigger or use an app token for the merge step.
- All phases are sequential. Each builds on the state left by the previous one.
- If a phase fails, fix the issue before moving to the next phase.
