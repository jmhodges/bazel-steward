# Design: Targeted single-dependency updates (`--only`, `--only-kind`, `--only-branch`) and a kind-qualified branch scheme

| | |
|---|---|
| **Status** | Draft / Proposed |
| **Author** | jeff@somethingsimilar.com |
| **Created** | 2026-06-06 |
| **Affected components** | `app`, `core`, `repo-config`, `kinds/*`, `github`, `docs` |

## Summary

Add the ability to run bazel-steward against **one** dependency (or one open
pull request) instead of scanning and updating everything. Three user-facing
inputs are introduced:

- `--only <name>` — target dependencies by id (exact / `*` glob / `regex:`),
  across any kind.
- `--only-kind <kind>` — restrict to a single dependency kind; usable to
  disambiguate `--only`, or on its own to scope a normal run.
- `--only-branch <branch>` — identify the dependency from a bazel-steward
  branch name and refresh exactly that one (ideal for CI / "rebase this PR").

To make branch names a reliable, unambiguous target — and to fix a
pre-existing cross-kind branch collision — branch generation moves to a
**kind-qualified `bazel-steward-v2/` scheme**, while still recognizing and
superseding legacy `bazel-steward/` branches so no open PRs are orphaned.

## Background and motivation

bazel-steward today performs a full run: every enabled dependency kind extracts
its libraries, resolves available versions from remote sources (Maven Central,
the Bazel Central Registry, GitHub, GCS), computes updates, and opens/updates a
PR per dependency.

There is no way to say "just refresh dependency X." Operators want this for:

- **Speed / low noise** — refresh one PR without re-scanning every ecosystem.
- **Recovery** — a specific PR developed conflicts, or a config/post-update-hook
  bug was fixed for one dependency, and you want to re-run only that one.
- **CI ergonomics** — "rebase the PR this workflow is about," keyed off the
  branch name.

### How bazel-steward already "rebases"

It never runs `git rebase`. Every run re-creates each branch from the latest
base and force-pushes when a PR already exists:

- `GitOperations.createBranchWithCommits` (`core/.../common/GitOperations.kt:51`)
  calls `checkoutBaseBranch()` then `git.checkout(branch, newBranch = true)`.
- `PullRequestManager.createOrUpdateBranchAndPr`
  (`app/.../PullRequestManager.kt:62`) force-pushes when `prStatus != NONE`.
- Docs confirm: *"When a pull request is no longer mergeable (has conflicts), it
  will force push the branch with the same version"*
  (`docs/docs/workflow/lifecycle-of-a-pull-request.md`).

So "rebase and update a single module's branch" is primarily about **scoping a
run to one dependency** and **force-refreshing that one PR** — not a new git
primitive.

### Key finding: the branch name does not encode the kind, and kinds collide today

Branches are built forward only, by `BazelStewardGitBranch`
(`app/.../BazelStewardGitBranch.kt`):

```
branch = <commonPrefix><libraryId.name with ':' -> '/'>/<version>
```

with the default `commonPrefix = "bazel-steward/"`
(`PullRequestConfigProvider.default.branchPrefix:52`). There is **no reverse
parser** anywhere in the codebase; the "same dependency?" check
(`PullRequestManager.isDifferentPrForTheSameDependency:103`) only ever computes
a prefix *forward* and does `startsWith`.

Per-kind id formats:

| Kind (`DependencyKind.name`) | `libraryId.name` | Legacy branch |
|---|---|---|
| `bazel` (version) | `bazel` (constant) | `bazel-steward/bazel/7.1.0` |
| `bzlmod` | module name, e.g. `rules_go` | `bazel-steward/rules_go/0.46.0` |
| `bazel-rules` | rule repo name, e.g. `rules_go` | `bazel-steward/rules_go/0.46.0` |
| `maven` | `group:artifact` | `bazel-steward/junit/junit/4.13.2` |

