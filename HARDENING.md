<!-- markdownlint-disable -->

# Hardening Report: SvanBoxel--delete-merged-branch/1.4.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **SvanBoxel--delete-merged-branch/1.4.3** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference actions using mutable tags or branch names instead of full 40-character commit SHAs, making them vulnerable to supply-chain attacks if the referenced tag is moved or the branch is compromised.

- codeql-analysis.yml: actions/checkout@v2, github/codeql-action/init@v1, github/codeql-action/autobuild@v1, github/codeql-action/analyze@v1
- e2e-test.yml: actions/checkout@v2.3.2, actions/github-script@v3 (×3)
- publish-release.yml: actions/checkout@v1 (×3), actions/setup-node@v1, sonarsource/sonarcloud-github-action@master

Locations:

- `.github/workflows/codeql-analysis.yml:20`
- `.github/workflows/codeql-analysis.yml:28`
- `.github/workflows/codeql-analysis.yml:31`
- `.github/workflows/codeql-analysis.yml:34`
- `.github/workflows/e2e-test.yml:29`
- `.github/workflows/e2e-test.yml:40`
- `.github/workflows/e2e-test.yml:50`
- `.github/workflows/e2e-test.yml:62`
- `.github/workflows/publish-release.yml:17`
- `.github/workflows/publish-release.yml:19`
- `.github/workflows/publish-release.yml:33`
- `.github/workflows/publish-release.yml:36`
- `.github/workflows/publish-release.yml:46`

### script-injection (severity: high)

Rule (a): The 'Create commit' run: block in e2e-test.yml directly interpolates ${{ github.sha }} into shell command strings. Even though github.sha is not directly attacker-controlled, any ${{ ... }} expression inside a run: block flows through YAML template substitution before the shell processes it, creating a script-injection risk. Offending lines include:
  SHA='${{ github.sha }}'
  echo '${{ github.sha }}' >> README.md
  git commit -m "Testing delete-merged-branch bot sha ${{ github.sha }}"

Locations:

- `.github/workflows/e2e-test.yml:33`

### github-env-injection (severity: high)

The 'Create commit' run: block in e2e-test.yml writes a value derived from ${{ github.sha }} to the GitHub environment using the legacy ::set-env workflow command without sanitization. The variable $SHORT_SHA is derived from ${{ github.sha }} (an untrusted-input expression) and written via: echo "::set-env name=short_sha::$SHORT_SHA" — no printf '%s' ... | tr -d '\n\r' sanitization is applied before the write.

Locations:

- `.github/workflows/e2e-test.yml:42`

### missing-permissions (severity: medium)

None of the three workflow files define a top-level 'permissions:' key, and no individual job within them defines job-level permissions. Without explicit permissions, workflows run with the default (often broad) token permissions, violating the principle of least privilege.

- codeql-analysis.yml: no permissions defined
- e2e-test.yml: no permissions defined
- publish-release.yml: no permissions defined

Locations:

- `.github/workflows/codeql-analysis.yml:1`
- `.github/workflows/e2e-test.yml:1`
- `.github/workflows/publish-release.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all four findings across three workflow files:

1. **unpinned-uses**: Pinned all 13 action references to full 40-char SHAs with tag comments preserved:
   - actions/checkout@v2 → @0717577d45739eb3c851188b29f50ed6c0b2194e
   - actions/checkout@v2.3.2 → @2036a08e25fa78bbd946711a407b529a0a1204bf
   - actions/checkout@v1 → @50fbc622fc4ef5163becd7fab6573eac35f8462e
   - github/codeql-action/{init,autobuild,analyze}@v1 → @231aa2c8a89117b126725a0e11897209b7118144
   - actions/github-script@v3 → @ffc2c79a5b2490bd33e0a41c1de74b877714d736
   - actions/setup-node@v1 → @f1f314fca9dfce2769ece7d933488f076716723e
   - sonarsource/sonarcloud-github-action@master → @ba3875ecf642b2129de2b589510c81a8b53dbf4e

2. **script-injection**: Moved ${{ github.sha }} out of the run: shell string into the step's env: block as GITHUB_SHA_VALUE; shell script now uses $GITHUB_SHA_VALUE. Also updated github-script steps to use process.env.* instead of ${{ env.* }} template interpolation.

3. **github-env-injection**: Replaced legacy ::set-env command with modern $GITHUB_ENV file approach, with sanitization via printf '%s' "$SHORT_SHA" | tr -d '\n\r' before writing.

4. **missing-permissions**: Added top-level permissions blocks to all three files with minimal required permissions (contents: read for all; security-events: write added for codeql-analysis.yml to allow uploading scan results).

