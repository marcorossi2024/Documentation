# Release Simulation: Developer Point of View

Three complete release cycles with a **real git repository** showing actual commits, branches, merges, and tags.

---

# Release Cycle 1: User Authentication

## Phase 1: Feature Development

```bash
# Create feature branch
git checkout -b feature/user-auth

# Implement the feature with conventional commits
git commit -m "feat: add login endpoint with token generation"
git commit -m "feat: add password hashing utility"
git commit -m "test: add authentication unit tests"
```

PR shows these commits:

```
d86fc90 test: add authentication unit tests
92b8e04 feat: add password hashing utility
444e1cc feat: add login endpoint with token generation
```

> **ci.yml** triggers on PR: Build ✅ Tests ✅. Reviewer approves.

```bash
git checkout main
git merge feature/user-auth --no-ff -m "Merge feature/user-auth (#1)"
git branch -d feature/user-auth
```

## Phase 2: Automatic Dev Deployment

The merge triggers **deploy-dev.yml** automatically:

```
WORKFLOW: deploy-dev.yml (triggered by push to main)

  1. Read version.properties       -> 1.0.0-SNAPSHOT
  2. Validate SNAPSHOT suffix       -> PASS
  3. ./gradlew clean build -x test  -> myapp-1.0.0-SNAPSHOT.jar
  4. ./gradlew publish              -> GitHub Packages (SNAPSHOT)
  5. scp jar to dev-server          -> /opt/app/myapp-1.0.0-SNAPSHOT.jar
  6. Write deploy-info.json         -> {version:1.0.0-SNAPSHOT, env:dev}
  7. systemctl restart app          -> RUNNING
```

```bash
# Developer verifies on Dev
curl http://dev-server:8080/health
# -> {"status":"UP","version":"1.0.0-SNAPSHOT"}

curl -X POST http://dev-server:8080/auth/login -d '{"user":"test","pass":"s3cret"}'
# -> {"token":"tok-3556498"}
# Feature works on Dev!
```

### Bug Fix During Dev Testing

QA finds empty passwords are accepted:

```bash
git commit -m "fix: reject empty password in login"
git push origin main
```

> **deploy-dev.yml** fires again. Still 1.0.0-SNAPSHOT, but with the fix.
> This is the SNAPSHOT benefit: same version, iterable builds.

## Phase 3: QA Release (Manual Trigger)

Navigate to **GitHub Actions -> Deploy to QA -> Run workflow -> confirm = true**

```
WORKFLOW: deploy-qa.yml (triggered by manual dispatch)

  1.  Read version.properties        -> 1.0.0-SNAPSHOT
  2.  Validate SNAPSHOT               -> PASS
  3.  Strip -SNAPSHOT                 -> version.properties = 1.0.0
  4.  Generate release notes          -> from git log (5 commits)
  5.  ./gradlew clean build test      -> myapp-1.0.0.jar
  6.  sha256sum myapp-1.0.0.jar       -> e3b0c44298fc... checksums.txt
  7.  git commit -m "release: v1.0.0"
  8.  git tag -a v1.0.0
  9.  git push origin main --tags
  10. Create GitHub Release v1.0.0    -> jar + checksums.txt + notes
  11. ./gradlew publish               -> GitHub Packages
  12. scp jar to qa-server            -> /opt/app/myapp-1.0.0.jar
  13. Write deploy-info.json          -> {version:1.0.0, env:qa, tag:v1.0.0}
```

### Auto-generated release notes:

```markdown
# Release Notes - v1.0.0
**Release Date:** 2026-03-03  |  **Commits:** 5

## New Features
- feat: add login endpoint with token generation (Luigi)
- feat: add password hashing utility (Luigi)

## Bug Fixes
- fix: reject empty password in login (Luigi)

## Tests
- test: add authentication unit tests (Luigi)
```

> QA tests pass. Sign-off granted.

## Phase 4: Production Release (Manual Trigger)

