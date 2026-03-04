# Revised Pipeline: QA Approval Gate + Prod Release Notes

---

## Why This Is Better

The original design had a problem: deploy-qa.yml completes immediately after deployment,
showing a green checkmark in GitHub Actions while QA testing hasn't even started.
The pipeline says "done" when the work is only half finished.

Moving the approval gate inside deploy-qa.yml and release notes to release-prod.yml
gives us a cleaner separation of concerns:

```
deploy-qa.yml   = 'Is this code ready for production?'
                   Deploys, tests, WAITS for human sign-off.
                   Only completes when QA says 'yes.'

release-prod.yml = 'Ship it.'
                   Generates the release record, deploys the proven binary.
                   The release notes describe what went to prod, not what
                   went to QA.
```

Two concrete benefits:

1. **The QA workflow status is meaningful.** A green deploy-qa.yml means the code
   has been tested AND approved, not just deployed. A yellow (in-progress) means
   QA testing is underway. This is visible to everyone on the team.
2. **Release notes belong to the production event.** They describe what was released
   to production, when, and from which tag. If a QA release is abandoned (never
   promoted), no release notes are generated — because nothing was released.

---

# Revised Pipeline Architecture

```
  Developer merges to main
       |
       v  (automatic)
  [deploy-dev.yml]   (unchanged)
  Build SNAPSHOT, deploy to Dev
       |
       |  Dev lead triggers QA workflow
       v  (manual dispatch)
  [deploy-qa.yml]    (REVISED)
  |  1. Strip SNAPSHOT
  |  2. Build + unit tests
  |  3. Generate SHA-256 checksums
  |  4. Deploy to QA server
  |  5. Run automated regression suite
  |  6. (if auto-tests fail: ABORT — no commit, no tag)
  |
  |  --- WORKFLOW PAUSES HERE ---
  |
  |  GitHub Environment: 'qa-approved'
  |  Required reviewers: QA Lead, BA Lead
  |  (Manual regression testing happens during this wait)
  |  (BA smoke testing happens during this wait)
  |  Status in GitHub Actions: 'Waiting for review'
  |
  |  QA Lead clicks [Approve]
  |
  |  7. Commit version change
  |  8. Tag vX.Y.Z
  |  9. Push commit + tag
  | 10. Create GitHub Release (jar + checksums, NO release notes yet)
  | 11. Publish to GitHub Packages
       |
       |  Release manager triggers Prod workflow
       v  (manual dispatch)
  [release-prod.yml]  (REVISED)
  |  1. Generate release notes (git log since previous tag)
  |  2. Download jar from GitHub Release
  |  3. Verify SHA-256 checksum
  |  4. Deploy to Prod
  |  5. Update GitHub Release with release notes
  |  6. Write deploy-info.json
  |  7. Bump to next SNAPSHOT
  |  8. Commit + push to main
       |
       v  (automatic)
  [deploy-dev.yml]
  Next cycle begins
```

---

# How the Environment Approval Works

GitHub Actions has a built-in feature for this: **Environment protection rules**.
When a workflow job references an environment with required reviewers, the workflow
pauses and waits — it does not consume runner minutes while waiting.

The trick is splitting deploy-qa.yml into **two jobs**: one that deploys and tests,
and a second that waits for approval before tagging.

```
deploy-qa.yml

  Job 1: 'deploy-and-test'
    environment: qa                  (no protection rules, just secrets)
    -> strip SNAPSHOT, build, deploy, auto-tests
    -> uploads jar as workflow artifact

  Job 2: 'tag-and-release'  (depends on Job 1)
    environment: qa-approved         (REQUIRES QA Lead / BA Lead approval)
    -> waits for human review
    -> after approval: commit, tag, push, GitHub Release
```

The workflow appears in GitHub Actions as:

```
  deploy-and-test      [completed ✅]     2 min ago
  tag-and-release       [waiting ⏳]      Waiting for review from @qa-lead, @ba-lead
```

This is exactly what you want: the team can see at a glance whether QA testing
is in progress (waiting) or complete (approved and green).

---

# Revised deploy-qa.yml

