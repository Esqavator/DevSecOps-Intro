# Lab 10 — DefectDojo Capstone

## Environment

DefectDojo was deployed locally using Docker Compose.

- DefectDojo version: `2.58.2-alpine`
- Platform: `linux/arm64`
- URL: `http://localhost:8080`
- Product: `OWASP Juice Shop`
- Product ID: `1`
- Engagement: `Course Semester Run`
- Engagement ID: `1`
- Engagement type: `CI/CD`
- Engagement status: `In Progress`

The initial deployment failed because the Compose configuration used
`defectdojo/defectdojo-django:latest` together with the source code of
DefectDojo `2.58.2`.

This caused a dependency mismatch:

```text
ModuleNotFoundError: No module named 'social_django'
```

The problem was fixed by pinning the DefectDojo images:

```text
defectdojo/defectdojo-django:2.58.2-alpine
defectdojo/defectdojo-nginx:2.58.2-alpine
```

After that:

- the initializer completed with exit code `0`;
- the DefectDojo UI returned HTTP `200`;
- the containers used native `linux/arm64` images.

## Imported security reports

The following tests were created in DefectDojo:

| Test ID | Lab | Test | Scan type |
|---:|---:|---|---|
| 1 | 4 | Grype from SBOM | Anchore Grype |
| 2 | 4 | Trivy filesystem or image scan | Trivy Scan |
| 3 | 5 | Semgrep SAST | Semgrep JSON Report |
| 5 | 6 | Checkov Terraform | Checkov Scan |
| 6 | 6 | KICS Ansible | KICS Scan |
| 7 | 6 | KICS Pulumi | KICS Scan |
| 8 | 7 | Trivy container image | Trivy Scan |
| 9 | 7 | Trivy Kubernetes | Trivy Operator Scan |
| 11 | 5 | ZAP authenticated DAST | ZAP Scan |

A total of nine reports were imported using seven different scan types:

1. Anchore Grype
2. Trivy Scan
3. Semgrep JSON Report
4. ZAP Scan
5. Checkov Scan
6. KICS Scan
7. Trivy Operator Scan

## ZAP report conversion

The ZAP report from Lab 5 was generated in JSON format, while the
DefectDojo `ZAP Scan` parser expected XML.

The first import attempt returned:

```text
Internal error: Wrong file format, please use xml.
```

The report was converted from ZAP JSON into the XML structure expected by
the parser:

```text
OWASPZAPReport
└── site
    └── alerts
        └── alertitem
            └── instances
                └── instance
```

The converted report contained:

- 4 sites;
- 12 alerts;
- 42 request and response instances.

The corrected ZAP import completed with HTTP status `201` and created 12
findings.

The imported ZAP findings included:

- SQL Injection — High;
- Content Security Policy Header Not Set — Medium;
- Cross-Domain Misconfiguration — Medium;
- Missing Anti-clickjacking Header — Medium;
- Session ID in URL Rewrite — Medium;
- Private IP Disclosure — Low;
- Timestamp Disclosure — Low;
- X-Content-Type-Options Header Missing — Low.

## Findings summary

Before deduplication:

- Total findings: `397`
- Active findings: `397`
- Duplicate findings: `0`

After enabling and running deduplication:

- Total findings: `397`
- Primary findings: `349`
- Duplicate findings: `48`
- Active findings: `349`
- Inactive findings: `48`

After creating one Risk Acceptance:

- Total findings: `397`
- Active findings: `348`
- Inactive findings: `49`
- Primary findings: `349`
- Duplicate findings: `48`
- Risk accepted findings: `1`

## Active primary findings by severity

| Severity | Count |
|---|---:|
| Critical | 12 |
| High | 122 |
| Medium | 172 |
| Low | 29 |
| Info | 13 |
| **Total** | **348** |

## Deduplication

DefectDojo deduplication was initially disabled:

```text
enable_deduplication = False
```

The setting was enabled:

```text
enable_deduplication = True
false_positive_history = False
```

Backlog deduplication was then executed for the `Trivy Scan` parser.

The final result was:

```text
total: 397
duplicates: 48
primary: 349
active: 349
inactive: 48
```

### Deduplication example

`CVE-2026-45447` was detected in `libssl3t64` version
`3.5.5-1~deb13u2` by both Anchore Grype and Trivy.

| Finding ID | Test ID | Scanner | Duplicate | Primary finding |
|---:|---:|---|---|---:|
| 12 | 1 | Anchore Grype | No | — |
| 119 | 2 | Trivy Scan | No | — |
| 336 | 8 | Trivy Scan | Yes | 119 |

The two Trivy findings had the same hash:

```text
7d19791d5a04c3ee77d1d2db7741aef3bbeef024e934248078a2355ffe31c200
```

