# Simplified Pipeline: Three Independent Workflows

---

## Why This Is Simpler

The revised pipeline had a clever two-job structure inside `deploy-qa.yml` with a GitHub Environment approval gate that paused the workflow mid-flight. That's elegant engineering, but it introduces complexity that a small team doesn't need:

- Two jobs coordinating via workflow artifacts and output variables
- A `qa-approved` environment with reviewer rules that need configuration and maintenance
- Version commits, tagging, and GitHub Release creation happening inside the QA workflow — before anyone has decided to release to production
- The QA workflow "owns" too many concerns: building, deploying, waiting for humans, tagging, publishing

The simplified approach removes all of that. Each workflow does one thing. The human decision between QA and Prod is not a GitHub Environment gate — it's simply whether someone triggers `release-prod.yml` or not.

```
deploy-dev.yml    = Build and deploy to Dev            (unchanged)
deploy-qa.yml     = Fetch, validate, and deploy to QA  (simplified)
release-prod.yml  = Tag, release, and deploy to Prod   (owns all release concerns)
```

---

## Pipeline Architecture

```
  Developer merges to main
       |
       v  (automatic)
  [deploy-dev.yml]   (UNCHANGED)
  Build, test, publish to GitHub Packages, deploy to Dev
       |
       |  Dev lead triggers QA workflow
       v  (manual dispatch — input: version to fetch)
  [deploy-qa.yml]    (SIMPLIFIED)
  |  1. Fetch jar from GitHub Packages
  |  2. Verify SHA-256 checksum
  |  3. Validate version metadata
  |  4. Run automated test suite against the artifact
  |  5. Deploy to QA server
  |  6. Notify team: "ready for manual regression"
  |
  |  WORKFLOW COMPLETES HERE ✅
  |
  |  --- MANUAL PHASE (outside any pipeline) ---
  |
  |  BAs and QA run manual regression tests on the QA server.
  |
  |  Decision point (human, not automated):
  |    Pass → release manager triggers release-prod.yml
  |    Fail → developers fix, push to main, new dev build,
  |            re-trigger deploy-qa.yml with new version
       |
       v  (manual dispatch — input: version to release)
  [release-prod.yml]  (SIMPLIFIED)
  |  1. Fetch jar from GitHub Packages
  |  2. Verify SHA-256 checksum
  |  3. Create git tag vX.Y.Z
  |  4. Generate release notes (git log since previous tag)
  |  5. Create GitHub Release (jar + checksums + notes)
  |  6. Deploy to Prod server
  |  7. Bump version.properties to next SNAPSHOT
  |  8. Commit + push to main
       |
       v  (automatic)
  [deploy-dev.yml]
  Next cycle begins
```

---

## What Changed vs. the Revised Pipeline

| Aspect | Revised Pipeline | Simplified Pipeline |
|--------|-----------------|-------------------|
| deploy-qa.yml structure | Two jobs (deploy-and-test → tag-and-release) with environment approval gate | Single job: fetch, validate, deploy. Done. |
| QA approval mechanism | GitHub Environment `qa-approved` with required reviewers; workflow pauses mid-flight | No gate. Workflow completes after deployment. Human decides whether to trigger prod. |
| Who builds the artifact? | QA workflow strips SNAPSHOT, builds from source | Dev workflow builds and publishes to GitHub Packages. QA just fetches. |
| When does tagging happen? | After QA approval, inside deploy-qa.yml Job 2 | At production release time, inside release-prod.yml |
| When is the GitHub Release created? | After QA approval (placeholder body, updated at prod time) | At production release time, with final release notes from the start |
| GitHub Packages publish | After QA approval, inside deploy-qa.yml Job 2 | During dev build (already published before QA fetches) |
| Environments needed | `qa`, `qa-approved` (with reviewer rules), `production` | `qa` (secrets only), `production` (secrets only) |
| Rejection flow | Reviewer clicks Reject in GitHub UI; Job 2 fails | No formal rejection. Devs fix, push, re-trigger QA with new version. |
| Workflow artifacts | Job 1 uploads jar as workflow artifact for Job 2 | Not needed. GitHub Packages is the single artifact store. |
| Number of git commits per release | 2 (version strip at QA + version bump at prod) | 1 (version bump at prod) |

### What We Gained

1. **Fewer moving parts.** No environment approval gates, no two-job coordination, no workflow artifact handoff between jobs.
2. **Single artifact store.** GitHub Packages is the source of truth. Both QA and Prod fetch from the same place. No workflow artifacts.
3. **Clear ownership.** `deploy-qa.yml` deploys. `release-prod.yml` releases. No split responsibilities.
4. **Less GitHub configuration.** No `qa-approved` environment with reviewer rules. Just two environments with secrets.
5. **Release concerns consolidated.** Tagging, GitHub Release creation, release notes, and version bumping all happen in one place — `release-prod.yml`.