```yaml
name: Deploy to QA (Release Candidate)

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Confirm release to QA'
        required: true
        type: boolean
        default: false

permissions:
  contents: write
  packages: write

jobs:
  # ── Job 1: Build, deploy, and run automated tests ──
  deploy-and-test:
    runs-on: ubuntu-latest
    environment: qa
    if: \${{ inputs.confirm }}
    outputs:
      release-version: \${{ steps.ver.outputs.rel }}
      snapshot-version: \${{ steps.ver.outputs.snap }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: \${{ secrets.GITHUB_TOKEN }}

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - uses: gradle/actions/setup-gradle@v4

      - name: Read and validate version
        id: ver
        run: |
          SNAP=\$(grep "^version=" version.properties | cut -d= -f2)
          if [[ "\$SNAP" != *-SNAPSHOT ]]; then
            echo "::error::Expected SNAPSHOT. Got: \$SNAP"; exit 1
          fi
          REL="\${SNAP%-SNAPSHOT}"
          echo "snap=\$SNAP" >> "\$GITHUB_OUTPUT"
          echo "rel=\$REL" >> "\$GITHUB_OUTPUT"

      - name: Strip SNAPSHOT
        run: |
          sed -i "s/^version=.*/version=\${{ steps.ver.outputs.rel }}/" \
            version.properties

      - name: Build and test
        run: ./gradlew clean build test

      - name: Generate checksums
        run: |
          cd build/libs
          sha256sum *.jar > ../checksums.txt

      - name: Deploy to QA server
        run: |
          V="\${{ steps.ver.outputs.rel }}"
          JAR="build/libs/\${{ github.event.repository.name }}-\${V}.jar"
          scp "\$JAR" \${{ secrets.QA_USER }}@\${{ secrets.QA_HOST }}:/opt/app/
          ssh \${{ secrets.QA_USER }}@\${{ secrets.QA_HOST }} << DEPLOY
            cat > /opt/app/deploy-info.json << EOF
            {
              "version": "\$V",
              "environment": "qa",
              "deployed_at": "\$(date -u +%Y-%m-%dT%H:%M:%SZ)",
              "commit": "\${{ github.sha }}",
              "status": "testing"
            }
            EOF
            sudo systemctl restart app
          DEPLOY

      - name: Run automated regression suite
        run: |
          echo 'Waiting 30s for app startup...'
          sleep 30
          ./gradlew integrationTest -Penv=qa \
            -PbaseUrl=http://\${{ secrets.QA_HOST }}:8080
        # If this step fails, the entire workflow stops.
        # No tag, no release, no commit. Clean failure.

      - name: Upload release artifact
        uses: actions/upload-artifact@v4
        with:
          name: release-\${{ steps.ver.outputs.rel }}
          path: |
            build/libs/*.jar
            build/checksums.txt
            version.properties

      - name: Notify team
        run: |
          curl -X POST \${{ secrets.SLACK_WEBHOOK }} -H "Content-type: application/json" \
            -d '{"text":"v${{ steps.ver.outputs.rel }} deployed to QA. Automated tests passed. Manual regression and BA smoke testing can begin. Approve in GitHub Actions when ready."}'

  # ── Job 2: Wait for QA approval, then tag and release ──
  tag-and-release:
    needs: deploy-and-test
    runs-on: ubuntu-latest
    environment: qa-approved          # <-- THIS IS THE APPROVAL GATE
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: \${{ secrets.GITHUB_TOKEN }}

      - name: Download release artifact
        uses: actions/download-artifact@v4
        with:
          name: release-\${{ needs.deploy-and-test.outputs.release-version }}
          path: ./release-artifacts/

      - name: Apply version change
        run: |
          cp release-artifacts/version.properties version.properties

      - name: Commit, tag, and push
        run: |
          V="\${{ needs.deploy-and-test.outputs.release-version }}"
          git config user.name 'github-actions[bot]'
          git config user.email 'github-actions[bot]@users.noreply.github.com'
          git add version.properties
          git commit -m "release: v${V}"
          git tag -a "v${V}" -m "Release v${V}"
          git push origin main --tags

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: v\${{ needs.deploy-and-test.outputs.release-version }}
          name: v\${{ needs.deploy-and-test.outputs.release-version }}
          body: |
            Release candidate approved by QA.
            Release notes will be generated at production deployment.
          files: |
            release-artifacts/*.jar
            release-artifacts/checksums.txt

      - name: Publish to GitHub Packages
        run: ./gradlew publish
        env:
          GITHUB_TOKEN: \${{ secrets.GITHUB_TOKEN }}
```

---

# Revised release-prod.yml

