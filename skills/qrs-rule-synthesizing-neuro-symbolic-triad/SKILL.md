---
name: "qrs-rule-synthesizing-neuro-symbolic-triad"
description: "Autonomous vulnerability discovery using the QRS (Query, Review, Sanitize) neuro-symbolic triad. Generates CodeQL queries from CWE schemas, validates findings through semantic reasoning, and confirms exploitability via PoC synthesis. Use when: 'scan this Python package for vulnerabilities', 'generate CodeQL queries for CWE-078', 'review these SAST findings for false positives', 'find injection vulnerabilities in this codebase', 'write a taint-tracking query for path traversal', 'triage these security alerts'."
---

This skill enables Claude to perform autonomous vulnerability discovery on Python codebases using the QRS (Query, Review, Sanitize) framework from Tsigkourakos & Patsakis (2026). Instead of relying solely on predefined SAST rules, Claude synthesizes CodeQL queries from CWE specifications and few-shot patterns, then validates each finding through semantic code review and automated exploit synthesis -- a three-agent pipeline that discovers vulnerability classes beyond what static rule sets catch while drastically reducing false positives.

## When to Use

- When the user asks to scan a Python package or codebase for security vulnerabilities
- When the user wants CodeQL queries generated for specific CWE categories (injection, path traversal, SSRF, deserialization, etc.)
- When the user needs to triage or validate SAST tool output and reduce false positives
- When the user asks to find zero-day vulnerabilities in open-source Python libraries
- When the user wants to write taint-tracking queries that go beyond default CodeQL rule packs
- When the user asks to generate proof-of-concept exploits for discovered vulnerabilities
- When the user wants to audit dependencies in a PyPI-based project for known and novel vulnerability patterns

## Key Technique

**The QRS framework inverts the traditional SAST paradigm.** Conventional tools like CodeQL, Semgrep, and SonarQube require expert-crafted queries and only detect predefined patterns. Prior LLM-augmented approaches merely triage existing tool output. QRS instead uses LLMs to *generate* the queries themselves, then validates results through multi-stage semantic reasoning. This enables discovery of vulnerability classes that no existing rule set covers.

**The three-agent architecture works as follows.** The **Q (Query) agent** receives a compact schema definition (`knowledge.md`) containing CodeQL syntax primitives, canonical taint-tracking patterns, and CWE-specific source/sink definitions. Given a target package and CWE focus, it synthesizes CodeQL queries with a `fixup_query` recovery loop that automatically repairs compilation errors. The **R (Review) agent** clusters raw findings using the MITRE CWE taxonomy, reconstructs execution context via `trace_flow` (data-flow path mapping with 1-3 line tolerance) and sandboxed `grep_search`, then classifies each finding as True Positive (>=90% confidence), Manual Review (>=70%), or False Positive. The **S (Sanitize) agent** performs clean-slate verification: it queries CVE databases for novelty, assesses API reachability and exploitability, and applies multi-label tags (ZERO_DAY, KNOWN_CVE, HALLUCINATION, CODE_SMELL, CONTEXT_MISMATCH, etc.) to produce the final report.

**On real-world evaluation**, QRS achieved 90.6% detection accuracy across 20 historical CVEs in popular PyPI libraries, and discovered 39 medium-to-high-severity vulnerabilities in the top 100 most-downloaded packages -- 5 of which received new CVE assignments. The framework spans 115 unique CWEs across five major categories: injection/code execution, path traversal, DoS/resource exhaustion, cryptographic weaknesses, and race conditions.

## Step-by-Step Workflow

1. **Profile the target package.** Identify the package name, version, entry points, and public API surface. List imported modules and external dependencies. Determine the CWE categories most relevant to the package's functionality (e.g., web frameworks -> injection/XSS; file utilities -> path traversal; serialization libraries -> CWE-502).

2. **Build the CodeQL knowledge schema.** Create a compact reference containing: (a) CodeQL syntax primitives for Python (`import python`, `DataFlow::PathGraph`, `RemoteFlowSource`), (b) canonical source/sink pairs for each target CWE, (c) 2-3 few-shot query examples showing the pattern of `from DataFlow::PathNode source, DataFlow::PathNode sink` with `isSource`/`isSink` predicates. Keep this under 2000 tokens.