```
WORKFLOW: release-prod.yml
  Inputs: release_tag=v1.0.0, bump_type=minor

  1. Download v1.0.0 from GitHub Release
  2. sha256sum --check checksums.txt  -> OK (same binary as QA)
  3. scp jar to prod-server           -> /opt/app/myapp-1.0.0.jar
  4. Write deploy-info.json           -> {version:1.0.0, env:production}
  5. Compute next: 1.0.0 + minor      -> 1.1.0
  6. version.properties               -> 1.1.0-SNAPSHOT
  7. git commit + push to main
```

> **deploy-dev.yml** auto-triggers. Dev now runs **1.1.0-SNAPSHOT**.

```
SERVER STATE after Release Cycle 1:
  Dev:  myapp-1.1.0-SNAPSHOT.jar
  QA:   myapp-1.0.0.jar (v1.0.0)
  Prod: myapp-1.0.0.jar (v1.0.0, SHA-256 verified)
  Tags: v1.0.0
```

Git log:

```
* 42ce311 (HEAD -> main) chore: bump to 1.1.0-SNAPSHOT for next dev cycle
* cc4b96a (tag: v1.0.0) release: v1.0.0
* 517e59e fix: reject empty password in login
*   e8607fc Merge feature/user-auth (#1)
|\  
| * d86fc90 test: add authentication unit tests
| * 92b8e04 feat: add password hashing utility
| * 444e1cc feat: add login endpoint with token generation
|/  
* f48cb3b chore: initial project setup
```

---

# Release Cycle 2: CSV Export + Rate Limiting

Two features developed **in parallel** by different developers.

## Phase 1: Parallel Development

```bash
# Developer A: CSV export
git checkout -b feature/csv-export
git commit -m "feat: add CSV export service"
git commit -m "feat: add export download endpoint"
git commit -m "fix: escape commas in CSV values"

# Developer B: Rate limiting (simultaneously)
git checkout main && git checkout -b feature/rate-limiting
git commit -m "feat: add token-bucket rate limiter"
git commit -m "feat: add rate limit headers to responses"
git commit -m "docs: add rate limiting section to API docs"

# Both PRs pass CI, reviewed, merged to main
git merge feature/csv-export --no-ff
git merge feature/rate-limiting --no-ff
```

## Phases 2-4: Dev -> QA -> Prod

> deploy-dev.yml auto-triggers. Both features on Dev. Tested. All good.

```
WORKFLOW: deploy-qa.yml -> strip SNAPSHOT -> 1.1.0 -> build -> tag v1.1.0 -> deploy QA

Release notes v1.1.0 (since v1.0.0):
  ## New Features
  - feat: add CSV export service (Developer A)
  - feat: add export download endpoint (Developer A)
  - feat: add token-bucket rate limiter (Developer B)
  - feat: add rate limit headers to responses (Developer B)
  ## Bug Fixes
  - fix: escape commas in CSV values (Developer A)
  ## Documentation
  - docs: add rate limiting section to API docs (Developer B)
  ## Contributors: Developer A, Developer B

QA tests pass. Sign-off granted.

WORKFLOW: release-prod.yml -> download v1.1.0 -> SHA-256 OK -> deploy Prod -> bump 1.2.0-SNAP
```

```
SERVER STATE after Release Cycle 2:
  Dev:  myapp-1.2.0-SNAPSHOT.jar
  QA:   myapp-1.1.0.jar (v1.1.0)
  Prod: myapp-1.1.0.jar (v1.1.0, SHA-256 verified)
  Tags: v1.0.0  v1.1.0
```

---

# Release Cycle 3: Production Hotfix

**Scenario:** Customer reports rate limiter blocks traffic at 100 req/min.
Prod runs v1.1.0. Main is at 1.2.0-SNAPSHOT.

**Strategy:** Fix on main, expedite through QA, ship as v1.2.0.

## Hotfix Development

```bash
# Fix directly on main for speed - no feature branch needed
git commit -m "fix: increase default rate limit from 100 to 1000 req/min"
git commit -m "test: add rate limiter load test"
git push origin main
```

