---
name: "realsec-bench-benchmark-evaluating-secure"
description: >
  Evaluate and improve secure code generation using the RealSec-bench methodology:
  multi-stage vulnerability detection with CodeQL SAST scanning, inter-procedural
  data flow analysis, and composite security+correctness scoring via SecurePass@K.
  Applies lessons from real-world Java repository vulnerabilities spanning 19 CWE types.
  Use when: "audit this code for security vulnerabilities", "generate secure Java code",
  "check for CWE issues in this repo", "evaluate code security with CodeQL",
  "review this code for injection/crypto/data-flow vulnerabilities",
  "write secure code that avoids common weaknesses"
---

# RealSec-bench: Secure Code Generation Evaluation & Guidance

This skill enables Claude to apply the RealSec-bench methodology for evaluating and generating secure code. RealSec-bench (Wang et al., 2026) demonstrated that current LLMs produce functionally correct but insecure code, that RAG helps correctness but not security, and that naive security prompting causes compilation failures. This skill encodes actionable lessons from those findings: how to detect real-world vulnerabilities through inter-procedural data flow analysis, how to evaluate generated code for both correctness and security simultaneously, and how to guide secure code generation without sacrificing functionality.

## When to Use

- When the user asks to **generate Java code** that handles sensitive operations (cryptography, SQL queries, file I/O, deserialization, authentication, HTTP requests)
- When the user wants to **audit existing code** for security vulnerabilities, especially across method/class boundaries
- When the user asks to **evaluate whether generated code is secure**, not just functionally correct
- When reviewing code that involves **inter-procedural data flows** where tainted input crosses multiple method calls before reaching a sink
- When the user asks to **set up CodeQL scanning** or interpret CodeQL results for a Java project
- When the user wants to understand **which CWE types** are most relevant to their Java codebase
- When the user asks to **write security-aware code** without breaking compilation or functional correctness

## Key Technique

**The RealSec-bench Pipeline.** The paper constructs a benchmark from real-world high-risk Java repositories using three stages: (1) systematic SAST scanning with CodeQL to identify potential vulnerabilities, (2) LLM-based false positive elimination to filter out non-genuine findings, and (3) human expert validation to confirm true positives. This pipeline yields 105 ground-truth vulnerability instances across 19 CWE types. The critical insight is that vulnerabilities in real code are not localized — they involve inter-procedural data flows where tainted data may pass through up to 34 method calls (hops) between source and sink.

**SecurePass@K Metric.** Traditional metrics like Pass@K only measure functional correctness (does the code compile and pass tests?). RealSec-bench introduces SecurePass@K, a composite metric that requires generated code to *both* pass functional tests *and* be free of security vulnerabilities as verified by CodeQL. Given K generated samples, SecurePass@K measures the probability that at least one sample is both functionally correct and secure. This reveals the true gap: models achieving 60-80% Pass@K often drop to 20-40% SecurePass@K.

**Counter-Intuitive Findings.** Two results reshape how we should approach secure code generation: (1) RAG improves functional correctness by providing repository context, but does *not* improve security — models use context to write code that works, not code that is safe. (2) Adding generic security instructions to prompts (e.g., "write secure code", "avoid SQL injection") causes compilation failures and reduces functional correctness without reliably preventing vulnerabilities. The effective approach is *specific, targeted* security guidance tied to the exact CWE and data flow pattern at hand.

## Step-by-Step Workflow

### For Generating Secure Code

1. **Identify the security-sensitive operation** in the user's request. Classify it by CWE category: injection (CWE-89, CWE-79, CWE-78), cryptographic (CWE-327, CWE-330, CWE-328), deserialization (CWE-502), path traversal (CWE-22), or other applicable types from the 19 covered by RealSec-bench.

2. **Trace the data flow from source to sink.** Before writing code, map how user-controlled input enters the system (source), what transformations it undergoes, and where it reaches a security-sensitive API (sink). Document each hop explicitly — real vulnerabilities average 5-10 hops and can reach 34.

3. **Apply CWE-specific mitigations at the correct program point.** Do NOT add generic security boilerplate. Instead, place the specific defense (parameterized query, input validation, safe API choice) at the precise point in the data flow where it neutralizes the threat. For example:
   - CWE-89 (SQL Injection): Use `PreparedStatement` with parameterized queries, never string concatenation
   - CWE-79 (XSS): Apply output encoding at the rendering boundary, not at input
   - CWE-327 (Weak Crypto): Use `AES/GCM/NoPadding` instead of `DES` or `AES/ECB`
   - CWE-502 (Deserialization): Use allowlists with `ObjectInputFilter`, never deserialize untrusted data directly
   - CWE-22 (Path Traversal): Canonicalize paths and validate against a base directory