### What We Gave Up

1. **No in-workflow audit trail for QA approval.** The revised pipeline recorded who approved and when, directly in the GitHub Actions UI. In the simplified pipeline, the audit trail is simply "who triggered `release-prod.yml`."
2. **QA workflow status is less informative.** A green `deploy-qa.yml` means "deployed and automated tests passed," not "QA-approved." You lose the yellow "waiting for review" status.
3. **No formal rejection mechanism.** In the revised pipeline, a reviewer clicking Reject produced a clear failed workflow. Here, a failed QA cycle is just silence — nobody triggers prod.

For a small team where everyone is in the same room (or Slack channel), these trade-offs are worth it.

---

## deploy-qa.yml

```yaml
name: Deploy to QA

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to fetch from GitHub Packages (e.g. 1.0.0-SNAPSHOT)'
        required: true
        type: string

permissions:
  packages: read

jobs:
  deploy-to-qa:
    runs-on: ubuntu-latest
    environment: qa
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - uses: gradle/actions/setup-gradle@v4

      # ── Fetch from GitHub Packages ──
      - name: Download artifact from GitHub Packages
        run: |
          V="${{ inputs.version }}"
          REPO="${{ github.repository }}"
          ARTIFACT="${{ github.event.repository.name }}"

          echo "Fetching ${ARTIFACT}-${V}.jar from GitHub Packages..."

          # Use Maven coordinates to download from GitHub Packages
          mvn dependency:copy \
            -Dartifact=com.yourorg:${ARTIFACT}:${V}:jar \
            -DoutputDirectory=./artifacts/ \
            -DrepositoryId=github \
            -s settings.xml
          # settings.xml should contain GitHub Packages auth
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # ── Validate ──
      - name: Verify checksum
        run: |
          cd artifacts
          # Download the published checksums
          sha256sum --check checksums.txt
          echo "Checksum verified — artifact matches published build"

      - name: Validate version metadata
        run: |
          V="${{ inputs.version }}"
          JAR=$(ls artifacts/*.jar | head -1)
          # Extract version from jar manifest and compare
          MANIFEST_VER=$(unzip -p "$JAR" META-INF/MANIFEST.MF | grep 'Implementation-Version' | cut -d' ' -f2 | tr -d '\r')
          if [[ "$MANIFEST_VER" != "$V" ]]; then
            echo "::error::Version mismatch. Expected: $V, Got: $MANIFEST_VER"
            exit 1
          fi
          echo "Version verified: $V"

      # ── Automated testing ──
      - name: Run automated test suite
        run: |
          echo "Running automated tests against the fetched artifact..."
          # Start the jar locally or against a test environment
          java -jar artifacts/*.jar --server.port=8090 &
          APP_PID=$!
          sleep 30

          ./gradlew integrationTest -Penv=qa \
            -PbaseUrl=http://localhost:8090

          kill $APP_PID || true
        # If tests fail, workflow stops here. No deployment.

      # ── Deploy ──
      - name: Deploy to QA server
        run: |
          V="${{ inputs.version }}"
          JAR=$(ls artifacts/*.jar | head -1)
          scp "$JAR" ${{ secrets.QA_USER }}@${{ secrets.QA_HOST }}:/opt/app/
          ssh ${{ secrets.QA_USER }}@${{ secrets.QA_HOST }} << DEPLOY
            cat > /opt/app/deploy-info.json << EOF
            {
              "version": "$V",
              "environment": "qa",
              "deployed_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
              "commit": "${{ github.sha }}",
              "status": "testing"
            }
            EOF
            sudo systemctl restart app
          DEPLOY

      - name: Notify team
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H "Content-type: application/json" \
            -d '{"text":"v${{ inputs.version }} deployed to QA. Automated tests passed. Ready for manual regression testing."}'
```

---

## release-prod.yml

