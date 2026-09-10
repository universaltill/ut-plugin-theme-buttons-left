# Code review: manifest.json version-bump CI guard (ut-docs#1948)

**Date:** 2026-09-10
**Author:** Farshid Mirza (pipeline, Sonnet dev, `complexity:medium`)
**Independent reviewer:** Opus, fresh-context subagent (`isolation: "worktree"`)
**PR:** universaltill/ut-plugin-theme-buttons-left#3 (branch: `feat/1948-version-bump-guard`)

## What shipped

Fourth deployment of the ut-docs#1940 guard: `ut-plugin-language-de`
(original) → `ut-plugin-theme-midnight` → `ut-plugin-theme-screen-top` →
`ut-plugin-theme-buttons-left` (this PR), the last of the three
straightforward asset-only theme repos named in ut-docs#1948. Byte-identical
shape to the two prior theme repos (`runtime: "none"`, no `src/`,
`entries=(manifest.json assets README.md)`), so `check-version-bump.sh`
and `.github/workflows/ci.yml`'s new `version-bump` job carry over
verbatim; only the test fixture's manifest id/name/theme key and README
placeholder strings needed repo-specific adaptation. Ships with the
malformed-shellcheck-directive fix (F1, found in `theme-midnight`'s
review) already in the first commit.

## Independent review findings

Verdict: **SAFE TO MERGE**, no blockers. The reviewer ran the full
verification suite itself rather than trusting the diff or the prior
repos' review records:

- Diffed `check-version-bump.sh` against the reference
  (`ut-plugin-theme-screen-top`, cloned read-only): **byte-identical**.
- Diffed `check-version-bump.test.sh` against the same reference: differs
  in exactly 5 lines — `id`, `name`, `entries[0].key`, and two README
  fixture strings (`screen-top` → `buttons-left`). No logic drift.
- `.github/workflows/ci.yml`'s new `version-bump` job: byte-identical to
  the reference (`if: github.event_name == 'pull_request'` gate,
  `fetch-depth: 0`, self-test step before the real check).
- `CLAUDE.md`'s new bullet: byte-identical to `theme-screen-top`'s
  (the newer wording — `docs/`, `.github/` **and** `scripts/` all named
  as exempt — confirming the port took the corrected text, not
  `theme-midnight`'s older phrasing).
- 11/11 self-tests reproduced locally, including the `entries mirror`
  and already-tagged cases. `validate.sh`/`package.sh` both clean.
- Read `package.sh`'s real `entries=(...)` line and `SHIPPED_PATTERNS`
  itself (not just trusting the self-test) — confirmed an exact 1:1
  correspondence, and empirically confirmed `assets/*` crosses into a
  nested subdirectory (`assets/sub/nested.css` correctly triggers a FAIL).
- Independent scratch-commit simulation on a detached HEAD in the
  reviewer's own worktree: asset edit without bump → FAIL naming the
  file and suggesting `1.0.4`; bump too → PASS; a locally-created
  `v1.0.4` tag → FAIL, `ALREADY TAGGED`; the no-env local-fallback path
  (comparing against `origin/main`) also resolved correctly. All scratch
  commits/tags discarded, worktree confirmed clean afterward.
- `shellcheck` wasn't available in the orchestrating session — the
  reviewer downloaded v0.10.0 itself and got **0 findings** on both
  scripts at default severity and `-S style`. Proved the F1 fix is
  load-bearing (not just present) by stripping the directive from a copy
  and confirming SC2053 then fires.
- Confirmed the repo-specific claims in the guard's own header comment
  hold here too: `commit-attribution.yml` uses the same `BASE_SHA`/
  `HEAD_SHA` convention, `auto-tag-release.yml` no-ops on an existing tag
  and accepts `-prerelease` suffixes, `validate.sh`'s version regex is
  unanchored.
- Self-consistency: ran the guard against this PR's own diff (touches
  only `scripts/`, `.github/`, `CLAUDE.md`) — correctly reports no
  shipped file changed. Commit author verified as
  `Farshid Mirza <4035824+farshidmirza@users.noreply.github.com>`.

**Nits, all inherited verbatim from the reference implementation, not
fixed here (same "prove the pattern, don't fix piecemeal per repo"
scoping this rollout has used throughout):**
- A file **rename** of a shipped file (e.g. `README.md` →
  `docs/README.md`) is a false negative — `git diff --name-only` reports
  only the new path, which doesn't match any literal `SHIPPED_PATTERNS`
  entry, so the guard passes even though the release bundle silently
  loses the file. Low severity (obscure, and `validate.sh` would still
  catch a missing `config.css`).
- The `entries mirror` self-test extracts *every* single-quoted string
  literal from the real script, not just the `SHIPPED_PATTERNS` array
  contents — could false-*pass* (never false-fail) if a future
  `package.sh` entry happened to collide with an unrelated quoted
  literal elsewhere in the script.
- `ci.yml`'s self-test step `if: always()` is a near-no-op (it's the
  first step after checkout); only effect is running the self-test past
  a failed checkout, producing a confusing failure instead of a clear one.

None of these are new findings — they've each been raised in at least one
prior repo's review and left un-fixed there too, tracked as an ut-docs#1948
follow-up rather than resolved piecemeal.

## What was verified beyond automated tests

Everything in "Independent review findings" above was run for real by the
reviewer in an isolated worktree — the byte-diffs against the reference,
the scratch-commit failure-mode simulation (including the already-tagged
case and the no-env fallback path), and the shellcheck load-bearing proof
(strip → confirm it fires → this repo's version doesn't have that bug).

## Safe-to-merge verdict

**Yes.** No blockers. This closes out the three straightforward
asset-only theme repos named in ut-docs#1948
(`theme-midnight`, `theme-screen-top`, `theme-buttons-left`, all now
shipped). Remaining scope on ut-docs#1948 itself: the Go/WASM plugins
need a design variant (gitignored `bin/` build artifact isn't visible to
a git-diff-based guard), `ut-plugin-integration-ai`'s non-array
`package.sh` convention needs a different detection approach, and the F1
shellcheck fix should be ported back to `ut-plugin-language-de`
(confirmed still broken) and checked in `-es`.
