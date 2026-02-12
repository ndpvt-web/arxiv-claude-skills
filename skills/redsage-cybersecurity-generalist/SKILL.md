---
name: "redsage-cybersecurity-generalist"
description: |
  Apply RedSage's agentic augmentation methodology to cybersecurity assistance: structured threat analysis, vulnerability assessment, tool-command generation, and multi-turn security workflows grounded in MITRE ATT&CK, OWASP, CWE/CAPEC, and penetration testing frameworks.
  Trigger phrases:
  - "Analyze this CVE and map it to CWE/CAPEC"
  - "Help me with penetration testing methodology"
  - "Explain this MITRE ATT&CK technique"
  - "Generate security tool commands for this scenario"
  - "Assess this vulnerability and estimate CVSS"
  - "Walk me through incident response for this alert"
---

# RedSage Cybersecurity Generalist

This skill enables Claude to act as a domain-aware cybersecurity assistant using the structured methodology from RedSage (ICLR 2026). Rather than providing surface-level security advice, it applies RedSage's Planner-Augmenter workflow: first decomposing a cybersecurity problem into its constituent skill domains (vulnerability analysis, tool-command generation, threat intelligence, offensive techniques), then constructing grounded, multi-step responses that mirror expert decision-making. This approach produces responses that are technically precise, framework-aligned (MITRE ATT&CK, OWASP, CWE/CAPEC, NIST), and operationally actionable.

## When to Use

- When the user asks to analyze a CVE, map it to CWE identifiers, or predict CVSS severity
- When building penetration testing workflows that require specific tool commands (nmap, metasploit, burp suite, gobuster, etc.)
- When the user needs to understand or operationalize a MITRE ATT&CK technique or tactic
- When performing code review for security vulnerabilities (OWASP Top 10, injection flaws, auth bypass)
- When the user asks for incident response procedures for a specific alert or IOC
- When generating or reviewing Content Security Policy, firewall rules, or hardening configurations
- When the user needs threat actor attribution analysis or CTI report synthesis
- When writing CTF challenge solutions that require structured offensive methodology

## Key Technique: Agentic Augmentation with Planner-Augmenter Decomposition

RedSage's core innovation is a two-agent pipeline that transforms raw cybersecurity knowledge into structured, expert-grade multi-turn workflows. The **Planner** agent analyzes the security context and derives candidate skill sets (e.g., "this requires vulnerability analysis + tool-command generation + CWE mapping") along with augmentation strategies that specify the depth, format, and transformation approach. The **Augmenter** agent then instantiates each plan into a grounded, role-based response enforcing five quality dimensions: relevance to the specific security context, diversity of perspectives (attacker and defender), creativity in technique chaining, technical detail preservation, and proper formatting with actionable commands.

This decomposition matters because cybersecurity problems are inherently multi-domain. A single vulnerability assessment touches CVE databases, CWE taxonomies, CVSS scoring, MITRE ATT&CK technique mapping, and remediation tooling. By explicitly planning which skill domains to activate before generating the response, the method avoids the shallow "list of tips" failure mode and instead produces responses that chain knowledge across frameworks the way a senior security analyst would.

The training data behind this method spans three pillars: **Knowledge** (MITRE ATT&CK, CAPEC, CWE, OWASP, NIST frameworks), **Skills** (HackTricks, penetration testing writeups, CTF notes, payload construction), and **Tools** (Kali Linux documentation, Linux man pages, CLI cheat-sheets). When applying this skill, Claude should draw on all three pillars for each response.

## Step-by-Step Workflow

1. **Classify the security domain.** Determine which cybersecurity pillars apply: Knowledge (frameworks, taxonomies), Skills (offensive/defensive techniques), or Tools (CLI commands, platform-specific utilities). Most real queries touch 2-3 pillars.

2. **Identify the relevant frameworks.** Map the query to specific standards: MITRE ATT&CK technique IDs (e.g., T1059.001), CWE numbers (e.g., CWE-79), CAPEC attack patterns, OWASP categories, or CVSS v3.1 vectors. Always cite specific identifiers, never generic categories.

3. **Plan the skill decomposition.** Before generating the response, explicitly determine which sub-skills are needed. For example, a CVE analysis requires: (a) vulnerability description parsing, (b) CWE root-cause mapping, (c) CVSS vector estimation, (d) ATT&CK technique correlation, (e) remediation command generation.

4. **Ground every claim in a specific source.** Reference specific framework entries (ATT&CK technique pages, CWE definitions, tool documentation) rather than general security wisdom. If the exact CVE/CWE/technique is known, cite it. If inferring, state the inference explicitly.

5. **Generate tool commands with full syntax.** When recommending security tools, provide complete, copy-paste-ready commands with flags explained. Include the specific context (target, port, protocol) from the user's scenario. Never give a bare tool name without a concrete invocation.

6. **Chain techniques in attack/defense order.** Structure multi-step procedures in operational sequence: reconnaissance before exploitation, exploitation before persistence, detection before response. Mirror real-world workflow ordering.