> deploy-dev.yml auto-triggers. Dev updated with fix. Quick verification: rate limiter OK.

## Emergency QA + Prod Release

```
WORKFLOW: deploy-qa.yml -> strip SNAPSHOT -> 1.2.0 -> build -> tag v1.2.0 -> deploy QA

Release notes v1.2.0 (since v1.1.0):
  ## Bug Fixes
  - fix: increase default rate limit from 100 to 1000 req/min (Luigi)
  ## Tests
  - test: add rate limiter load test (Luigi)

Expedited QA: focused on fix + regression -> PASS

WORKFLOW: release-prod.yml -> download v1.2.0 -> SHA-256 OK -> deploy Prod -> bump 1.2.1-SNAP
```

> Customer confirms: rate limiting resolved. Incident closed.

**The hotfix used the exact same workflow as normal releases. No special process, no emergency branches, no manual version edits.**

---

# Release Cycle 4: Failed QA Build (Recovery)

**Scenario:** The team tries to cut a QA release, but the build fails.
This demonstrates why the workflow builds BEFORE committing the version change.

## Development

```bash
git commit -m "feat: add webhook notification system"
git commit -m "feat: add retry logic for failed webhooks"
git commit -m "test: add webhook integration tests"
```

> deploy-dev.yml auto-triggers. Dev runs 1.2.1-SNAPSHOT with webhooks.

## First QA Attempt: FAILS

```
WORKFLOW: deploy-qa.yml (triggered by manual dispatch)

  1.  Read version.properties       -> 1.2.1-SNAPSHOT
  2.  Validate SNAPSHOT              -> PASS
  3.  Strip -SNAPSHOT                -> 1.2.1
  4.  Generate release notes
  5.  ./gradlew clean build test     -> FAILED!

  WebhookRetryTest > testExponentialBackoff FAILED
    java.lang.AssertionError: Expected delay 2000ms but was 1500ms

  BUILD FAILED in 47s

  !! Build failed BEFORE commit/tag/push.
  !! No version change was committed to the repo.
  !! No tag was created.
  !! No GitHub Release exists.
  !! Repository remains at: version=1.2.1-SNAPSHOT
  !! The release can simply be re-attempted after fixing the test.
```

This is the critical safety net: **the build happens before the git commit**.
Because the test failed, the workflow aborted before any permanent changes.
Nothing to clean up. Nothing to revert.

## Developer fixes the test

```bash
git commit -m "fix: correct exponential backoff timing in webhook retry"
git push origin main
```

> deploy-dev.yml triggers. Dev updated. Test passes locally.

## Second QA Attempt: SUCCEEDS

```
WORKFLOW: deploy-qa.yml (re-triggered)

  1.  Read version.properties       -> 1.2.1-SNAPSHOT
  2.  Strip -SNAPSHOT                -> 1.2.1
  3.  ./gradlew clean build test     -> PASS
  4.  Tag v1.2.1, GitHub Release, deploy to QA

  Release notes v1.2.1 (since v1.2.0):
    ## New Features
    - feat: add webhook notification system (Luigi)
    - feat: add retry logic for failed webhooks (Luigi)
    ## Bug Fixes
    - fix: correct exponential backoff timing (Luigi)
    ## Tests
    - test: add webhook integration tests (Luigi)
```

**Key takeaway:** A failed QA release is a non-event. No cleanup required. Fix the issue and retry.

## Prod Release

```
WORKFLOW: release-prod.yml -> download v1.2.1 -> SHA-256 OK -> deploy Prod -> bump 1.3.0-SNAP
```

```
SERVER STATE:
  Dev:  myapp-1.3.0-SNAPSHOT.jar
  QA:   myapp-1.2.1.jar (v1.2.1)
  Prod: myapp-1.2.1.jar (v1.2.1, SHA-256 verified)
  Tags: v1.0.0  v1.1.0  v1.2.0  v1.2.1
```

---

# Release Cycle 5: Abandoned QA Release (Superseded)