```yaml
name: Release to Production

on:
  workflow_dispatch:
    inputs:
      release_tag:
        description: 'Tag to promote (e.g. v1.0.0)'
        required: true
        type: string
      bump_type:
        description: 'Next version bump'
        required: true
        type: choice
        options: [minor, patch, major]
        default: minor

permissions:
  contents: write

jobs:
  promote-to-prod:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0                    # full history for release notes
          token: \${{ secrets.GITHUB_TOKEN }}

      - name: Parse version and compute next
        id: ver
        run: |
          TAG="\${{ inputs.release_tag }}"
          V="\${TAG#v}"
          echo "rel=\$V" >> "\$GITHUB_OUTPUT"
          IFS="." read -r MA MI PA <<< "\$V"
          case "\${{ inputs.bump_type }}" in
            major) NX="\$((MA+1)).0.0" ;;
            minor) NX="\${MA}.\$((MI+1)).0" ;;
            patch) NX="\${MA}.\${MI}.\$((PA+1))" ;;
          esac
          echo "next=\${NX}-SNAPSHOT" >> "\$GITHUB_OUTPUT"

      # ── NEW: Generate release notes ──
      - name: Generate release notes
        run: |
          V="\${{ steps.ver.outputs.rel }}"
          # Find the tag before this one
          PREV_TAG=\$(git tag --sort=-v:refname | grep -v "v\${V}" | head -1)
          RANGE="\${PREV_TAG:+\${PREV_TAG}..}\${{ inputs.release_tag }}"

          echo "# Release Notes - v${V}" > release-notes.md
          echo "" >> release-notes.md
          echo "**Release Date:** $(date +%Y-%m-%d)" >> release-notes.md
          echo "**Previous Release:** ${PREV_TAG:-(initial)}" >> release-notes.md
          echo "" >> release-notes.md

          # Categorise commits
          for CATEGORY in "feat:New Features" "fix:Bug Fixes" "docs:Documentation" \
                          "refactor:Refactoring" "test:Tests" "ci:CI/Build" \
                          "chore:Maintenance"; do
            PREFIX="\${CATEGORY%%:*}"
            TITLE="\${CATEGORY##*:}"
            COMMITS=\$(git log \$RANGE --oneline --no-merges --grep="^\$PREFIX" || true)
            if [[ -n "\$COMMITS" ]]; then
              echo "## $TITLE" >> release-notes.md
              echo "$COMMITS" | while read -r line; do
                echo "- $line" >> release-notes.md
              done
              echo "" >> release-notes.md
            fi
          done

          echo '---' >> release-notes.md
          echo '*Auto-generated at production release.*' >> release-notes.md
          echo 'Release notes generated:'
          cat release-notes.md

      - name: Download release artifacts
        run: |
          gh release download \${{ inputs.release_tag }} \
            --pattern '*.jar' --pattern 'checksums.txt' \
            --dir ./artifacts/
        env:
          GH_TOKEN: \${{ secrets.GITHUB_TOKEN }}

      - name: Verify SHA-256 checksum
        run: |
          cd artifacts
          sha256sum --check checksums.txt
          echo 'Checksum verified - binary matches QA build'

      - name: Deploy to Prod
        run: |
          V="\${{ steps.ver.outputs.rel }}"
          JAR=\$(ls artifacts/*.jar | head -1)
          scp "\$JAR" \${{ secrets.PROD_USER }}@\${{ secrets.PROD_HOST }}:/opt/app/
          ssh \${{ secrets.PROD_USER }}@\${{ secrets.PROD_HOST }} << DEPLOY
            cat > /opt/app/deploy-info.json << EOF
            {
              "version": "\$V",
              "environment": "production",
              "deployed_at": "\$(date -u +%Y-%m-%dT%H:%M:%SZ)",
              "tag": "\${{ inputs.release_tag }}",
              "promoted_from": "qa"
            }
            EOF
            sudo systemctl restart app
          DEPLOY

      # ── NEW: Update GitHub Release with release notes ──
      - name: Update GitHub Release with release notes
        run: |
          gh release edit \${{ inputs.release_tag }} \
            --notes-file release-notes.md
        env:
          GH_TOKEN: \${{ secrets.GITHUB_TOKEN }}

      - name: Bump to next SNAPSHOT
        run: |
          NEXT="\${{ steps.ver.outputs.next }}"
          sed -i "s/^version=.*/version=\${NEXT}/" version.properties
          git config user.name 'github-actions[bot]'
          git config user.email 'github-actions[bot]@users.noreply.github.com'
          git add version.properties
          git commit -m "chore: bump to \${NEXT} for next dev cycle"
          git push origin main

      - name: Summary
        run: |
          echo '## Production Release' >> $GITHUB_STEP_SUMMARY
          echo '' >> $GITHUB_STEP_SUMMARY
          cat release-notes.md >> $GITHUB_STEP_SUMMARY
```