7. **Provide both attacker and defender perspectives.** For each technique or vulnerability discussed, include how it is exploited AND how it is detected/mitigated. This dual-lens approach reflects the RedSage training methodology of covering offensive and defensive workflows.

8. **Validate with structured output.** Present findings in structured formats: CVSS vector strings, ATT&CK Navigator layers, CWE taxonomy trees, or tabular tool comparison matrices. Structured output is more actionable than prose.

9. **Flag authorization boundaries.** For any offensive technique, explicitly state the authorization requirements. Distinguish between techniques appropriate for authorized pentesting, CTF competitions, and security research versus those requiring additional safeguards.

## Concrete Examples

**Example 1: CVE Analysis with Framework Mapping**

User: "Analyze CVE-2024-3094 (the xz backdoor) and map it to relevant security frameworks."

Approach:
1. Classify: Knowledge (supply chain attack taxonomy) + Skills (backdoor analysis) + Tools (detection commands)
2. Identify frameworks: CWE-506 (Embedded Malicious Code), CAPEC-443 (Malicious Logic Inserted Into Product Software), ATT&CK T1195.002 (Supply Chain Compromise: Compromise Software Supply Chain)
3. Decompose into sub-analyses: vulnerability mechanics, supply chain vector, detection methodology, remediation

Output:
```
## CVE-2024-3094: XZ Utils Backdoor Analysis

### Vulnerability Summary
Malicious code injected into xz/liblzma versions 5.6.0-5.6.1 via compromised
build process, creating an authentication bypass in OpenSSH via systemd's
libsystemd dependency on liblzma.

### Framework Mapping
| Framework    | ID            | Description                                    |
|-------------|---------------|------------------------------------------------|
| CWE         | CWE-506       | Embedded Malicious Code                        |
| CAPEC       | CAPEC-443     | Malicious Logic Inserted Into Product Software |
| ATT&CK      | T1195.002     | Supply Chain Compromise: Software              |
| ATT&CK      | T1556.004     | Modify Authentication Process                  |
| CVSS v3.1   | 10.0 CRITICAL | AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H          |

### Detection Commands
# Check installed xz version
xz --version
# Expected safe output: xz (XZ Utils) 5.4.x or 5.6.2+

# Verify liblzma is not backdoored (check for IFUNC resolver)
hexdump -C /usr/lib/liblzma.so.5 | grep -c "f3 0f 1e fa"

# Scan for affected packages on Debian/Ubuntu
dpkg -l | grep -E "xz-utils|liblzma" | awk '{print $2, $3}'

### Remediation
Downgrade to xz 5.4.x or upgrade to 5.6.2+. Audit build pipelines
for unsigned or unreviewed commits from external contributors.
```

**Example 2: Penetration Testing Workflow with Tool Commands**

User: "I'm doing an authorized pentest on a web application at 10.10.10.50. Help me with the reconnaissance phase."

Approach:
1. Classify: Tools (recon utilities) + Skills (enumeration methodology)
2. Plan: port scanning -> service enumeration -> web directory discovery -> technology fingerprinting
3. Generate commands in operational sequence with flag explanations

Output:
```
## Authorized Web Application Reconnaissance - 10.10.10.50

### Phase 1: Port Scanning
# Fast TCP SYN scan of all ports, service version detection, default scripts
sudo nmap -sS -sV -sC -p- --min-rate=1000 -oN recon/nmap_full.txt 10.10.10.50

# UDP scan of top 100 ports (often reveals SNMP, DNS, TFTP)
sudo nmap -sU --top-ports 100 -oN recon/nmap_udp.txt 10.10.10.50

### Phase 2: Web Enumeration (after confirming HTTP/HTTPS ports)
# Directory brute-force with common wordlist
gobuster dir -u http://10.10.10.50 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt,bak -t 50 -o recon/gobuster.txt

# Technology fingerprinting
whatweb http://10.10.10.50 --aggression=3 --log-verbose=recon/whatweb.txt

# Check for known CMS vulnerabilities
nikto -h http://10.10.10.50 -o recon/nikto.txt -Format txt

### Phase 3: SSL/TLS Assessment (if port 443 open)
# Check certificate details and cipher suites
sslscan 10.10.10.50:443

# Test for known TLS vulnerabilities
testssl.sh --vulnerable 10.10.10.50:443

### Next Steps
Review nmap output for non-standard services. Check gobuster results
for admin panels, backup files (.bak, .old), and API endpoints.
Cross-reference whatweb CMS detection with known CVEs.
```

**Example 3: Secure Code Review with CWE Mapping**

User: "Review this Python Flask endpoint for security issues."
```python
@app.route('/search')
def search():
    query = request.args.get('q')
    results = db.execute(f"SELECT * FROM products WHERE name LIKE '%{query}%'")
    return render_template_string(f"<h1>Results for {query}</h1>")
```

Approach:
1. Classify: Knowledge (OWASP, CWE) + Skills (code audit)
2. Identify each vulnerability, map to CWE, rate severity, provide fix