```yaml
name: Release to Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to release (e.g. 1.0.0-SNAPSHOT → will tag as v1.0.0)'
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
  packages: read

jobs:
  release-to-prod:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Compute release and next versions
        id: ver
        run: |
          RAW="${{ inputs.version }}"
          # Strip -SNAPSHOT suffix if present
          REL="${RAW%-SNAPSHOT}"
          echo "rel=$REL" >> "$GITHUB_OUTPUT"

          IFS="." read -r MA MI PA <<< "$REL"
          case "${{ inputs.bump_type }}" in
            major) NX="$((MA+1)).0.0" ;;
            minor) NX="${MA}.$((MI+1)).0" ;;
            patch) NX="${MA}.${MI}.$((PA+1))" ;;
          esac
          echo "next=${NX}-SNAPSHOT" >> "$GITHUB_OUTPUT"

      # ── Fetch from GitHub Packages ──
      - name: Download artifact from GitHub Packages
        run: |
          V="${{ inputs.version }}"
          ARTIFACT="${{ github.event.repository.name }}"
          mvn dependency:copy \
            -Dartifact=com.yourorg:${ARTIFACT}:${V}:jar \
            -DoutputDirectory=./artifacts/ \
            -DrepositoryId=github \
            -s settings.xml
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Verify SHA-256 checksum
        run: |
          cd artifacts
          sha256sum --check checksums.txt
          echo "Checksum verified — binary matches QA-tested build"

      # ── Tag ──
      - name: Create git tag
        run: |
          V="${{ steps.ver.outputs.rel }}"
          git config user.name 'github-actions[bot]'
          git config user.email 'github-actions[bot]@users.noreply.github.com'
          git tag -a "v${V}" -m "Release v${V}"
          git push origin "v${V}"

      # ── Release notes ──
      - name: Generate release notes
        run: |
          V="${{ steps.ver.outputs.rel }}"
          PREV_TAG=$(git tag --sort=-v:refname | grep -v "v${V}" | head -1)
          RANGE="${PREV_TAG:+${PREV_TAG}..}v${V}"

          echo "# Release Notes - v${V}" > release-notes.md
          echo "" >> release-notes.md
          echo "**Release Date:** $(date +%Y-%m-%d)" >> release-notes.md
          echo "**Previous Release:** ${PREV_TAG:-(initial)}" >> release-notes.md
          echo "" >> release-notes.md

          for CATEGORY in "feat:New Features" "fix:Bug Fixes" "docs:Documentation" \
                          "refactor:Refactoring" "test:Tests" "ci:CI/Build" \
                          "chore:Maintenance"; do
            PREFIX="${CATEGORY%%:*}"
            TITLE="${CATEGORY##*:}"
            COMMITS=$(git log $RANGE --oneline --no-merges --grep="^$PREFIX" || true)
            if [[ -n "$COMMITS" ]]; then
              echo "## $TITLE" >> release-notes.md
              echo "$COMMITS" | while read -r line; do
                echo "- $line" >> release-notes.md
              done
              echo "" >> release-notes.md
            fi
          done

          echo '---' >> release-notes.md
          echo '*Auto-generated at production release.*' >> release-notes.md
          cat release-notes.md

      # ── GitHub Release ──
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: v${{ steps.ver.outputs.rel }}
          name: v${{ steps.ver.outputs.rel }}
          body_path: release-notes.md
          files: |
            artifacts/*.jar

      # ── Deploy ──
      - name: Deploy to Prod
        run: |
          V="${{ steps.ver.outputs.rel }}"
          JAR=$(ls artifacts/*.jar | head -1)
          scp "$JAR" ${{ secrets.PROD_USER }}@${{ secrets.PROD_HOST }}:/opt/app/
          ssh ${{ secrets.PROD_USER }}@${{ secrets.PROD_HOST }} << DEPLOY
            cat > /opt/app/deploy-info.json << EOF
            {
              "version": "$V",
              "environment": "production",
              "deployed_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
              "tag": "v${V}",
              "promoted_from": "qa"
            }
            EOF
            sudo systemctl restart app
          DEPLOY

      # ── Bump version for next cycle ──
      - name: Bump to next SNAPSHOT
        run: |
          NEXT="${{ steps.ver.outputs.next }}"
          sed -i "s/^version=.*/version=${NEXT}/" version.properties
          git add version.properties
          git commit -m "chore: bump to ${NEXT} for next dev cycle"
          git push origin main

      - name: Summary
        run: |
          echo '## Production Release' >> $GITHUB_STEP_SUMMARY
          echo '' >> $GITHUB_STEP_SUMMARY
          cat release-notes.md >> $GITHUB_STEP_SUMMARY
```

---

## GitHub Environment Setup

Only two environments needed (down from three):

```
GitHub repo → Settings → Environments

Environment: 'qa'
  Required reviewers:  (none)
  Deployment branches: main
  Secrets: QA_USER, QA_HOST, SLACK_WEBHOOK
  Purpose: provides QA server credentials

Environment: 'production'
  Required reviewers:  (none — the gate is "who can trigger the workflow")
  Deployment branches: main
  Secrets: PROD_USER, PROD_HOST
  Purpose: provides Prod server credentials
```

If you want an additional layer of protection on production, you can restrict who has permission to trigger `release-prod.yml` via GitHub repository roles rather than adding environment reviewers. For a small team, controlling who can click "Run workflow" is sufficient.

---

## Simulation: Simplified Pipeline in Action

### Monday 09:00 — Feature merged

```
Developer merges feature/payment-api to main.
AUTOMATIC: deploy-dev.yml → builds myapp-1.0.0-SNAPSHOT.jar
           → publishes to GitHub Packages
           → deploys to Dev server
```