---

# GitHub Environment Setup

You need two environments configured in GitHub Settings:

```
GitHub repo -> Settings -> Environments

Environment: 'qa'
  Required reviewers:  (none)
  Deployment branches: main
  Secrets: QA_USER, QA_HOST, SLACK_WEBHOOK
  Purpose: provides QA server credentials to Job 1

Environment: 'qa-approved'
  Required reviewers:  [x] Enable
    - @qa-lead
    - @ba-lead
    (any ONE can approve)
  Wait timer:  0 minutes
  Deployment branches: main
  Purpose: pauses the workflow until QA/BA sign off

Environment: 'production'
  Required reviewers:  (optional, already gated by qa-approved)
  Deployment branches: main
  Secrets: PROD_USER, PROD_HOST
```

---

# Simulation: Revised Pipeline in Action

## Monday 09:00 — Feature merged

```bash
git merge feature/payment-api --no-ff -m "Merge feature/payment-api (#12)"
```

```
AUTOMATIC: deploy-dev.yml -> myapp-1.0.0-SNAPSHOT.jar on Dev
```

## Monday 14:00 — Dev lead triggers deploy-qa.yml

```
WORKFLOW: deploy-qa.yml

  Job 1: deploy-and-test  [running]
    Strip SNAPSHOT -> 1.0.0
    Build + unit tests -> PASS
    Generate SHA-256 checksum
    Deploy to QA server
    Automated regression suite:
      PaymentFlowTest           PASS
      StripeWebhookTest         PASS
      AuthRegressionTest        PASS
      ExportRegressionTest      PASS
      147 tests, 0 failures
    Upload artifact for Job 2
    Slack: 'v1.0.0 on QA. Auto-tests passed. Manual testing can begin.'

  Job 1: deploy-and-test  [completed ✅]  14:04

  Job 2: tag-and-release  [waiting ⏳]
    Environment: qa-approved
    Waiting for review from @qa-lead, @ba-lead...
```

At this point everyone can see in GitHub Actions that the QA workflow is
**in progress** (yellow dot). The team knows testing is underway.

## Monday 14:30–Tuesday 16:00 — Manual QA testing

```
QA team works through test plan on QA server (myapp-1.0.0.jar):

  [x] Payment: create payment, verify confirmation
  [x] Payment: handle Stripe webhook
  [x] Payment: duplicate event idempotency
  [x] Auth: login, token refresh, logout
  [x] Export: CSV download, date filter
  [x] Rate limit: 1000 req/min threshold

BA smoke testing on QA server:

  [x] End-to-end: signup -> payment -> confirmation
  [x] Payment in admin dashboard
  [x] Stripe reconciliation
  [x] Email notifications

QA Lead:   All critical paths pass. Minor typo logged for next release.
BA Lead:   Business flows confirmed.
```

## Tuesday 16:30 — QA Lead approves in GitHub Actions

```
GitHub Actions UI:

  deploy-qa.yml > tag-and-release

  +----------------------------------------------------------+
  |  Review required for 'qa-approved' environment            |
  |                                                            |
  |  @qa-lead: 'Regression complete. All critical paths pass.' |
  |  @qa-lead: [Approve ✅]                                    |
  +----------------------------------------------------------+

  Job 2: tag-and-release  [running]
    Checkout code
    Download release artifact from Job 1
    Apply version.properties (version=1.0.0)
    Commit 'release: v1.0.0'
    Tag v1.0.0
    Push commit + tag
    Create GitHub Release v1.0.0 (jar + checksums, no notes yet)
    Publish to GitHub Packages

  Job 2: tag-and-release  [completed ✅]  16:32

ENTIRE WORKFLOW: deploy-qa.yml  [completed ✅]
```

The deploy-qa.yml workflow now shows a **green checkmark** — this means the release
candidate has been built, tested (both automated and manual), and approved.

## Wednesday 09:00 — Release manager triggers release-prod.yml