4. **Verify the code compiles and passes functional requirements first.** Security fixes that break compilation are worse than no fix — they produce zero value. Always confirm the code is syntactically valid and functionally correct before layering in security.

5. **Validate security with targeted CodeQL queries.** Run the specific CodeQL query for the identified CWE, not a blanket scan. For example: `codeql query run java/ql/src/Security/CWE-089/SqlInjection.ql` against the generated code.

6. **Check inter-procedural flows.** If the generated code calls helper methods that handle tainted data, trace through those calls to ensure sanitization is not bypassed by an alternate code path. A method that sanitizes input in one branch but passes it raw in another is still vulnerable.

### For Auditing Existing Code

7. **Set up CodeQL for the repository.** Create a CodeQL database: `codeql database create <db-name> --language=java --source-root=<repo-path>`. Run the security-and-quality suite: `codeql database analyze <db-name> java-security-and-quality --format=sarif-latest --output=results.sarif`.

8. **Filter false positives with contextual analysis.** For each CodeQL finding, examine the full data flow path. Check whether: (a) the source is actually user-controllable, (b) intermediate transformations effectively sanitize the data, (c) the sink is genuinely security-sensitive in this context. Discard findings where all three conditions are not met.

9. **Classify surviving findings by severity and data flow complexity.** Rank by: number of inter-procedural hops (more hops = harder to find and fix), CWE severity, and whether the vulnerability is in a public-facing code path.

10. **Generate fixes that preserve functional behavior.** For each confirmed vulnerability, produce a minimal patch that adds the specific mitigation without changing the method signature, return type, or observable behavior for valid inputs.

## Concrete Examples

**Example 1: Secure Database Query Generation**

User: "Write a Java method that searches users by name in a database"

Approach:
1. Identify CWE-89 (SQL Injection) risk — user-supplied name flows into SQL query
2. Trace data flow: parameter `name` -> SQL query string -> `Statement.executeQuery()` (2 hops)
3. Apply mitigation: `PreparedStatement` with parameter binding

```java
// INSECURE — string concatenation creates SQL injection (CWE-89)
public List<User> findByName(Connection conn, String name) throws SQLException {
    String sql = "SELECT * FROM users WHERE name = '" + name + "'";
    Statement stmt = conn.createStatement();
    ResultSet rs = stmt.executeQuery(sql);
    // ... process results
}

// SECURE — parameterized query neutralizes injection at the sink
public List<User> findByName(Connection conn, String name) throws SQLException {
    String sql = "SELECT * FROM users WHERE name = ?";
    PreparedStatement stmt = conn.prepareStatement(sql);
    stmt.setString(1, name);
    ResultSet rs = stmt.executeQuery();
    List<User> users = new ArrayList<>();
    while (rs.next()) {
        users.add(new User(rs.getLong("id"), rs.getString("name"), rs.getString("email")));
    }
    return users;
}
```

Validation: `codeql query run java/ql/src/Security/CWE-089/SqlTaintedLocal.ql` returns zero results.

**Example 2: Multi-Hop Inter-Procedural Vulnerability Audit**

User: "Audit this Java service for path traversal vulnerabilities"

Approach:
1. Identify entry points where user input arrives (controller parameters, request bodies)
2. Trace data flow through service layers to file system operations
3. Count hops and check for sanitization at each boundary

```
Data flow trace (7 hops):
  [Source] FileController.download(HttpServletRequest req)
    -> req.getParameter("filepath")                          // Hop 1: user input
    -> FileService.getFile(String path)                      // Hop 2: passed to service
    -> FileValidator.normalize(String path)                  // Hop 3: validation attempt
    -> PathUtils.resolve(String base, String relative)       // Hop 4: path construction
    -> new File(resolvedPath)                                // Hop 5: File object created
    -> FileInputStream(file)                                 // Hop 6: file opened
    -> IOUtils.copy(inputStream, response.getOutputStream()) // Hop 7: data sent to user

Finding: FileValidator.normalize() calls path.replace("../", "") which is
bypassable with "....//" — after one replacement, "../" remains.

Fix: Replace string manipulation with canonical path validation:
```

```java
// SECURE fix for path traversal (CWE-22)
public Path resolveSafely(Path baseDir, String userInput) throws IOException {
    Path resolved = baseDir.resolve(userInput).toRealPath();
    if (!resolved.startsWith(baseDir.toRealPath())) {
        throw new SecurityException("Path traversal attempt blocked");
    }
    return resolved;
}
```