**Scenario:** A release is deployed to QA but testing reveals a fundamental
design flaw. Instead of patching it, the team decides to rework the feature
and release a new version. The QA release is never promoted to Prod.

## Feature development and QA release

```bash
git commit -m "feat: add real-time dashboard with WebSocket"
git commit -m "feat: add dashboard chart widgets"
```

```
WORKFLOW: deploy-qa.yml -> strip SNAPSHOT -> 1.3.0 -> tag v1.3.0 -> deploy QA
```

## QA Rejects the Release

```
QA feedback:
  - WebSocket connections drop under load (>50 concurrent users)
  - Dashboard refresh rate causes browser memory leak
  - Team decision: REWORK the approach, switch from WebSocket to SSE
  - v1.3.0 will NOT be promoted to Production
```

**What happens to v1.3.0?** Nothing special. The git tag and GitHub Release
remain as a historical record, but nobody runs release-prod.yml for it.
Production stays on v1.2.1.

## Rework and new release

Since the Prod workflow was never run for v1.3.0, no automatic version bump happened.
The developer needs to manually advance to the next SNAPSHOT:

```bash
# The QA workflow already committed version=1.3.0 (clean).
# Normally Prod would bump this, but we skipped Prod.
# Manually set the next version:
sed -i "s/version=1.3.0/version=1.4.0-SNAPSHOT/" version.properties
git commit -am "chore: skip v1.3.0 prod release, advance to 1.4.0-SNAPSHOT"
git push origin main
```

> This is the **one case** where a developer manually edits version.properties.
> It only happens when a QA release is abandoned without ever going to Prod.

```bash
git commit -m "refactor: replace WebSocket with Server-Sent Events"
git commit -m "fix: resolve memory leak in dashboard refresh"
git commit -m "test: add load test for SSE connections"
```

```
WORKFLOW: deploy-qa.yml -> strip SNAPSHOT -> 1.4.0 -> tag v1.4.0 -> deploy QA

Release notes v1.4.0 (since v1.3.0):
  ## Refactoring
  - refactor: replace WebSocket with Server-Sent Events (Luigi)
  ## Bug Fixes
  - fix: resolve memory leak in dashboard refresh (Luigi)
  ## Tests
  - test: add load test for SSE connections (Luigi)

QA tests pass this time. Dashboard stable under load. Approved.

WORKFLOW: release-prod.yml -> download v1.4.0 -> SHA-256 OK -> deploy Prod -> bump 1.5.0-SNAP
```

**Note:** Production jumped from v1.2.1 to v1.4.0. v1.3.0 was never in production.
The git tags preserve the audit trail that it existed and was tested, but it was skipped.

```
SERVER STATE:
  Dev:  myapp-1.5.0-SNAPSHOT.jar
  QA:   myapp-1.4.0.jar (v1.4.0)
  Prod: myapp-1.4.0.jar (v1.4.0, SHA-256 verified)  -- jumped from v1.2.1!
  Tags: v1.0.0  v1.1.0  v1.2.0  v1.2.1  v1.3.0  v1.4.0
```

---

# Release Cycle 6: Production Rollback

**Scenario:** v1.4.0 has been in Production for two days. A subtle data corruption
bug is discovered in the SSE dashboard that only manifests under specific conditions.
The team needs to roll back Production to the previous known-good version immediately.

## Immediate Rollback

Rollback does not require a new workflow. The existing Prod workflow can deploy **any**
previously released version, since all releases are preserved as immutable GitHub Releases.

```
WORKFLOW: release-prod.yml
  Inputs: release_tag=v1.2.1, bump_type=patch

  >>> DEPLOYING AN OLDER TAG <<<  

  1. Download v1.2.1 from GitHub Release
     (this release still exists -- GitHub Releases are immutable)
  2. sha256sum --check checksums.txt  -> OK
     (identical to the binary that was originally in Prod)
  3. Deploy to Prod server
  4. Write deploy-info.json

  PRODUCTION IS NOW RUNNING v1.2.1 (rolled back)
```

### Version conflict after rollback

