---
name: "artificial-intelligence-open-source"
description: >
  Analyze open-source projects for sustainability risks and apply AI-driven interventions
  for bug triaging, community health assessment, vulnerability detection, contributor
  onboarding, and maintenance automation. Trigger phrases: "analyze OSS health",
  "assess project sustainability", "triage issues automatically", "detect community smells",
  "onboard new contributors", "audit OSS security posture"
---

# AI-Driven Open Source Sustainability Analysis

This skill enables Claude to audit open-source software projects for sustainability risks and recommend concrete AI-augmented interventions. Drawing from a systematic literature review of AI applications in OSS (Karim, Lu & Goggins, 2026), it operationalizes six intervention domains: automated bug triaging, community health analytics, vulnerability detection, contributor onboarding pipelines, maintenance automation, and environmental impact assessment. The core principle is treating AI as a cooperative augmentation layer for human infrastructure, not a replacement.

## When to Use

- When the user asks to **assess the health or sustainability** of an open-source project (e.g., "Is this project at risk of abandonment?")
- When the user wants to **triage a backlog of issues** and prioritize them by severity, staleness, or contributor fit
- When the user asks to **detect community smells** -- organizational anti-patterns like bus factor, toxic communication, or contributor attrition
- When the user wants to **set up onboarding automation** for new contributors (task recommendation, mentor matching, good-first-issue labeling)
- When the user asks to **audit an OSS codebase for security vulnerabilities** with a sustainability lens (detection coverage, patch velocity, risk hotspots)
- When the user wants to **automate maintenance tasks** like dependency updates, stale issue cleanup, or release note generation
- When the user asks to **evaluate environmental impact** of AI tooling used in their OSS workflow

## Key Technique: Multi-Domain Sustainability Assessment

The paper synthesizes research across six sustainability challenge domains, each mapped to specific AI techniques:

1. **Bug fixing and maintenance**: LLM-based patch generation combined with neural machine translation for code repair. The key insight is chaining detection-to-patch-to-review as an automated pipeline rather than treating each step in isolation. Defect prediction models identify high-risk modules before bugs manifest, reducing reactive maintenance burden.

2. **Community health analytics**: Machine learning models analyze trace data from repositories, issue trackers, and communication platforms to compute health metrics. Tools like YOSHI map project activity to community behavioral patterns, while csDetector identifies "community smells" -- organizational dysfunction indicators like information silos, lone wolves, or radio silence. Sentiment analysis on discussion threads detects toxicity or disengagement early.

3. **Vulnerability detection and security**: AI-driven static and dynamic analysis, NLP-based code scanning, and predictive risk modeling work together. The critical differentiator from standard SAST/DAST is the predictive layer -- identifying which code regions are likely to develop vulnerabilities based on change patterns, contributor experience, and historical defect density. Explainable AI (XAI) is emphasized so developers understand *why* code is flagged.

## Step-by-Step Workflow

1. **Gather project metadata**: Clone the repository (or read its structure) and collect key signals: commit frequency over time, contributor count and distribution, open/closed issue ratio, PR merge times, dependency freshness, and CI/CD status.

2. **Compute community health indicators**: Analyze contributor activity patterns to identify bus factor (how many contributors account for 80%+ of commits), attrition rate (contributors who stopped in the last 6 months), and response latency on issues/PRs. Flag projects where bus factor < 3 or attrition exceeds 50%.

3. **Detect community smells**: Scan for organizational anti-patterns:
   - **Lone Wolf**: Single contributor making >60% of commits
   - **Radio Silence**: Median issue response time > 30 days
   - **Organizational Silo**: Disjoint contributor clusters with no cross-collaboration
   - **Black Cloud**: Concentrated negative sentiment from specific participants

4. **Triage issue backlog**: Classify open issues by type (bug, feature, documentation, security), estimate severity from description and labels, identify stale issues (no activity > 90 days), and recommend prioritization. Tag issues suitable for newcomers based on complexity signals (file count, code churn, test coverage of affected areas).

5. **Audit security posture**: Scan for known vulnerabilities in dependencies (CVE databases), identify code hotspots with high churn and low test coverage (vulnerability-prone zones), check for security policy presence (SECURITY.md, responsible disclosure process), and assess patch velocity (time from CVE disclosure to fix merge).