**Example 3: Evaluating Generated Code with SecurePass@K**

User: "I generated 5 code samples for a crypto function. How do I evaluate them?"

Approach:
1. Test each sample for functional correctness (compilation + unit tests)
2. Scan each passing sample with CodeQL for CWE-327 (weak crypto) and CWE-330 (weak randomness)
3. Compute SecurePass@K

```
Sample 1: Compiles ✓  Tests pass ✓  CodeQL: uses DES (CWE-327) ✗
Sample 2: Compiles ✓  Tests pass ✓  CodeQL: uses SecureRandom + AES-GCM ✓
Sample 3: Compiles ✗  (missing import)
Sample 4: Compiles ✓  Tests fail ✗
Sample 5: Compiles ✓  Tests pass ✓  CodeQL: hardcoded IV (CWE-329) ✗

Pass@5:    3/5 samples compile and pass tests = high functional correctness
SecurePass@5: 1/5 samples are both correct AND secure = low security rate

Conclusion: 60% functional correctness but only 20% secure correctness.
Sample 2 is the only acceptable output.
```

## Best Practices

- **Do:** Apply CWE-specific mitigations tied to the exact vulnerability type. "Use PreparedStatement for CWE-89" is effective; "write secure code" is not.
- **Do:** Trace complete data flows from source to sink before writing fixes. Vulnerabilities hide in intermediate hops across method and class boundaries.
- **Do:** Validate that security fixes compile and pass functional tests before considering them complete. A fix that breaks the build is not a fix.
- **Do:** Use CodeQL queries targeted to the specific CWE under investigation, not broad security suites that produce noisy results.
- **Avoid:** Adding generic security disclaimers or boilerplate to prompts. The paper shows this *increases* compilation failures and does not improve security.
- **Avoid:** Assuming RAG context alone will produce secure code. Repository context helps the model write *working* code, not *safe* code — security reasoning must be explicit.
- **Avoid:** Treating single-method analysis as sufficient. Real vulnerabilities span 5-34 inter-procedural hops; examining one method in isolation misses the attack surface.

## Error Handling

| Problem | Cause | Resolution |
|---------|-------|------------|
| CodeQL reports no findings on obviously insecure code | Wrong query suite or incomplete database | Rebuild the CodeQL database with `--overwrite`; verify the language extractor matched the build system |
| Security fix causes compilation failure | Overly aggressive input validation or wrong API usage | Start from the functionally correct version, make the *minimal* change needed for security |
| False positive from CodeQL | Sanitization happens in a method CodeQL doesn't model | Add a CodeQL model extension (`.yml`) marking the sanitizer, or document the false positive with a `// lgtm` annotation and justification |
| Data flow too complex to trace manually | Vulnerability spans 20+ hops across packages | Use CodeQL's path-problem queries which output the full source-to-sink trace: `@kind path-problem` |
| Fix addresses one code path but misses another | Multiple callers pass tainted data through different routes | Run CodeQL after the fix to verify all paths are covered; check for method overrides and interface implementations |

## Limitations

- **Java-focused.** RealSec-bench is constructed from Java repositories. The methodology (CodeQL + data flow analysis) generalizes to other CodeQL-supported languages (C/C++, Python, JavaScript, Go, C#), but the specific CWE distributions and vulnerability patterns may differ.
- **SAST-only detection.** CodeQL is a static analysis tool. It cannot detect runtime-dependent vulnerabilities (race conditions, timing attacks, business logic flaws) or vulnerabilities that depend on deployment configuration.
- **19 CWE types out of 900+.** The benchmark covers the most prevalent CWEs in high-risk Java repos, but many vulnerability classes (e.g., CWE-362 race conditions, CWE-863 authorization) are not represented.
- **SecurePass@K requires a test suite.** The metric depends on having functional tests to verify correctness. For code without tests, only the security component (CodeQL scan) can be evaluated.
- **Not a replacement for penetration testing.** Static analysis catches known vulnerability patterns. Novel attack vectors, logic flaws, and configuration-dependent issues require dynamic testing and manual review.

## Reference

Wang, Y., Zhang, Z., Wang, C., Xu, X., & Liu, M. (2026). *RealSec-bench: A Benchmark for Evaluating Secure Code Generation in Real-World Repositories.* arXiv:2601.22706v1. [https://arxiv.org/abs/2601.22706v1](https://arxiv.org/abs/2601.22706v1)

Key takeaway: RAG and generic security prompting are insufficient for secure code generation. Effective security requires CWE-specific mitigations applied at precise points in inter-procedural data flows, validated by targeted static analysis.