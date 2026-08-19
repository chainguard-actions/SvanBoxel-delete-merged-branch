<!-- markdownlint-disable -->

# Hardening Report: SvanBoxel--delete-merged-branch/1.4.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **SvanBoxel--delete-merged-branch/1.4.2** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference actions using mutable tags or branch names instead of immutable 40-character commit SHAs, making them vulnerable to supply-chain attacks if the referenced tag or branch is moved.

- codeql-analysis.yml: `actions/checkout@v2`, `github/codeql-action/init@v1`, `github/codeql-action/autobuild@v1`, `github/codeql-action/analyze@v1`
- e2e-test.yml: `actions/checkout@v2.3.2`, `actions/github-script@v3` (used 3 times)
- publish-release.yml: `actions/checkout@v1` (used 3 times), `actions/setup-node@v1`, `sonarsource/sonarcloud-github-action@master`

Locations:

- `.github/workflows/codeql-analysis.yml:23`
- `.github/workflows/codeql-analysis.yml:32`
- `.github/workflows/codeql-analysis.yml:35`
- `.github/workflows/codeql-analysis.yml:38`
- `.github/workflows/e2e-test.yml:29`
- `.github/workflows/e2e-test.yml:55`
- `.github/workflows/e2e-test.yml:66`
- `.github/workflows/e2e-test.yml:77`
- `.github/workflows/publish-release.yml:17`
- `.github/workflows/publish-release.yml:19`
- `.github/workflows/publish-release.yml:34`
- `.github/workflows/publish-release.yml:36`
- `.github/workflows/publish-release.yml:44`

### missing-permissions (severity: medium)

None of the three workflow files define a top-level `permissions:` block, and no individual job within them defines job-level permissions either. Without explicit permissions, workflows run with the default (often broad) token permissions, violating the principle of least privilege.

Locations:

- `.github/workflows/codeql-analysis.yml:1`
- `.github/workflows/e2e-test.yml:1`
- `.github/workflows/publish-release.yml:1`

### script-injection (severity: high)

Sub-rule (a): The 'Create commit' step in e2e-test.yml directly interpolates `${{ github.sha }}` inside a `run:` shell script. GitHub Actions performs YAML template substitution before the shell ever sees the string, so any value containing shell metacharacters could alter the command being executed. Offending lines:
  `SHA='${{ github.sha }}'`
  `echo '${{ github.sha }}' >> README.md`
  `git commit -m "Testing delete-merged-branch bot sha ${{ github.sha }}"`
These should be moved to an `env:` block and referenced as a quoted shell variable (e.g. `"$GITHUB_SHA"`).

Locations:

- `.github/workflows/e2e-test.yml:33`

### github-env-injection (severity: high)

The 'Create commit' step in e2e-test.yml writes to GITHUB_ENV via the legacy `::set-env` workflow command without sanitizing the value first. The variable `SHORT_SHA` is derived from `${{ github.sha }}` (interpolated directly into the shell script) and then written with `echo "::set-env name=short_sha::$SHORT_SHA"`. No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the write, allowing a newline-containing value to inject additional environment variables.

Locations:

- `.github/workflows/e2e-test.yml:42`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all four findings across three workflow files:

1. **unpinned-uses**: Pinned all action references to full 40-char SHAs with tag comments:
   - codeql-analysis.yml: actions/checkout@v2 → 0717577d..., github/codeql-action/{init,autobuild,analyze}@v1 → 231aa2c8...
   - e2e-test.yml: actions/checkout@v2.3.2 → 2036a08e..., actions/github-script@v3 → ffc2c79a... (3 uses)
   - publish-release.yml: actions/checkout@v1 → 50fbc622... (3 uses), actions/setup-node@v1 → f1f314fc..., sonarsource/sonarcloud-github-action@master → ba3875ec...

2. **missing-permissions**: Added top-level `permissions:` blocks to all three files. codeql-analysis.yml gets `contents: read` + `security-events: write` (needed for CodeQL to upload SARIF results); the other two get `contents: read`.

3. **script-injection**: In e2e-test.yml's 'Create commit' step, moved `${{ github.sha }}` to an `env:` block as `GIT_SHA` and replaced all three inline interpolations with `"$GIT_SHA"` shell variable references.

4. **github-env-injection**: Replaced the legacy `::set-env` command with the modern `>> "$GITHUB_ENV"` approach, and sanitized the value with `printf '%s' "$SHORT_SHA" | tr -d '\n\r'` before writing.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings:
1. e2e-test.yml (lines 49, 63, 76): Replaced all `${{ env.* }}` template expressions interpolated directly into JavaScript string literals in three `actions/github-script` steps with `process.env.*` references. The env vars (owner, repo, short_sha, pull_number) are already available in the Node.js process environment at runtime, so reading them via `process.env` is safe and avoids template-engine substitution before script execution.
2. publish-release.yml (line 57): Added double quotes around `$GCP_TOKEN` in the `echo` command (`echo "$GCP_TOKEN"`) to prevent shell metacharacter injection from the secret value.