The Prod workflow normally bumps version.properties. But main is at 1.5.0-SNAPSHOT
and the rollback wants to write 1.2.2-SNAPSHOT. This creates a version conflict.

```
Problem:  main has version=1.5.0-SNAPSHOT (active development)
          Prod workflow wants to write version=1.2.2-SNAPSHOT (from rollback bump)

Solution: Skip the automatic bump for rollbacks.
          The rollback is an operational action, not a version reset.
          Main continues at 1.5.0-SNAPSHOT.

Recommended workflow enhancement:
  Add a 'skip_bump' boolean input to release-prod.yml.
  Set skip_bump=true when rolling back.
```

> **Key insight:** Rollback is just deploying an older GitHub Release. The binary
> is guaranteed identical via SHA-256. No rebuild. No source checkout.

```
SERVER STATE (during rollback):
  Dev:  myapp-1.5.0-SNAPSHOT.jar  (active development continues)
  QA:   myapp-1.4.0.jar (v1.4.0) (last tested release)
  Prod: myapp-1.2.1.jar (v1.2.1) (ROLLED BACK from v1.4.0)
```

## Fix and re-release

```bash
# Developer fixes the SSE data corruption bug
git commit -m "fix: prevent SSE race condition causing data corruption"
git commit -m "test: add concurrent SSE session stress test"
git push origin main
```

```
WORKFLOW: deploy-qa.yml -> 1.5.0 -> tag v1.5.0 -> deploy QA
  QA specifically retests the SSE race condition scenario -> PASS

WORKFLOW: release-prod.yml -> download v1.5.0 -> SHA-256 OK -> deploy Prod -> bump 1.6.0-SNAP
```

```
SERVER STATE (after fix):
  Dev:  myapp-1.6.0-SNAPSHOT.jar
  QA:   myapp-1.5.0.jar (v1.5.0)
  Prod: myapp-1.5.0.jar (v1.5.0, SHA-256 verified)
  Prod history: v1.4.0 -> v1.2.1 (rollback) -> v1.5.0 (fix)
```

---

# Release Cycle 7: Major Version Bump (Breaking API Change)

**Scenario:** The team migrates from the legacy auth system to OAuth 2.0.
The /auth/login endpoint is removed and replaced with /oauth/authorize.
This is a breaking change that requires a major version bump.

## Development

```bash
git checkout -b feature/oauth2-migration

# Note the '!' after 'feat' -- this signals a BREAKING CHANGE
# per the Conventional Commits specification
git commit -m "feat!: replace login endpoint with OAuth 2.0 flow"
git commit -m "feat: add OAuth token refresh endpoint"
git commit -m "feat: add OAuth scope validation"
git commit -m "docs: update API migration guide for v2.0"
git commit -m "refactor: remove legacy AuthController"
git commit -m "test: add OAuth 2.0 integration test suite"

git checkout main
git merge feature/oauth2-migration --no-ff
```

> deploy-dev.yml triggers. Dev runs 1.6.0-SNAPSHOT with OAuth 2.0.

## QA Release

```
WORKFLOW: deploy-qa.yml -> strip SNAPSHOT -> 1.6.0 -> tag v1.6.0 -> deploy QA

Release notes v1.6.0 (since v1.5.0):
  ## BREAKING CHANGES
  - feat!: replace login endpoint with OAuth 2.0 flow (Luigi)
    The /auth/login endpoint has been removed.
    Use /oauth/authorize instead.
  ## New Features
  - feat: add OAuth token refresh endpoint (Luigi)
  - feat: add OAuth scope validation (Luigi)
  ## Refactoring
  - refactor: remove legacy AuthController (Luigi)
  ## Documentation
  - docs: update API migration guide for v2.0 (Luigi)
  ## Tests
  - test: add OAuth 2.0 integration test suite (Luigi)

QA tests pass. API consumers notified of breaking change.
```

## Prod Release with Major Bump

The operator selects **bump_type = major** because this is a breaking change:

```
WORKFLOW: release-prod.yml
  Inputs: release_tag=v1.6.0, bump_type=major

  Download v1.6.0 -> SHA-256 OK -> deploy Prod
  Compute next: 1.6.0 + major -> 2.0.0
  version.properties -> 2.0.0-SNAPSHOT
  Commit + push to main
```

**Note:** The version jumped from 1.6.0 to 2.0.0. The major bump signals to
all consumers that this release contains breaking API changes. The release notes
auto-detected the `feat!` prefix and created a BREAKING CHANGES section.

```
SERVER STATE:
  Dev:  myapp-2.0.0-SNAPSHOT.jar
  QA:   myapp-1.6.0.jar (v1.6.0)
  Prod: myapp-1.6.0.jar (v1.6.0, SHA-256 verified)
  Tags: v1.0.0  v1.1.0  v1.2.0  v1.2.1  v1.3.0  v1.4.0  v1.5.0  v1.6.0
```

---

# Final State: All 7 Cycles Complete

## Servers

| Environment | Jar | Version | Notes |
|-------------|-----|---------|-------|
| **Dev** | myapp-2.0.0-SNAPSHOT.jar | 2.0.0-SNAPSHOT | Ready for v2 development |
| **QA** | myapp-1.6.0.jar | 1.6.0 | Last tested release |
| **Prod** | myapp-1.6.0.jar | 1.6.0 | Same binary as QA (SHA-256 verified) |

## Complete Release History

| Release | Contents | Type | Promoted to Prod? |
|---------|----------|------|-------------------|
| v1.0.0 | User Authentication | Normal (initial) | Yes |
| v1.1.0 | CSV Export + Rate Limiting | Normal (minor) | Yes |
| v1.2.0 | Hotfix: Rate limiter threshold | Hotfix (patch) | Yes |
| v1.2.1 | Webhook notifications | Normal (patch) | Yes (after build retry) |
| v1.3.0 | Dashboard (WebSocket) | Normal (minor) | **No - QA rejected** |
| v1.4.0 | Dashboard (SSE rework) | Normal (minor) | Yes, then **rolled back** |
| v1.5.0 | Dashboard (SSE fix) | Normal (minor) | Yes (after rollback) |
| v1.6.0 | OAuth 2.0 migration | Breaking (major) | Yes |

## Version Timeline

```
1.0.0-S -> 1.0.0 -> 1.1.0-S -> 1.1.0 -> 1.2.0-S -> 1.2.0 -> 1.2.1-S -> [FAIL] -> 1.2.1
---Dev-- -QA/Prod- ---Dev---- -QA/Prod ---Dev---- -QA/Prod- ---Dev----  build err  -QA/Prod

 -> 1.3.0-S -> 1.3.0 -> 1.4.0-S -> 1.4.0 -> 1.5.0-S -> 1.5.0 -> 1.6.0-S -> 1.6.0 -> 2.0.0-S
    --Dev--- -QA ONLY   ---Dev---- -QA/Prod- ---Dev---- -QA/Prod- ---Dev---- -QA/Prod- ---Dev---
             ABANDONED            ROLLED BACK            RESTORED             BREAKING
```

## Complete Git History