6. **Evaluate onboarding readiness**: Check for CONTRIBUTING.md quality, good-first-issue label usage, mentorship program indicators, documentation coverage, and development environment setup automation. Score the project's newcomer friendliness.

7. **Generate sustainability report**: Produce a structured assessment with risk scores per domain, specific findings with evidence, and prioritized recommendations. Each recommendation maps to a concrete action (script, CI workflow, bot configuration, or process change).

8. **Recommend AI-augmented interventions**: For each identified risk, suggest specific tooling: issue labeling bots, automated dependency update workflows, sentiment analysis integrations, contributor matching systems, or security scanning pipelines. Provide implementation sketches, not just tool names.

9. **Assess environmental footprint**: If the project uses AI tooling (Copilot, CI-integrated ML models, LLM-based bots), estimate computational overhead and suggest efficiency measures like parameter-efficient fine-tuning, caching, or batched inference.

## Concrete Examples

**Example 1: Full project sustainability audit**

User: "Analyze the health of this open-source project and tell me if it's at risk"

Approach:
1. Read repository structure, recent commit history, contributor list, and issue tracker
2. Compute health metrics via git log analysis
3. Identify community smells and risk factors
4. Generate prioritized sustainability report

Output:
```markdown
## OSS Sustainability Report: example-project

### Risk Level: MODERATE (Score: 58/100)

### Community Health
- **Bus Factor: 2** (critical -- alice and bob account for 87% of commits)
- **Contributor Attrition: 40%** (6 of 15 contributors inactive >6 months)
- **Issue Response Time: 12 days median** (acceptable)
- **PR Merge Time: 4 days median** (healthy)

### Community Smells Detected
- [HIGH] **Lone Wolf**: alice authored 62% of commits in last 12 months
- [MEDIUM] **Organizational Silo**: frontend/ and backend/ directories have
  zero contributor overlap

### Security Posture
- 3 outdated dependencies with known CVEs (2 moderate, 1 high)
- No SECURITY.md or disclosure policy found
- Patch velocity: N/A (no prior security fixes tracked)

### Onboarding Readiness: LOW
- CONTRIBUTING.md exists but lacks setup instructions
- 0 issues labeled "good-first-issue" in last 6 months
- No automated dev environment (devcontainer, Nix, etc.)

### Recommended Interventions
1. **Immediate**: Update 3 vulnerable dependencies and add SECURITY.md
2. **Short-term**: Label 5-10 issues as good-first-issue; add devcontainer
3. **Medium-term**: Set up Dependabot/Renovate for automated dependency PRs
4. **Structural**: Recruit 1-2 maintainers for frontend/ to reduce bus factor
```

**Example 2: Automated issue triage**

User: "Help me triage these 200+ open issues and figure out what to prioritize"

Approach:
1. Fetch all open issues with metadata (labels, age, comments, assignees)
2. Classify by type and severity using title/description analysis
3. Identify stale, duplicate, and newcomer-friendly issues
4. Produce prioritized triage report

Output:
```markdown
## Issue Triage Report (214 open issues)

### By Priority
- **Critical (5)**: #189 (security), #201 (data loss), #145 (crash), #198 (auth bypass), #203 (regression)
- **High (23)**: Performance regressions, broken integrations, accessibility blockers
- **Medium (87)**: Feature requests with community demand, non-critical bugs
- **Low (99)**: Cosmetic issues, edge cases, nice-to-haves

### Stale Issues (no activity >90 days): 67
- Recommend closing 31 (resolved or no longer relevant)
- Recommend pinging 22 (potentially valid, need reproduction)
- Recommend labeling 14 as "help-wanted"

### Good First Issues (newcomer-friendly): 18 candidates
- Issues touching <3 files, well-documented behavior, existing test coverage
- Examples: #156 (typo in error message), #171 (add input validation),
  #183 (update deprecated API call)

### Duplicates Detected: 11 clusters
- #102 / #134 / #178 (same rendering bug, different reports)
- #88 / #165 (same feature request with different framing)
```

**Example 3: Security-focused sustainability audit**

User: "Audit this project's security posture and set up automated vulnerability detection"

Approach:
1. Scan dependencies for known CVEs
2. Identify code hotspots (high churn + low test coverage)
3. Check security policies and disclosure process
4. Generate CI workflow for continuous scanning

