# Darkmoon Security Scan — Jenkins plugin

Run [Darkmoon](https://github.com/ASCIT31) AI penetration-test campaigns from a Jenkins
job or declarative pipeline, publish findings through **Warnings Next Generation**, and
gate the build on severity — against **Darkmoon OSS** (local CLI) or **Darkmoon Pro**
(REST API).

The plugin owns the Jenkins concerns (credentials, step configuration, result
publishing, artifact archiving, fail policy). It delegates the OSS/Pro contract entirely
to the portable [`darkmoon-ci`](https://github.com/ASCIT31) CLI, which it shells out to.

## Requirements

- Jenkins **2.479+** (parent POM 5.x, JDK 17+ runtime).
- The [`darkmoon-ci`](https://www.npmjs.com/package/@darkmoon/client) CLI on the
  build agent's `PATH`, plus Node.js 18+. Pin a compatible release
  (`npm i -g @darkmoon/client@^0.1`) so the plugin talks to a `1.x`-contract CLI.
- [Warnings Next Generation](https://plugins.jenkins.io/warnings-ng/) (installed as a
  dependency) to visualize/trend findings.
- Credentials plugin + Plain Credentials plugin (installed as dependencies) for secrets.

## Pipeline step: `darkmoonScan`

```groovy
node {
  darkmoonScan(
    target: 'http://juice-shop:3000',   // required — URL or host
    mode: 'oss',                        // auto | oss | pro   (default auto)
    failOn: 'high',                     // critical|high|medium|low|info|none (build FAILURE)
    unstableOn: 'none',                 // same set (build UNSTABLE) — only if FAILURE gate is not tripped
    credentialsId: 'darkmoon-token',    // Secret Text credential -> DARKMOON_PRO_TOKEN (Pro)
    // Pro:
    // apiUrl: 'https://darkmoon.example.com',
    // OSS backend (advanced):
    // ossDataDir: '/var/darkmoon/data', ossReportsDir: '/var/darkmoon/reports',
    recordIssues: true,                 // write SARIF for warnings-ng (default true)
    archiveReport: true,                // archive report + SARIF + CLI log (default true)
    includeFullReport: false,           // also archive the UN-redacted report (default false)
    timeout: 1800                       // seconds; 0 = CLI default
  )

  // Publish the SARIF the step produced. enabledForFailure:true is REQUIRED because
  // darkmoonScan may already have marked the build FAILURE via its fail policy.
  recordIssues(
    enabledForFailure: true,
    tools: [sarif(pattern: 'darkmoon-reports/darkmoon.sarif')]
  )
}
```

### What the step does

1. `darkmoon-ci run --target … --mode … --fail-on <csv> --json` — launch + wait for the
   campaign, read the fail-policy verdict, obtain the campaign id.
2. `darkmoon-ci findings <id> --json` — the normalized, **redaction-safe** findings, which
   the plugin maps to **SARIF 2.1.0** (`darkmoon-reports/darkmoon.sarif`).
3. `darkmoon-ci summary <id> --json` — authoritative severity counts.
4. `darkmoon-ci report <id> --out …` — the redacted report (and, if `includeFullReport`,
   the un-redacted one).
5. Prints a redaction-safe summary to the console, archives the report/SARIF/CLI-log as
   build artifacts, and applies the fail policy from the severity summary.

## Secrets

Configure a **Secret text** credential and pass its id as `credentialsId` (Pro token) and
optionally `licenseCredentialsId`. The plugin resolves it via the Jenkins Credentials API
and passes it to `darkmoon-ci` **through the environment only**
(`DARKMOON_PRO_TOKEN` / `DARKMOON_LICENSE`) — never on the command line, never in the
console, never in an archived artifact.

### Connecting to Darkmoon Pro

```groovy
node {
  darkmoonScan(
    target: 'https://staging.example.com',
    mode: 'pro',
    apiUrl: 'https://darkmoon.example.com',   // Pro REST base URL (https)
    credentialsId: 'darkmoon-token',          // Secret Text -> DARKMOON_PRO_TOKEN
    failOn: 'critical,high'
  )
  recordIssues(enabledForFailure: true, tools: [sarif(pattern: 'darkmoon-reports/darkmoon.sarif')])
}
```

Pro auth is **token-only** in the plugin: obtain a JWT out of band and store it as a
Secret Text credential. The interactive username/password + insecure-default
handshake exposed by `@darkmoon/client` is intentionally not surfaced here (a CI
job should carry a pre-issued token, not a password).

## Safety model (see THREAT-MODEL / §4)

- **Redaction-safe by default.** Only the redacted surface (titles, categories, CVSS,
  MITRE, endpoint, description, remediation) reaches the console, the SARIF and the
  warnings-ng UI. Evidence, raw requests/responses and extracted data are never emitted.
- **Full report is opt-in and internal.** `includeFullReport: true` archives the
  un-redacted report; it may contain rehydrated sensitive values, so restrict artifact
  access. Off by default.
- **Fail from findings, not exit code.** The gate is computed from the normalized severity
  summary, so it is identical for OSS and Pro.
- **One data dir per job (OSS).** OSS runs share a data dir; run one container /
  compose-project per CI job to avoid campaign mis-attribution.

## Build

```bash
# JDK 17+ (built/tested with JDK 21) and Maven 3.9.6+
mvn -B clean verify        # unit + JenkinsRule tests
mvn -B -DskipTests package # produces target/darkmoon-scan.hpi
```

Install `target/darkmoon-scan.hpi` via **Manage Jenkins → Plugins → Advanced → Deploy
Plugin**, or drop it into `$JENKINS_HOME/plugins/`.

See [`docs/CLI-CONTRACT.md`](docs/CLI-CONTRACT.md) for the exact `darkmoon-ci` interface
the plugin depends on, and [`docs/THREAT-MODEL.md`](docs/THREAT-MODEL.md) for the security
model. An end-to-end example lives in [`examples/Jenkinsfile`](examples/Jenkinsfile).

## License

MIT — see [LICENSE](LICENSE).