**`bzlmod` and `bazel-rules` share an id namespace.** A repo can have `rules_go`
as a `bazel_dep` in `MODULE.bazel` *and* as an `http_archive`/release-artifact
rule; both produce `name = "rules_go"` and the identical branch prefix
`bazel-steward/rules_go/`. This is a real, present-day collision, not a
hypothetical:

- The two cannot be distinguished from the branch name.
- Worse, because `isDifferentPrForTheSameDependency` matches purely on the
  shared prefix, a bzlmod `rules_go` update can today consider an (unmodified)
  bazel-rules `rules_go` PR "a different PR for the same dependency" and close
  it.

This finding drives two design decisions: (1) targeting must stay on the
**forward** path (enumerate live deps → compute branch → match), never reverse
a branch into a kind; and (2) to make branch-based targeting and PR
supersession correct, the **kind must be encoded into the branch**.

## Goals

- Update exactly one dependency by id, by kind, or by branch name.
- Refresh ("rebase") an existing open PR for the targeted dependency even when
  it is currently mergeable.
- Fix the cross-kind branch/PR collision so distinct kinds never share a branch.
- Fail loudly (non-zero exit) when a target matches nothing, or is ambiguous.
- Be additive: behavior is unchanged when none of the new flags are passed.

## Non-goals

- No new dependency kinds, version-selection strategies, or config-file schema
  changes.
- No `git rebase`; we keep the existing recreate-from-base + force-push model.
- No system-wide change making dependency identity kind-qualified everywhere
  (only the branch is qualified).
- No `action.yaml` change required (`--only*` flow through the existing
  `additional-args` input); a dedicated input may be added later for ergonomics.

## Design decisions (resolved)

| # | Decision | Choice |
|---|---|---|
| 1 | `--only` scope | Generic: matches any kind by `LibraryId.name`. |
| 2 | Targeted PR refresh | Auto-force the matched PR even if mergeable; `OPEN_MODIFIED` (human-pushed) PRs still require `--update-all-prs`. |
| 3 | No-match behavior | Hard error, exit code 1 (not a silent no-op). |
| 4 | Disambiguation | `--only-kind` + an ambiguity guard for exact `--only` matching >1 kind. |
| 5 | Collision fix | Kind-qualified `bazel-steward-v2/` branches; legacy `bazel-steward/` branches recognized and superseded. |
| 6 | `--only-branch` | Forward-match, v2-only (legacy rejected with guidance), mutually exclusive with `--only`/`--only-kind`. |

## Detailed design

### 1. Kind-qualified `bazel-steward-v2/` branch scheme

New branches encode the kind as the **first segment** after the common prefix:

```
bazel-steward-v2/<kind>/<sanitized-library-id>/<version>
```

| Scheme | bzlmod `rules_go` | bazel-rules `rules_go` | maven `junit:junit` |
|---|---|---|---|
| Legacy (read-only) | `bazel-steward/rules_go/0.46.0` | `bazel-steward/rules_go/0.46.0` ⚠️ | `bazel-steward/junit/junit/4.13.2` |
| v2 (new writes) | `bazel-steward-v2/bzlmod/rules_go/0.46.0` | `bazel-steward-v2/bazel-rules/rules_go/0.46.0` ✅ | `bazel-steward-v2/maven/junit/junit/4.13.2` |

`<kind>` is `DependencyKind.name` (`bazel`, `bzlmod`, `bazel-rules`, `maven`).
Because it is always the first segment after the common prefix, the kind is
unambiguously recoverable, and no maven group/artifact (segment ≥ 2) can be
mistaken for it. The kind for a library is obtained via
`DependencyKind.acceptsLibrary` (each library type is accepted by exactly one
kind), so no new field needs threading through `core`.

**Grouped PRs.** Config-defined groups use a user-owned `groupId` and may span
kinds. Rule: `kindSegment` = the lone kind when all of a suggestion's updates
share one (the normal single-dependency and multi-version cases), otherwise a
reserved `group` segment → `bazel-steward-v2/group/<groupId>/<version>`.
(`group` is not a kind name and is documented as reserved.)