Output:
```markdown
## Security Sustainability Audit

### Dependency Vulnerabilities
| Package       | Current | Fixed   | Severity | CVE            |
|---------------|---------|---------|----------|----------------|
| lodash        | 4.17.19 | 4.17.21 | High     | CVE-2021-23337 |
| axios         | 0.21.0  | 0.21.1  | Moderate | CVE-2021-3749  |
| node-fetch    | 2.6.0   | 2.6.7   | Moderate | CVE-2022-0235  |

### Code Risk Hotspots (high churn + low coverage)
- src/auth/session.js -- 47 commits, 12% test coverage
- src/api/handlers.js -- 38 commits, 23% test coverage
- lib/crypto/utils.js -- 22 commits, 0% test coverage

### Missing Security Infrastructure
- [ ] No SECURITY.md
- [ ] No automated dependency scanning in CI
- [ ] No branch protection on main
- [ ] No signed commits required

### Recommended CI Workflow (GitHub Actions)
```yaml
# .github/workflows/security-scan.yml
name: Security Scan
on: [push, pull_request, schedule]
# ... (generates complete workflow file)
```
```

## Best Practices

- **Do**: Analyze quantitative signals (commit frequency, issue velocity, contributor distribution) before making qualitative judgments about project health.
- **Do**: Present findings with evidence -- cite specific issue numbers, commit hashes, or contributor counts rather than vague assessments.
- **Do**: Frame AI interventions as augmenting maintainer capacity, not replacing human judgment. Automated triage suggests priority; humans decide.
- **Do**: Check for community smells as leading indicators -- organizational dysfunction predicts technical debt and abandonment before code metrics do.
- **Avoid**: Reducing sustainability to a single score. Always show the multi-dimensional breakdown (community, security, maintenance, onboarding).
- **Avoid**: Recommending heavy AI tooling for small projects. A 2-maintainer project needs a CONTRIBUTING.md and good-first-issue labels, not an ML-powered triage bot.

## Error Handling

- **Insufficient git history**: If the repository has fewer than 50 commits or less than 6 months of history, note that trend analysis is unreliable. Fall back to snapshot metrics (current state) rather than trajectory analysis.
- **Private or restricted issue trackers**: If issues are not publicly accessible, work with available signals (commits, PR activity, documentation) and explicitly note the gap in the report.
- **Monorepo or multi-project repositories**: Scope analysis to the relevant subdirectory. Contributor metrics for a monorepo are misleading if not filtered by path.
- **No CI/CD configuration detected**: Flag this as a sustainability risk in itself. Recommend minimum viable CI (linting, tests, dependency scanning) as a priority intervention.
- **Conflicting signals**: When metrics diverge (e.g., high commit activity but rising issue backlog), surface both signals and let the user interpret. Do not smooth over contradictions.

## Limitations

- **Community sentiment analysis is approximate**: Without access to real-time communication channels (Slack, Discord, mailing lists), assessment relies on public GitHub/GitLab data, which captures only a fraction of community dynamics.
- **Predictive accuracy degrades for small projects**: Health metrics and attrition models are most reliable for projects with 10+ regular contributors. For smaller projects, individual contributor decisions dominate and are unpredictable.
- **Security scanning is not exhaustive**: This approach identifies known CVEs and code risk hotspots but does not replace dedicated penetration testing or formal security audits.
- **Bias in historical data**: Contributor metrics may reflect systemic biases (timezone, language, institutional affiliation) rather than actual contribution quality. Flag this caveat when analyzing contributor distribution.
- **Snapshot vs. trajectory**: A single-point audit captures current state but may miss trends. Recommend periodic re-assessment for projects where sustainability is a concern.

## Reference

**Paper**: Karim, S. M. R. U., Lu, W., & Goggins, S. (2026). *Artificial Intelligence in Open Source Software Engineering: A Foundation for Sustainability*. [arXiv:2602.07071v1](https://arxiv.org/abs/2602.07071v1)

**What to look for**: Table 1 maps sustainability challenges to AI techniques with specific research references. Table 2 provides a comparative analysis of AI technique strengths and limitations per domain. The community health analytics section (csDetector, YOSHI) and the ethical concerns framework are the most directly actionable for building assessment tooling.