3. **Synthesize CodeQL queries (Q agent phase).** For each target CWE, generate a CodeQL query that defines: the source (e.g., `RemoteFlowSource` for user input), the sink (e.g., `os.system()` call for CWE-078), and the taint-tracking configuration connecting them. Use a `fixup_query` loop: if the query fails to compile, read the error, identify the broken predicate or import, and regenerate with corrections. Aim for 3-5 queries per CWE category.

4. **Execute queries against the CodeQL database.** Build a CodeQL database from the target package source (`codeql database create`), then run each synthesized query (`codeql query run`). Collect raw results as SARIF or CSV with file paths, line numbers, and matched source/sink pairs.

5. **Cluster findings by MITRE taxonomy (R agent phase - clustering).** Group raw results by CWE hierarchy, then by severity and prevalence. Deduplicate findings that share the same sink but differ only in source path. This reduces noise before semantic review.

6. **Reconstruct context and validate each finding (R agent phase - review).** For each clustered finding: (a) extract the source code at the flagged location with surrounding context (10-20 lines), (b) trace the data-flow path from source to sink, mapping each propagation step, (c) check whether sanitization functions (escaping, validation, parameterization) exist along the path, (d) classify as True Positive (>=90% confidence the vulnerability is real and exploitable), Manual Review (>=70%, needs human inspection), or False Positive (sanitized, unreachable, or test-only code).

7. **Synthesize proof-of-concept exploits (R/S agent phase - exploitation).** For each True Positive finding, generate a minimal PoC that demonstrates: prerequisites (imports, setup), the malicious payload, and the expected vulnerable behavior. Include both the exploit code and a written scenario describing the attack path.

8. **Verify novelty and classify (S agent phase).** For each confirmed vulnerability: (a) query the CVE database (NVD/MITRE) to check if it's already known, (b) assess operational context -- is the vulnerable code in production paths or only in build scripts/tests?, (c) apply multi-label classification: ZERO_DAY, KNOWN_CVE, NOVEL_FINDING, BUILD_SCRIPT, TEST_CODE, EXPLOITABILITY, HALLUCINATION, CODE_SMELL, or CONTEXT_MISMATCH.

9. **Generate the final security report.** Produce a structured report containing: (a) executive summary with finding counts by severity, (b) per-finding details with CWE ID, CVSS estimate, affected file/line, data-flow trace, PoC code, and classification labels, (c) remediation recommendations specific to each finding.

10. **Iterate on false negatives.** If the user expects certain vulnerability types that weren't found, revisit the Q agent phase with refined CWE focus and additional source/sink definitions. Consider custom sinks for framework-specific APIs (e.g., Django ORM `raw()`, Flask `send_file()`).

## Concrete Examples

**Example 1: Scanning a Python web framework library for injection vulnerabilities**

User: "Scan the `flask-admin` package for SQL injection and command injection vulnerabilities."

Approach:
1. Profile flask-admin: identify model views, form handlers, and any raw SQL usage.
2. Synthesize CodeQL queries targeting CWE-089 (SQL injection) and CWE-078 (OS command injection):

```ql
/**
 * @name SQL injection in flask-admin model view
 * @kind path-problem
 * @id python/sql-injection-flask-admin
 */
import python
import semmle.python.dataflow.new.DataFlow
import semmle.python.dataflow.new.TaintTracking
import semmle.python.dataflow.new.RemoteFlowSources
import semmle.python.Concepts

class SqlInjectionConfig extends TaintTracking::Configuration {
  SqlInjectionConfig() { this = "SqlInjectionConfig" }

  override predicate isSource(DataFlow::Node source) {
    source instanceof RemoteFlowSource
  }

  override predicate isSink(DataFlow::Node sink) {
    exists(SqlExecution sqlExec | sink = sqlExec.getSql())
  }
}

from SqlInjectionConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select sink.getNode(), source, sink, "SQL injection from $@.", source.getNode(), "user input"
```

3. Run queries, cluster results by affected module, trace each flow path.
4. For confirmed findings, generate PoC showing how a crafted form submission reaches `db.session.execute()`.