### 2. Migration without orphaning PRs

Two parallel prefixes are tracked per resolved PR config:

- `branchPrefix` (new common prefix) = `configured ?: "bazel-steward-v2/"`
- `legacyBranchPrefix` = `configured ?: "bazel-steward/"`

This single rule covers both populations:

- **Default users:** new `bazel-steward-v2/…`, legacy `bazel-steward/…`.
- **Custom-prefix users** (e.g. `branchPrefix: my-bot/`): they only *gain* the
  kind segment — new `my-bot/<kind>/<id>/…`, legacy `my-bot/<id>/…`. No `-v2`
  rename is forced on them.

**Supersession.** When a v2 PR is created for dependency *(kind K, id I)*, the
existing supersede check matches open PRs whose branch starts with **either**:

- the v2 identity prefix `<commonPrefix><K>/<sanitized(I)>/` (older v2 versions), or
- the legacy identity prefix `<legacyBranchPrefix><sanitized(I)>/` (pre-migration PRs).

On the first run after upgrade, each active dependency's old
`bazel-steward/<id>/<oldver>` PR is found and **closed as superseded**, and its
`bazel-steward-v2/<kind>/<id>/<newver>` replacement is opened. The existing
`isUnmodified` guard still applies — human-modified legacy PRs (`OPEN_MODIFIED`)
are left alone. Open-PR **limit counting** includes both prefixes so the cap
stays accurate across the mixed state.

This is a deliberate one-time churn: each open bazel-steward PR moves namespaces
once. It must be called out in the changelog and lifecycle docs.

**Migration edge case.** A legacy `bazel-steward/rules_go/` branch is
kind-ambiguous. During migration, whichever kind's v2 run executes first closes
it; the other kind opens its own v2 PR with no legacy to close. Bounded and
self-healing — the dependency ends up correctly covered by both v2 PRs.

### 3. `--only <name>` and scoping

`--only` parses into the existing `DependencyNameFilter`
(`repo-config/.../DependencyNameFilter.kt`): exact, `*` glob, or `regex:`
prefix, matched against `LibraryId.name` (module name for bzlmod,
`group:artifact` for maven, etc.).

Scoping is applied in the existing `ignoreLibrary`/`skip` predicate in `App`,
which is already threaded into every kind's
`findAvailableVersions(workspaceRoot, skip)` and prunes libraries **before** the
expensive remote version lookups. The predicate is rebuilt per kind so it can
record `(kind.name, library.id)` for every target match (including disabled
libraries), which feeds the no-match and ambiguity checks.

```kotlin
val matches = mutableListOf<Pair<String, LibraryId>>()   // (kind.name, id)

fun skipFor(kind: DependencyKind<*>): (Library) -> Boolean = { lib ->
  val nameOk = targetMatches(lib)            // --only / --only-branch predicate, or true if none
  val kindOk = onlyKind == null || onlyKind == kind.name
  if (isTargeted && nameOk && kindOk) matches += kind.name to lib.id
  val disabled = !updateRulesProvider.resolveForLibrary(lib).enabled
  (disabled || !nameOk || !kindOk).also { if (it) logger.info { "Skipping ${lib.id} (…)" } }
}
```

### 4. `--only-kind <kind>`

Restricts a run to one kind, reusing the **same identifiers the config `kinds:`
filter uses** — it matches `DependencyKind.name` directly, exactly as
`UpdateRulesProvider.kt:17` and `DependencyFilterApplier.kt:15` do
(`bazel`, `bzlmod`, `bazel-rules`, `maven`). An unknown value fails fast.

- With `--only`: disambiguates (folds into `kindOk` above; suppresses the
  ambiguity guard).
- Alone: scopes a normal run to one kind, with **no** auto-force (auto-force is
  tied to naming a specific dependency, i.e. `--only`/`--only-branch`).

### 5. `--only-branch <branch>`

The most precise target. Because v2 branches encode the kind, a branch name maps
to **at most one** dependency — no ambiguity guard needed.