```
* ad61d27 (HEAD -> main) chore: bump to 2.0.0-SNAPSHOT for next dev cycle
* 3fb9fa3 (tag: v1.6.0) release: v1.6.0
*   3c40452 Merge feature/oauth2-migration (#7)
|\  
| * ad3444d test: add OAuth 2.0 integration test suite
| * 9041391 refactor: remove legacy AuthController
| * 01220ed docs: update API migration guide for v2.0
| * 622941f feat: add OAuth scope validation
| * 6071fcd feat: add OAuth token refresh endpoint
| * bacb627 feat!: replace login endpoint with OAuth 2.0 flow
|/  
* 235ab3e chore: bump to 1.6.0-SNAPSHOT for next dev cycle
* afe5906 (tag: v1.5.0) release: v1.5.0
* 5406fbe test: add concurrent SSE session stress test
* 615c92a fix: prevent SSE race condition causing data corruption
* c63a3d5 chore: bump to 1.5.0-SNAPSHOT for next dev cycle
* 332145f (tag: v1.4.0) release: v1.4.0
* 77e7120 test: add load test for SSE connections
* 6b07117 fix: resolve memory leak in dashboard refresh
* fa20c06 refactor: replace WebSocket with Server-Sent Events
* 44c8974 chore: skip v1.3.0 prod release, advance to 1.4.0-SNAPSHOT
* e027685 (tag: v1.3.0) release: v1.3.0
* 98f2b53 feat: add dashboard chart widgets
* 7828dda feat: add real-time dashboard with WebSocket
* c413292 chore: bump to 1.3.0-SNAPSHOT for next dev cycle
* b65fc0b (tag: v1.2.1) release: v1.2.1
* 5e49a85 fix: correct exponential backoff timing in webhook retry
* 0da511c test: add webhook integration tests
* 3e5ca98 feat: add retry logic for failed webhooks
* 740dbe1 feat: add webhook notification system
* 4e3f0e3 chore: bump to 1.2.1-SNAPSHOT for next dev cycle
* affc201 (tag: v1.2.0) release: v1.2.0
* edab757 test: add rate limiter load test
* 3be6121 fix: increase default rate limit from 100 to 1000 req/min
* 251e0c7 chore: bump to 1.2.0-SNAPSHOT for next dev cycle
* 9a879dd (tag: v1.1.0) release: v1.1.0
*   842b259 Merge feature/rate-limiting (#3)
|\  
| * 3294b93 docs: add rate limiting section to API docs
| * 2ddd9d6 feat: add rate limit headers to responses
| * 50d556a feat: add token-bucket rate limiter
* |   0869bcf Merge feature/csv-export (#2)
|\ \  
| |/  
|/|   
| * 20b4052 fix: escape commas in CSV values
| * 63e15b8 feat: add export download endpoint
| * fc406d9 feat: add CSV export service
|/  
* 42ce311 chore: bump to 1.1.0-SNAPSHOT for next dev cycle
* cc4b96a (tag: v1.0.0) release: v1.0.0
* 517e59e fix: reject empty password in login
*   e8607fc Merge feature/user-auth (#1)
|\  
| * d86fc90 test: add authentication unit tests
| * 92b8e04 feat: add password hashing utility
| * 444e1cc feat: add login endpoint with token generation
|/  
* f48cb3b chore: initial project setup
```

## Edge Cases Demonstrated

| Cycle | Scenario | What Happened | Key Insight |
|-------|----------|---------------|-------------|
| 1 | Normal release | Feature branch, merge, Dev, QA, Prod | Baseline happy path |
| 2 | Parallel development | 2 features, 2 developers | Pipeline handles multi-merge naturally |
| 3 | Production hotfix | Fix on main, same workflow | No special process needed |
| 4 | Failed QA build | Tests failed mid-workflow | Build before commit = no cleanup needed |
| 5 | Abandoned release | QA rejected v1.3.0 | Tag preserved for audit; one manual version edit |
| 6 | Production rollback | Re-deployed older release | Any past release re-deployable via SHA-256 |
| 7 | Major version bump | Breaking change (feat!) | Release notes auto-detect; operator picks major bump |

## Key Observations

1. **The same workflow handled** normal releases, hotfixes, rollbacks, and major bumps
2. **Failed builds are non-events** -- build before commit means nothing to clean up
3. **Abandoned releases leave an audit trail** but require one manual version edit
4. **Rollbacks deploy older GitHub Releases** -- the binary is proven identical via SHA-256
5. **Breaking changes** are signaled by Conventional Commits feat! and major bump type
6. **Version gaps are normal** -- Prod went v1.2.1 -> v1.4.0 -> v1.2.1 (rollback) -> v1.5.0 -> v1.6.0
7. **Zero rebuilds for any Prod deployment** -- including rollbacks, always the exact QA binary
8. **Only 1 manual version edit** in 7 cycles (the abandoned release scenario)

---
*Simulation ran against a real git repository with 50 commits and 8 tags.*