### Monday 14:00 — Dev lead triggers deploy-qa.yml

```
Input: version = 1.0.0-SNAPSHOT

WORKFLOW: deploy-qa.yml  [running]
  Fetch 1.0.0-SNAPSHOT from GitHub Packages
  Verify SHA-256 checksum        → OK
  Validate version metadata      → OK
  Automated test suite:
    PaymentFlowTest              PASS
    StripeWebhookTest            PASS
    AuthRegressionTest           PASS
    147 tests, 0 failures
  Deploy to QA server
  Slack: "v1.0.0-SNAPSHOT deployed to QA. Ready for manual regression."

WORKFLOW: deploy-qa.yml  [completed ✅]  14:06
```

### Monday 14:30 – Tuesday 16:00 — Manual regression

```
BAs and QA run manual tests on the QA server.
This happens completely outside of any GitHub Actions workflow.

Result: All critical paths pass.
```

### Wednesday 09:00 — Release manager triggers release-prod.yml

```
Input: version = 1.0.0-SNAPSHOT, bump_type = minor

WORKFLOW: release-prod.yml  [running]
  Compute release version: 1.0.0 (stripped SNAPSHOT)
  Compute next version: 1.1.0-SNAPSHOT
  Fetch 1.0.0-SNAPSHOT from GitHub Packages
  Verify SHA-256 checksum                  → OK
  Create git tag v1.0.0
  Generate release notes
  Create GitHub Release v1.0.0 (jar + notes)
  Deploy to Prod server
  Bump version.properties → 1.1.0-SNAPSHOT
  Commit + push to main

WORKFLOW: release-prod.yml  [completed ✅]  09:04
```

```
SERVER STATE:
  Dev:  myapp-1.1.0-SNAPSHOT.jar  (next cycle auto-started)
  QA:   myapp-1.0.0-SNAPSHOT.jar  (tested + approved by BAs)
  Prod: myapp-1.0.0-SNAPSHOT.jar  (same binary, verified by checksum)

GitHub:
  Tag: v1.0.0
  Release: v1.0.0 with jar, release notes
```

### What If QA Fails?

```
BAs find a bug during manual regression.
  → Developer fixes the issue, pushes to main.
  → deploy-dev.yml triggers automatically (new SNAPSHOT build).
  → Dev lead re-triggers deploy-qa.yml with the same version.
  → Cycle repeats.

No cleanup needed. No orphaned tags. No failed approval gates.
The old QA deployment is simply overwritten by the new one.
```

---

## Key Design Decisions

### Why QA fetches instead of building

In the revised pipeline, the QA workflow checked out the source, stripped the SNAPSHOT suffix, and built from scratch. This means the artifact deployed to QA was built in a different context (different workflow run, potentially different runner) than anything published to GitHub Packages.

In the simplified pipeline, the dev workflow builds once and publishes. Both QA and Prod fetch the exact same binary from GitHub Packages. The checksum verification confirms this. One build, verified everywhere.

### Why there's no approval gate

The revised pipeline used a GitHub Environment with required reviewers to create a formal approval step inside the QA workflow. This is a solid pattern for larger teams or regulated environments where you need an auditable sign-off.

For a small team, the approval gate is the decision to trigger `release-prod.yml`. If nobody triggers it, the release doesn't happen. The audit trail is the GitHub Actions run history: who triggered the prod workflow and when.

### Why release concerns belong in one place

The revised pipeline split release concerns across two workflows: QA created the tag and GitHub Release (with a placeholder body), then Prod updated the release with notes. This meant a QA-approved artifact that never shipped to production would still have a tag and a GitHub Release — which is misleading.

In the simplified pipeline, `release-prod.yml` owns everything: tagging, release creation, notes generation, and deployment. If a version never makes it to production, no release artifacts exist. Clean.

---

## Prerequisites

For the QA and Prod workflows to fetch from GitHub Packages, the dev workflow (`deploy-dev.yml`) must publish the built artifact with checksums. Ensure your Gradle `publish` task is configured to upload both the jar and a `checksums.txt` file to GitHub Packages.

Your `build.gradle` should include something like:

```groovy
publishing {
    publications {
        maven(MavenPublication) {
            from components.java
            artifact("build/checksums.txt") {
                classifier = 'checksums'
                extension = 'txt'
            }
        }
    }
    repositories {
        maven {
            name = "GitHubPackages"
            url = uri("https://maven.pkg.github.com/${System.getenv('GITHUB_REPOSITORY')}")
            credentials {
                username = System.getenv("GITHUB_ACTOR")
                password = System.getenv("GITHUB_TOKEN")
            }
        }
    }
}
```

---

*Three workflows, three jobs, three concerns. Dev builds. QA validates. Prod releases.*