**Resolution (forward, never un-sanitizing):**

1. Strip the longest known v2 common prefix (from
   `resolveBranchPrefixes()`); the next segment is the **kind**, validated
   against `dependencyKinds.map { it.name }` → sets `onlyKind` (enabling the
   per-kind skip optimization).
2. A live library `L` matches iff `kindOf(L) == kind` and
   `(branch.removeSuffix("/") + "/").startsWith(L.v2IdentityPrefix)`, where
   `L.v2IdentityPrefix` is computed forward. This handles maven's multi-segment
   id and custom multi-segment prefixes, and tolerates input with or without the
   trailing version segment.
3. The matched dependency runs the normal auto-force refresh.

**Version is not pinned.** The branch identifies the *dependency*; the run then
computes the normal best update. If a newer version has appeared, it produces
the newer branch and supersedes the named one — standard bazel-steward behavior.

**Legacy branches rejected with guidance.** A `bazel-steward/<id>/<ver>` branch
carries no kind:

> `--only-branch` needs a v2 branch (`bazel-steward-v2/<kind>/…`);
> `bazel-steward/rules_go/0.46.0` is a legacy branch and doesn't encode the
> kind. Use `--only rules_go --only-kind <kind>` instead.

**Mutually exclusive** with `--only`/`--only-kind`. Internally it is sugar that
resolves to the same `(onlyKind, target)` those flags set, reusing all shared
machinery.

**Groups.** A `bazel-steward-v2/group/<groupId>/…` branch selects libraries
whose `resolveGroup` name is `<groupId>`; a single-kind group branch resolves by
matching a library id or a group id under that kind. v1 focuses on
single-dependency branches; unresolvable branches error.

### 6. Auto-force semantics

`PullRequestsLimits.canCreateOrUpdate` (`app/.../PullRequestsLimits.kt:50`)
currently blocks refreshing `OPEN_MERGEABLE`/`OPEN_MODIFIED` PRs unless
`--update-all-prs`. The merged branch is split so targeting can force a
mergeable PR while still protecting human-modified ones:

```kotlin
OPEN_MERGEABLE -> updatesLimit() and Result.check(updateAllPullRequests || targetedForce) {
  "PR is mergeable and neither --update-all-prs nor a target is set"
}
OPEN_MODIFIED  -> updatesLimit() and Result.check(updateAllPullRequests) {
  "PR was modified by user and --update-all-prs is false"
}
```

`targetedForce = (--only or --only-branch is set)`. Because non-targeted
libraries are filtered out before reaching `PullRequestManager`, every
suggestion that arrives under a target is the target, so a single flag is
sufficient. Behavior is unchanged when `targetedForce == false`.

### 7. Failure modes (all exit 1, clean message via `Main`)

Exit code is derived from the result map (`AppResult.exitCode():21`), and `Main`
only calls `exitProcess` when non-zero (`Main.kt:18`). An empty result map
otherwise yields exit 0, so explicit user-input errors are raised as typed
exceptions caught in `Main`:

- **No match** (`--only`/`--only-branch` matched no extant dependency) →
  `NoMatchingDependencyException` → exit 1. Distinguished from "matched but
  already up to date / disabled" via the `matches` set, so targeting a
  current dependency still exits 0.
- **Ambiguous** (exact `--only`, no `--only-kind`, matches span > 1 kind) →
  `AmbiguousDependencyException` listing `kind:name` matches and suggesting
  `--only-kind`. Glob/regex are exempt (multi-match is intentional); within a
  single kind an exact name resolves to ≤ 1 library, so the guard is precise.
- **Parse errors** for `--only-branch` (unknown/non-v2 prefix, invalid kind
  segment, legacy prefix) → exit 1 with the guided message.

```kotlin
val exitCode = try {
  runBlocking { AppBuilder.fromArgs(args, Environment.system).run().exitCode() }
} catch (e: NoMatchingDependencyException) { logger.error { e.message }; 1 }
  catch (e: AmbiguousDependencyException)  { logger.error { e.message }; 1 }
if (exitCode != 0) exitProcess(exitCode)
```