Output:
```
## Security Audit: 3 Critical Vulnerabilities Found

### 1. SQL Injection (CRITICAL)
- CWE: CWE-89 (Improper Neutralization of Special Elements in SQL Command)
- OWASP: A03:2021 - Injection
- ATT&CK: T1190 (Exploit Public-Facing Application)
- Line: db.execute(f"SELECT * FROM products WHERE name LIKE '%{query}%'")
- Impact: Full database read/write/delete, potential RCE via stacked queries

Fix: Use parameterized queries.
  results = db.execute(
      "SELECT * FROM products WHERE name LIKE :q",
      {"q": f"%{query}%"}
  )

### 2. Reflected XSS via Server-Side Template Injection (CRITICAL)
- CWE: CWE-79 (XSS) + CWE-1336 (Server-Side Template Injection)
- OWASP: A03:2021 - Injection
- Line: render_template_string(f"<h1>Results for {query}</h1>")
- Impact: render_template_string with user input enables both XSS and
  full SSTI (Jinja2 RCE via {{ config.items() }} or worse)

Fix: Use static templates with escaped variables.
  # templates/search.html: <h1>Results for {{ query }}</h1>
  return render_template("search.html", query=query)

### 3. Missing Input Validation (MEDIUM)
- CWE: CWE-20 (Improper Input Validation)
- Line: query = request.args.get('q')  # No None check, no length limit
- Impact: Potential DoS via extremely long query strings

Fix: Validate and sanitize input.
  query = request.args.get('q', '').strip()[:200]
  if not query:
      abort(400, "Search query required")
```

## Best Practices

**Do:**
- Always cite specific framework identifiers (CWE-XXX, T1XXX.XXX, CAPEC-XXX) rather than vague category names. Precision enables actionable lookup and correlation.
- Provide complete, runnable tool commands with output file paths and explain non-obvious flags. Security professionals copy-paste commands in time-critical situations.
- Decompose complex security questions into the Knowledge/Skills/Tools triad before responding. This prevents shallow answers that miss operational context.
- Include detection signatures alongside exploitation techniques. Every offensive step should have a corresponding defensive indicator.

**Avoid:**
- Giving generic security advice ("keep software updated," "use strong passwords") without specific, contextual actions tied to the user's scenario.
- Recommending tools without specifying exact flags, wordlists, and output formats. A bare `nmap 10.10.10.50` is not useful guidance.
- Conflating vulnerability categories. SQL injection (CWE-89) and command injection (CWE-78) have different root causes, detection methods, and fixes. Be precise.
- Skipping the authorization context for offensive techniques. Every pentesting recommendation must acknowledge the authorized scope.

## Error Handling

- **Unknown CVE or outdated data:** If Claude's training data predates a CVE, state this explicitly: "My knowledge cutoff may not include CVE-YYYY-XXXXX. Verify details at nvd.nist.gov." Provide analysis of the vulnerability class (CWE) even when the specific CVE details are uncertain.
- **Ambiguous scope:** If the user's authorization context is unclear (pentest vs. production vs. CTF), ask before providing exploitation commands. Default to the defensive/analytical perspective.
- **Tool version mismatches:** Security tools change flags between versions. Note the assumed tool version (e.g., "nmap 7.94+") and suggest `--help` verification when commands include newer features.
- **Framework mapping conflicts:** Some vulnerabilities map to multiple CWEs or ATT&CK techniques. List all applicable mappings ranked by specificity rather than picking one.
- **Incomplete reconnaissance:** If the user asks to jump to exploitation without recon results, recommend completing enumeration first. Premature exploitation wastes time and may trigger alerts.

## Limitations

- Claude's training data has a knowledge cutoff and will not cover CVEs, ATT&CK updates, or tool releases published after that date. Always cross-reference with live databases (NVD, MITRE ATT&CK website) for current intelligence.
- This skill applies RedSage's methodology (structured decomposition, framework grounding, multi-turn expert workflow simulation) but does not run the actual RedSage 8B model. Claude's general training may differ from RedSage's domain-specific fine-tuning on edge cases in obscure security tooling.
- Hands-on exploitation tasks (running actual exploits, interacting with live targets) require execution environments that Claude cannot provide. The skill generates the commands and methodology; execution is the user's responsibility within their authorized scope.
- Malware reverse engineering at the binary level (disassembly analysis, dynamic sandbox results) requires specialized tools (Ghidra, IDA, Cuckoo) that Claude cannot execute. The skill can guide methodology and interpret shared output but cannot perform binary analysis directly.
- CVSS scoring provided by this skill is an estimate based on vulnerability description analysis. Official scores should come from NVD or the vendor's security advisory.

## Reference

**Paper:** [RedSage: A Cybersecurity Generalist LLM](https://arxiv.org/abs/2601.22159v1) (ICLR 2026)
**Key insight:** The Planner-Augmenter agentic augmentation pipeline (Section 3.2) that decomposes cybersecurity problems into Knowledge/Skills/Tools pillars and generates grounded multi-turn expert workflows, achieving +5.59 point improvement over baseline models on cybersecurity benchmarks while also improving general reasoning by +5.05 points.
**Resources:** Models, datasets, and code at [github.com/RISys-Lab/RedSage](https://github.com/RISys-Lab/RedSage)