Output:
```
## Security Report: flask-admin

### Finding 1 (HIGH) - CWE-089: SQL Injection
- File: flask_admin/contrib/sqla/view.py:342
- Source: Request parameter via `get_list()` filter argument
- Sink: `self.session.execute(text(query))`
- Flow: request.args["flt"] -> _apply_filters() -> session.execute()
- Sanitization: None detected on path
- Classification: TRUE_POSITIVE, EXPLOITABILITY
- PoC:
  curl 'http://target/admin/user/?flt=name%3D1%27%3B+DROP+TABLE+users--'

### Remediation
Use parameterized queries: `session.execute(text(query), {"param": value})`
```

**Example 2: Triaging existing SAST findings to reduce false positives**

User: "CodeQL flagged 47 path traversal findings in our file-upload service. Most look like false positives. Help me triage them."

Approach:
1. Group the 47 findings by CWE sub-type (CWE-022 directory traversal, CWE-073 external control of filename, CWE-059 symlink following).
2. For each cluster, reconstruct execution context:
   - Read the flagged code and 20 lines of surrounding context
   - Trace the data-flow from the identified source to sink
   - Check for sanitization: `os.path.basename()`, `secure_filename()`, path prefix validation, chroot/jail patterns
3. Classify each finding:
   - If `werkzeug.utils.secure_filename()` is called before file write -> FALSE_POSITIVE
   - If path is constructed from user input but validated against an allowlist -> FALSE_POSITIVE
   - If `os.path.join(base, user_input)` without `os.path.abspath()` check -> TRUE_POSITIVE

Output:
```
## Triage Results: 47 Path Traversal Findings

| Classification    | Count | Action                |
|-------------------|-------|-----------------------|
| True Positive     | 3     | Fix immediately       |
| Manual Review     | 7     | Developer review      |
| False Positive    | 37    | Suppress in config    |

### True Positives (3)
1. upload_handler.py:89 - User filename passed directly to os.path.join()
   without normalization. PoC: filename="../../etc/passwd"
2. export_service.py:156 - Template path from query param, no allowlist.
3. static_server.py:42 - send_file() with unvalidated relative path.

### False Positive Pattern (37)
All use secure_filename() or validate against ALLOWED_EXTENSIONS
before file operations. Safe to suppress with @SuppressWarnings annotation.
```

**Example 3: Generating CodeQL queries for a novel vulnerability class**

User: "Write CodeQL queries to detect unsafe YAML deserialization (CWE-502) in Python packages."

Approach:
1. Define sources (remote input, file reads, environment variables) and sinks (`yaml.load()` without `Loader=SafeLoader`, `yaml.unsafe_load()`, `yaml.full_load()`).
2. Synthesize the query:

```ql
/**
 * @name Unsafe YAML deserialization
 * @kind path-problem
 * @id python/unsafe-yaml-deserialization
 */
import python
import semmle.python.dataflow.new.DataFlow
import semmle.python.dataflow.new.TaintTracking
import semmle.python.dataflow.new.RemoteFlowSources
import semmle.python.ApiGraphs

class UnsafeYamlLoadSink extends DataFlow::Node {
  UnsafeYamlLoadSink() {
    exists(API::CallNode call |
      call = API::moduleImport("yaml").getMember("load").getACall() and
      not exists(call.getArgByName("Loader")) and
      this = call.getArg(0)
    )
    or
    this = API::moduleImport("yaml").getMember("unsafe_load").getACall().getArg(0)
  }
}

class UnsafeYamlConfig extends TaintTracking::Configuration {
  UnsafeYamlConfig() { this = "UnsafeYamlConfig" }

  override predicate isSource(DataFlow::Node source) {
    source instanceof RemoteFlowSource
  }

  override predicate isSink(DataFlow::Node sink) {
    sink instanceof UnsafeYamlLoadSink
  }
}

from UnsafeYamlConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select sink.getNode(), source, sink,
  "Unsafe YAML deserialization of $@.", source.getNode(), "user-controlled data"
```

3. Also generate a local-source variant catching `yaml.load(open(config_file))` where `config_file` comes from CLI arguments.

Output: Two CodeQL queries targeting remote-source and local-source YAML deserialization, plus remediation guidance to replace `yaml.load()` with `yaml.safe_load()`.

## Best Practices