## Examples

```bash
# By id
java -jar bazel-steward.jar --github --only aspect_bazel_lib       # bzlmod module
java -jar bazel-steward.jar --github --only junit:junit            # maven dep

# Disambiguated (rules_go exists as both bzlmod and bazel-rules)
java -jar bazel-steward.jar --github --only rules_go --only-kind bzlmod

# Scope a normal (non-forced) run to one kind
java -jar bazel-steward.jar --github --only-kind maven

# By branch — most precise, CI-friendly
java -jar bazel-steward.jar --github --only-branch bazel-steward-v2/maven/junit/junit/4.13.2
java -jar bazel-steward.jar --github --only-branch "$GITHUB_HEAD_REF"

# GitHub Action
# with:
#   additional-args: --only aspect_bazel_lib
```

## Implementation plan

All changes are additive; existing behavior is unchanged when no new flag is
passed. Suggested commit stack:

**Commit 1 — kind-qualified v2 branch scheme + migration**
- `app/.../BazelStewardGitBranch.kt` — add `kindSegment` and
  `legacyCommonPrefix`; `prefix = commonPrefix + kindSegment + "/" +
  sanitized(id) + "/"`; expose `legacyPrefix`.
- `app/.../provider/PullRequestConfigProvider.kt` — default →
  `bazel-steward-v2/`; add `legacyBranchPrefix`; include legacy default in
  `resolveBranchPrefixes()`.
- `app/.../PullRequestSuggester.kt` — take a `kindNameOf: (Library) -> String`
  resolver (backed by `acceptsLibrary`); compute `kindSegment`; build the v2
  branch; set `supersedePrefixes = [v2 identity, legacy identity]`.
- `app/.../PullRequestSuggester.kt` `PullRequestSuggestion` — replace
  `branchPrefix: String` with `supersedePrefixes: List<String>`.
- `app/.../PullRequestManager.kt` — `isDifferentPrForTheSameDependency`
  `startsWith` against any of `supersedePrefixes`.

**Commit 2 — `--only` / `--only-kind` / ambiguity guard / no-match → exit 1**
- `repo-config/.../DependencyNameFilter.kt` — add `isExact`.
- `app/.../AppBuilder.kt` — `--only`, `--only-kind` options; parse + validate
  kind; thread `targetFilter`, `onlyKind`, `targetedForce` into `App` and
  `PullRequestsLimitsProvider`.
- `app/.../App.kt` — per-kind `skip` with `(kind, id)` match recording;
  no-match and ambiguity checks; new exceptions.
- `app/.../PullRequestsLimits.kt` (+ `PullRequestsLimitsProvider.kt`) — split
  `OPEN_MERGEABLE`/`OPEN_MODIFIED`; add `targetedForce`.
- `app/.../Main.kt` — catch the new exceptions → exit 1.

**Commit 3 — `--only-branch`**
- `app/.../OnlyBranchResolver.kt` (new) — parse prefix + kind; build the
  forward-match predicate; legacy/parse error handling.
- `app/.../AppBuilder.kt` — `--only-branch` option; mutual-exclusion check;
  resolve into `onlyKind` + target.
- `app/.../App.kt` — accept a branch-target predicate in the same `skip` path.

**Docs** — `docs/docs/configuration/command-line-arguments.md` (all three
flags), `docs/docs/workflow/lifecycle-of-a-pull-request.md` (v2 scheme,
supersession, one-time migration churn).

## Testing strategy

- **Unit — branch format:** all four kinds + a config group (reserved `group`
  segment); maven multi-segment id; custom `branchPrefix`.
- **Unit — supersession:** a legacy `bazel-steward/<id>/…` PR is matched and
  closed when its v2 PR opens; an `OPEN_MODIFIED` legacy PR is not.