Finding `336` was marked as a duplicate of finding `119` and was
deactivated.

The Grype finding remained separate because its scanner-specific hash was
different:

```text
3902bc74c8c49dc8eb67afc7e1f5ccc519a355a4971054d2ed721988e5bb569d
```

This demonstrates same-parser deduplication between two Trivy tests. The
cross-tool Grype result remains separate because its calculated hash differs.

## Risk Acceptance

Risk Acceptance was created for:

- Finding ID: `393`
- Title: `X-Content-Type-Options Header Missing`
- Severity: `Low`
- Scanner: `ZAP Scan`
- Endpoint: `http://juice-shop:3000/socket.io/`
- Name: `Temporary acceptance for training environment`

Decision:

```text
Accept
```

Security recommendation:

```text
Fix
```

Decision justification:

> The application is intentionally vulnerable and is used only in an
> isolated local training environment. The risk is temporarily accepted for
> the duration of the course lab. This configuration must not be used in
> production.

Recommended remediation:

```text
Configure the application or web server to return the
X-Content-Type-Options: nosniff header on all responses.
```

Expiration date:

```text
2026-12-15
```

After Risk Acceptance was saved, the finding had the following state:

```json
{
  "id": 393,
  "title": "X-Content-Type-Options Header Missing",
  "severity": "Low",
  "active": false,
  "risk_accepted": true,
  "is_mitigated": false,
  "duplicate": false,
  "test": 11
}
```

## Top 10 active primary findings

| ID | Severity | Finding | Component | Version | Vulnerability ID | Test |
|---:|---|---|---|---|---|---:|
| 2 | Critical | GHSA-c7hr-j4mj-j2w6 in jsonwebtoken:0.1.0 | jsonwebtoken | 0.1.0 | GHSA-c7hr-j4mj-j2w6 | 1 |
| 3 | Critical | GHSA-c7hr-j4mj-j2w6 in jsonwebtoken:0.4.0 | jsonwebtoken | 0.4.0 | GHSA-c7hr-j4mj-j2w6 | 1 |
| 5 | Critical | GHSA-jf85-cpcp-j695 in lodash:2.4.2 | lodash | 2.4.2 | GHSA-jf85-cpcp-j695 | 1 |
| 22 | Critical | GHSA-xwcq-pm8m-c4vf in crypto-js:3.3.0 | crypto-js | 3.3.0 | GHSA-xwcq-pm8m-c4vf | 1 |
| 36 | Critical | CVE-2026-5450 in libc6:2.41-12+deb13u2 | libc6 | 2.41-12+deb13u2 | CVE-2026-5450 | 1 |
| 70 | Critical | CVE-2026-34182 in libssl3t64:3.5.5-1~deb13u2 | libssl3t64 | 3.5.5-1~deb13u2 | CVE-2026-34182 | 1 |
| 98 | Critical | GHSA-5mrr-rgp6-x4gr in marsdb:0.6.11 | marsdb | 0.6.11 | GHSA-5mrr-rgp6-x4gr | 1 |
| 139 | Critical | CVE-2023-46233 Crypto-Js 3.3.0 | crypto-js | 3.3.0 | CVE-2023-46233 | 2 |
| 146 | Critical | CVE-2015-9235 Jsonwebtoken 0.1.0 | jsonwebtoken | 0.1.0 | CVE-2015-9235 | 2 |
| 151 | Critical | CVE-2015-9235 Jsonwebtoken 0.4.0 | jsonwebtoken | 0.4.0 | CVE-2015-9235 | 2 |

## Reports not imported as normal vulnerability scans

### Falco

Falco output from Lab 9 was reviewed separately as runtime security
evidence.

The available file was a Falco runtime log rather than a vulnerability
report supported by one of the installed DefectDojo parsers.

It was therefore documented as runtime monitoring evidence and was not
artificially imported using an unrelated scan type.

### Cosign

Cosign verification proves container image signature and supply-chain
integrity. It is not a vulnerability scan and does not normally create
vulnerability findings.

Cosign verification results were therefore treated as security evidence
rather than imported into DefectDojo as findings.

## Conclusion

The lab created a centralized vulnerability management workflow in
DefectDojo for:

- software composition analysis;
- container image scanning;
- static application security testing;
- dynamic application security testing;
- infrastructure-as-code scanning;
- Kubernetes security scanning.

The final result includes:

- nine imported reports;
- seven different scan types;
- 397 total findings;
- 48 duplicate findings;
- 349 primary findings;
- 348 active primary findings after Risk Acceptance;
- one documented Risk Acceptance;
- a verified same-parser deduplication example between two Trivy tests;
- a documented cross-tool overlap between Anchore Grype and Trivy.