- **Do:** Start with the most relevant CWE categories for the target package's domain. A web framework needs injection/XSS queries; a file utility needs path traversal queries. Prioritize rather than scan for everything.
- **Do:** Use the `fixup_query` recovery pattern. When a generated CodeQL query fails to compile, parse the error message, identify the broken predicate or missing import, and regenerate. Budget 2-3 fix attempts per query before moving on.
- **Do:** Always trace the complete data-flow path from source to sink before classifying a finding. A sink without a reachable tainted source is not a vulnerability.
- **Do:** Apply the multi-label classification taxonomy (ZERO_DAY, KNOWN_CVE, HALLUCINATION, CODE_SMELL, TEST_CODE, BUILD_SCRIPT) to every finding. This prevents wasting time on test-only code or known issues.
- **Avoid:** Generating overly broad queries that match any function call as a sink. Always constrain sinks to specific dangerous APIs (e.g., `os.system`, `subprocess.call`, `eval`, `exec`).
- **Avoid:** Skipping the Sanitize phase. Without novelty verification and exploitability assessment, reports will include known CVEs, test-only findings, and theoretical-but-unexploitable issues that erode trust.
- **Avoid:** Treating LLM confidence scores as ground truth. The >=90% / >=70% thresholds are guidelines; always present Manual Review findings to the user rather than silently dropping them.

## Error Handling

- **CodeQL database creation fails:** Ensure the target package is pure Python or has its native extensions pre-built. Use `--language=python` explicitly. If the package requires complex build steps, create the database with `--command` pointing to the build script.
- **Generated query won't compile after 3 fix attempts:** Simplify the query by removing taint-tracking and using a direct sink-pattern match instead. This reduces precision but maintains coverage. Flag results from simplified queries as requiring extra manual review.
- **Too many raw findings (>500):** Increase clustering granularity -- group by file, then by function, then by CWE. Apply a pre-filter removing findings in test directories (`tests/`, `test_*.py`, `*_test.py`) and build scripts (`setup.py`, `setup.cfg`, `pyproject.toml` scripts).
- **CVE database query returns no results:** This doesn't confirm novelty -- the vulnerability may exist under a different identifier or in an advisory database not yet synced to NVD. Cross-check with GitHub Security Advisories and the package's own CHANGELOG/SECURITY.md.
- **PoC generation produces non-functional exploit:** The vulnerability may be real but require specific runtime conditions (authenticated session, specific config flags, race timing). Document the prerequisites rather than dismissing the finding.

## Limitations

- **Language scope:** The QRS framework was evaluated exclusively on Python packages. CodeQL supports other languages (Java, JavaScript, C/C++, Go), but the schema definitions, source/sink patterns, and few-shot examples need adaptation for each language.
- **CodeQL dependency:** The workflow requires CodeQL CLI and database creation, which adds infrastructure requirements. Packages that can't produce a CodeQL database (e.g., pure C extensions without Python stubs) are out of scope.
- **LLM hallucination risk:** The Q agent may generate syntactically valid but semantically incorrect queries that match benign code patterns. The R and S agents mitigate this but cannot eliminate it entirely -- human review remains necessary for high-stakes decisions.
- **Complex vulnerability patterns:** Multi-step vulnerabilities like TOCTOU race conditions (CWE-362/367), ASN.1 recursion bombs, and cross-module data flows that span multiple packages are harder to capture in single CodeQL queries. These accounted for lower detection rates in the paper's evaluation.
- **Token cost at scale:** Scanning 100 packages across all CWE categories costs ~$500 in LLM tokens. For budget-constrained scenarios, focus on the highest-risk CWE categories and use a cost-effective model (DeepSeek-Reasoner at ~$0.08/scan) for initial sweeps.
- **No runtime analysis:** QRS is purely static. Vulnerabilities that depend on runtime state, configuration, or environment variables may be missed or misclassified without dynamic testing to complement.

## Reference

Tsigkourakos, G. & Patsakis, C. (2026). *QRS: A Rule-Synthesizing Neuro-Symbolic Triad for Autonomous Vulnerability Discovery*. arXiv:2602.09774v1. https://arxiv.org/abs/2602.09774v1

Key sections to consult: Section 3 for the three-agent architecture and schema design, Section 4 for evaluation methodology and CWE coverage, Appendix Listings 1-9 for prompt templates and CodeQL query examples.