- **Unit — limits:** `PullRequestsLimits` with `targetedForce` allows
  `OPEN_MERGEABLE`, blocks `OPEN_MODIFIED`; `--update-all-prs` allows both;
  open-PR counting spans legacy + v2.
- **Unit — `--only-branch` parsing:** valid v2 branch per kind; trailing-version
  vs prefix-only input; legacy branch → guided error; bogus kind → parse error;
  mutual exclusion with `--only`.
- **Unit — `DependencyNameFilter.isExact`.**
- **e2e (`e2e/`, alongside `PrManagementTest`/`BzlModTest`):**
  - `--only <dep with update>` produces only that `bazel-steward-v2/<kind>/…`
    branch and supersedes its legacy PR;
  - `--only <dep already at latest>` → no throw, exit 0;
  - `--only doesNotExist` → `NoMatchingDependencyException`, exit 1;
  - `--only rules_go` (ambiguous) → `AmbiguousDependencyException`;
    `--only rules_go --only-kind bzlmod` resolves it;
  - `--only-branch <v2 branch>` refreshes exactly that dependency; with a newer
    version available, opens the newer branch and supersedes the named one;
  - migration: fixture repo with a pre-existing legacy PR → run → legacy closed,
    v2 opened.

## Alternatives considered

- **Reverse-parse the kind from a legacy branch.** Rejected: legacy branches
  don't encode the kind, the prefix/id/version split is ambiguous, and
  `bzlmod`/`bazel-rules` already share names. The forward-match + v2 scheme is
  the robust path.
- **`kind:name` inline syntax for `--only`** (e.g. `maven:junit:junit`).
  Rejected: maven names already contain colons, making the separator ambiguous;
  a distinct `--only-kind` flag is unambiguous and mirrors config.
- **"Any cross-kind match errors"** (instead of the exact-only ambiguity guard).
  Rejected: it would break intentional multi-match globs/regex.
- **Make dependency identity kind-qualified system-wide.** Rejected as
  disproportionate; only the branch needs the kind.
- **Rename the prefix to `bazel-steward-v2/` without inserting the kind.**
  Rejected: a prefix-only rename does not fix the collision (`rules_go` still
  collides under any shared prefix); the kind segment is the actual fix.

## Risks and open questions

- **One-time PR churn at migration** (close legacy, open v2 per active
  dependency). Mitigation: documented; `isUnmodified` guard preserves
  human-modified PRs.
- **Disabled-but-targeted dependency:** currently treated as a non-error
  ("nothing to do"), not a no-match error. Open question whether a `--only`
  that matches only a disabled dependency should also exit 1. Default: no.
- **Custom `branchPrefix` users** gain the kind segment; their legacy is the
  same prefix without the kind segment. No `-v2` rename is imposed.

## Appendix: key references

- Orchestration: `app/.../App.kt:37` (`run`), `:75` (`ignoreLibrary`).
- CLI parsing: `app/.../AppBuilder.kt:70`.
- PR lifecycle: `app/.../PullRequestManager.kt` (`:62` force-push, `:88`
  supersede, `:103` same-dependency check).
- Limits: `app/.../PullRequestsLimits.kt:50`; `provider/PullRequestsLimitsProvider.kt`.
- Branch naming: `app/.../BazelStewardGitBranch.kt`;
  `provider/PullRequestConfigProvider.kt:36` (`resolveBranchPrefixes`), `:52`
  (default prefix).
- Kind names: `BzlModDependencyKind.name = "bzlmod"`,
  `BazelRulesDependencyKind.name = "bazel-rules"`,
  `MavenDependencyKind.name = "maven"`,
  `BazelVersionDependencyKind.name = "bazel"`.
- Id formats: `BazelModule.kt` (bzlmod), `MavenCoordinates.kt:16` (maven),
  `RuleLibraryId.kt:12` (bazel-rules), `BazelUpdater.kt:15` (bazel).
- Exit code: `app/.../App.kt:21`; `app/.../Main.kt:18`.
