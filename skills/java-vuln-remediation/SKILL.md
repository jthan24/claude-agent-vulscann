---
name: java-vuln-remediation
description: Team rules to find, prove and fix security vulnerabilities in a Spring Boot repository that builds with Maven and Gradle, and to open one pull request per vulnerability. Use it in every run of the vulnerability patcher, before the first finding.
---

# Java vulnerability remediation — team rules

## 1. Threshold

- Open a pull request for CRITICAL and HIGH findings only.
- Put MEDIUM and LOW findings in the report, with the fix you suggest. Do not open a pull request for them.
- Severity of a CVE: the CVSS v3 or v4 base score from OSV. 9.0-10.0 CRITICAL, 7.0-8.9 HIGH, 4.0-6.9 MEDIUM, lower LOW. When OSV gives only a label (`database_specific.severity`), use the label.
- Severity of a code finding: the impact if an anonymous user on the network uses it. Remote data read, code execution or authentication bypass is HIGH or CRITICAL. Give the reason in one sentence in the report.

## 2. Collect the findings

Dependencies:

```bash
cd /workspace/spring-petclinic
./mvnw -B -q dependency:list -DincludeScope=runtime -DoutputFile=/tmp/deps.txt -DexcludeTransitive=false
```

Make one OSV batch query for all `group:artifact:version` lines:

```bash
curl -s https://api.osv.dev/v1/querybatch -d @/tmp/osv-query.json   # {"queries":[{"package":{"ecosystem":"Maven","name":"g:a"},"version":"v"}, ...]}
curl -s https://api.osv.dev/v1/vulns/<OSV-or-GHSA-id>                # details, aliases (CVE), affected ranges, fixed version
```

Use the CVE alias as the finding ID when it exists, else the GHSA ID.

Container images (`docker-compose.yml`, `k8s/*.yml`, any `Dockerfile`): an image with no tag or with `latest` is a finding (`IMG-<name>`). Severity MEDIUM, because it is a supply chain risk and not a known CVE.

Source code: follow the skill `security-audit`. Read only `src/main/**`, `src/main/resources/**`, the build files and the deployment files. Use the ID `SAST-<short-slug>`, for example `SAST-actuator-exposure`.

## 3. Branch names

`security/<ID>`, for example `security/CVE-2026-12345` or `security/SAST-actuator-exposure`. The branch name is the memory of the agent: a branch or pull request with this name, open or closed, means "already handled".

## 4. The remediation loop (one finding at a time)

1. `git checkout main && git pull --ff-only && git checkout -b security/<ID>` (local branch only).
2. Write the test first, in `src/test/java/org/springframework/samples/petclinic/security/<Name>SecurityTests.java`:
   - Code finding: a behavior test. Example: `MockMvc` GET `/actuator/env` and expect 404 or 401.
   - Dependency CVE: prefer a behavior test that uses the vulnerable function. When that is not possible in a unit test, assert the version that is on the classpath at runtime: read `META-INF/maven/<group>/<artifact>/pom.properties` from the class loader and assert that the version is equal to or higher than the first fixed version from OSV. Put the CVE ID and the fixed version in the test name and in a comment.
3. Run the test only: `./mvnw -B -q test -Dtest=<Name>SecurityTests`. It must FAIL. Save the failure lines for the pull request. If it passes, the test does not prove the finding: write a better test, or report the finding as "Not fixed".
4. Apply the smallest fix that removes the finding:
   - Dependency: change the version in `pom.xml` AND in `build.gradle`, because the repository builds with both and they must stay the same. Prefer a property or a managed version override over a new explicit dependency. A major version jump is allowed when the tests pass; write "BREAKING" in the pull request title when you change a major version.
   - Code or configuration: change only the lines that cause the finding.
5. Run the test again. It must PASS.
6. Run both full builds. Both must pass:
   ```bash
   ./mvnw -B -q verify
   ./gradlew --no-daemon -q build
   ```
   If the format check fails, run `./mvnw -B -q spring-javaformat:apply` and build again.
7. Commit on the local branch: `security: fix <ID> in <component>`.
8. Publish (not in a dry run): create the remote branch `security/<ID>` from `main`, push the changed files (full content of each file) to that branch, then open the pull request against `main` with the template in section 5.
9. Check out `main` again before the next finding.

## 5. Pull request template

Title: `security: fix <ID> in <component>` (add `[BREAKING]` at the start for a major version jump).

```markdown
## Vulnerability
- ID: <ID> (aliases: <GHSA / CVE>)
- Severity: <CRITICAL|HIGH> (CVSS <score>) · Confidence: <high|medium>
- Component: <group:artifact:version | file:line>
- Summary: <one or two sentences>

## Proof (test first)
Test: `<path to the test>`
Before the fix — FAIL:
<3 to 10 lines of the failure output>
After the fix — PASS.

## Fix
<What changed and why this is the smallest change.>

## Verification
- `./mvnw verify`: PASS
- `./gradlew build`: PASS

## Overlap
<"None", or: "The same change also fixes <ID>, <ID> (pull requests #N, #M). Merge one and close the others.">

---
Opened by the agent `petclinic-vuln-patcher`. A person must review this pull request before merge.
```

## 6. Overlap between pull requests

The team wants one pull request per CVE. When one version change fixes more than one CVE, open one pull request per CVE. Each has its own test, and each says in "Overlap" which other pull requests contain the same version change. Open them in order of severity, so that the reviewer starts with the worst one.