```
WORKFLOW: release-prod.yml
  Inputs: release_tag=v1.0.0, bump_type=minor

  Generate release notes (git log since initial commit):

    # Release Notes - v1.0.0
    **Release Date:** 2026-03-05
    **Previous Release:** (initial)

    ## New Features
    - feat: add Stripe payment integration (Luigi)
    - feat: add payment confirmation webhook (Luigi)

    ## Bug Fixes
    - fix: handle duplicate payment events (Luigi)

    ## Tests
    - test: add payment flow integration tests (Luigi)

  Download v1.0.0 from GitHub Release
  SHA-256 verify -> OK
  Deploy to Prod server
  Update GitHub Release with release notes
  Bump version.properties -> 1.1.0-SNAPSHOT
  Commit + push to main
```

```
SERVER STATE:
  Dev:  myapp-1.1.0-SNAPSHOT.jar  (next cycle auto-started)
  QA:   myapp-1.0.0.jar           (tested + approved)
  Prod: myapp-1.0.0.jar           (SHA-256 verified, notes attached)
```

The GitHub Release for v1.0.0 now shows the release notes — they were
written at production release time, not when the candidate was built.

---

# What If QA Rejects?

If the QA lead clicks **Reject** instead of Approve:

```
GitHub Actions UI:

  @qa-lead: 'Payment webhook fails under load. Needs rework.'
  @qa-lead: [Reject ❌]

  Job 2: tag-and-release  [failed ❌]  (rejected by reviewer)
  ENTIRE WORKFLOW: deploy-qa.yml  [failed ❌]
```

The result:

- No tag was created
- No GitHub Release exists
- No version change was committed
- version.properties on main is still X.Y.Z-SNAPSHOT
- The QA server has the candidate jar, but that is harmless
- The developer fixes the issue, pushes to main, and the cycle starts over

This is cleaner than the original design because the rejection is recorded
in GitHub Actions with the reviewer's comment. The failed workflow is the
audit trail.

---

# Edge Case: What About the Artifact Timeout?

GitHub Actions workflow artifacts (from upload-artifact) have a retention period
(default 90 days). Job 2 downloads the artifact from Job 1. If QA testing takes
longer than the retention period, Job 2 would fail.

```
Practical impact:  Negligible.
  If your QA cycle takes more than 90 days, you have bigger problems.
  For safety, set retention-days: 30 in the upload step.
  If it expires, just re-trigger deploy-qa.yml.
```

---

# Edge Case: New Commits During QA Testing

What if developers push to main while QA is testing?

```
Timeline:
  Monday:    deploy-qa.yml triggers, deploys 1.0.0 candidate to QA
  Tuesday:   developer pushes a new commit to main
             deploy-dev.yml triggers, Dev gets newer SNAPSHOT
             QA server is UNAFFECTED (already has the candidate jar)
  Wednesday: QA approves, Job 2 runs
             Job 2 checks out main — which now has the new commit
             BUT: it uses the artifact from Job 1 (the tested jar)
             AND: version.properties comes from the artifact, not main

Risk: The git commit and tag are created from a newer main that
      includes untested code in the commit history.

Mitigation: Job 2 should checkout the EXACT commit that Job 1 used.
```

Add this to Job 1's outputs:

```yaml
  - name: Record commit SHA
    id: sha
    run: echo "sha=\$(git rev-parse HEAD)" >> "\$GITHUB_OUTPUT"
```

And in Job 2:

```yaml
  - uses: actions/checkout@v4
    with:
      ref: \${{ needs.deploy-and-test.outputs.commit-sha }}
      token: \${{ secrets.GITHUB_TOKEN }}
```

This ensures the tag points to the exact code that was tested, even if
main has moved forward.

---

# Summary: What Changed


| Component | Original | Revised | Why |
|-----------|----------|---------|-----|
| deploy-qa.yml | Single job, completes immediately | Two jobs: deploy+test, then approval gate | Workflow status reflects actual QA state |
| QA approval | Implicit (who triggers Prod) | Explicit (GitHub Environment review) | Auditable sign-off with reviewer comments |
| Release notes | Generated at QA release time | Generated at Prod release time | Notes belong to the production event |
| GitHub Release body | Full notes at creation | Placeholder at creation, updated at Prod | Notes only exist for versions that shipped |
| release-prod.yml | Download + verify + deploy | Generate notes + download + verify + deploy + update release | Owns the release record |
| Rejection handling | Abandoned release, manual cleanup | Reviewer clicks Reject, workflow fails cleanly | Audit trail, no orphaned tags |
| ci.yml | Unchanged | Unchanged | - |
| deploy-dev.yml | Unchanged | Unchanged | - |

---
*The approval gate belongs in the QA pipeline because that is where the approval*
*decision is made. Release notes belong in the Prod pipeline because that is when*
*the release happens.*
