---
name: "yasa-scalable-multi-language-taint"
description: "Perform unified multi-language taint analysis across Java, JavaScript, Python, and Go codebases using YASA's UAST-based approach. Detects SQL injection, command injection, SSRF, XSS, deserialization, and privilege escalation vulnerabilities by tracing data flow from sources to sinks. Trigger phrases: 'find taint vulnerabilities across languages', 'multi-language security audit', 'trace user input to dangerous sinks', 'run taint analysis on this project', 'check for injection vulnerabilities', 'find data flow security issues'."
---

This skill enables Claude to perform systematic multi-language taint analysis on codebases written in Java, JavaScript, Python, and Go, following the YASA framework's Unified Abstract Syntax Tree (UAST) methodology. Rather than treating each language separately, Claude maps language-specific constructs to a shared semantic model, traces tainted data from user-controlled sources through assignments, container fields, function calls, and language-specific channels (promises, prototypes, goroutine channels) to dangerous sinks -- identifying real injection vulnerabilities that cross function boundaries and framework abstractions.

## When to Use

- When the user asks to audit a polyglot codebase for injection vulnerabilities (SQL injection, command injection, SSRF, XSS, code injection, deserialization)
- When the user wants to trace how user input flows through a web application built on Spring, Express, Flask, Django, FastAPI, Gin, or similar frameworks
- When reviewing a pull request for security issues involving untrusted data reaching dangerous functions
- When the user asks "is this user input sanitized before it reaches the database/shell/HTTP client?"
- When checking whether a Go channel, JavaScript promise chain, or Python generator propagates tainted data to a sink
- When performing a security review across multiple microservices written in different languages

## Key Technique

YASA's core insight is the **Unified Abstract Syntax Tree (UAST)** -- a 54-node-type intermediate representation that classifies language constructs into three tiers. **Universal nodes** (65% of constructs) cover operations shared across languages: literals, identifiers, binary expressions, if/while statements, function/class definitions, and variable declarations. **Language-specific nodes** (35%) preserve semantics that cannot be generalized without loss: Python's `yield`, Go's channel types, JavaScript's prototype chains. **Reducible nodes** are syntactic sugar desugared before analysis: list comprehensions become loops, arrow functions become regular function definitions. This tiered approach achieves 77% code reuse across languages while preserving the precision that single-language tools offer.

The analysis engine performs **context-sensitive, path-sensitive, field-sensitive interprocedural taint propagation**. It maintains an abstract value domain with four categories: primitive values (Prim), heap objects with field mappings (Obj), symbolic values for unresolvable references (Sym), and path-conditional values (Phi) that track different execution branches. Taint propagates through three core rules -- assignment (`x = y` taints x), container field access (`o.f = v` taints the field; `x = o.f` taints x), and function calls (tainted arguments propagate through return values) -- plus language-specific rules for JavaScript prototype chains and promise `.then()` chains, Go channel send/receive operations, and Python generator yields.

Framework-specific checkers identify entry points and sources automatically. For example, in Spring, `@RequestMapping` parameters are sources; in Express, `req.params`, `req.query`, and `req.body` are sources; in Flask, `request.args` and `request.form` are sources; in Gin, `c.Query()` and `c.Param()` are sources. Sinks include `os.system()`, `subprocess.run()`, `exec()`, `eval()`, SQL query functions, HTTP request builders, and deserialization calls. Sanitizers are functions that neutralize taint (e.g., parameterized queries, HTML escaping, input validation).

## Step-by-Step Workflow

1. **Identify the project's language(s) and frameworks.** Scan the repository for `pom.xml`/`build.gradle` (Java/Spring), `package.json` (JavaScript/Express/Egg.js), `requirements.txt`/`pyproject.toml` (Python/Flask/Django/FastAPI), or `go.mod` (Go/Gin/gRPC). This determines which framework-specific source/sink definitions to apply.

2. **Enumerate taint sources by framework.** Map each framework's entry points to user-controlled data:
   - **Spring (Java):** `@RequestMapping`/`@GetMapping`/`@PostMapping` method parameters, `@RequestParam`, `@PathVariable`, `@RequestBody`
   - **Express (JS):** `req.params.*`, `req.query.*`, `req.body.*`, `req.headers.*`
   - **Flask (Python):** `request.args`, `request.form`, `request.json`, `request.headers`
   - **Gin (Go):** `c.Query()`, `c.Param()`, `c.PostForm()`, `c.ShouldBindJSON()`

3. **Enumerate taint sinks by vulnerability class.** For each target vulnerability type, list the dangerous functions:
   - **Command Injection:** `os.system()`, `subprocess.run()`, `exec.Command()`, `child_process.exec()`
   - **SQL Injection:** Raw string concatenation in `cursor.execute()`, `db.Query()`, JDBC `Statement.execute()`, Sequelize `raw()`
   - **SSRF:** `http.Get()`, `requests.get()`, `axios.get()`, `HttpClient.send()` with tainted URLs
   - **Code Injection:** `eval()`, `exec()`, `Function()` constructor, `reflect.ValueOf().Call()`

4. **Trace taint propagation path-by-path from each source.** Follow data through assignments, function arguments/returns, object field accesses, and collection operations. At each step, ask: does this variable carry the taint forward? Apply the three core propagation rules:
   - Assignment: `x = tainted_value` -- x is now tainted
   - Field: `obj.field = tainted_value` -- obj.field is tainted; later `y = obj.field` taints y
   - Call: `result = func(tainted_arg)` -- if func returns data derived from its argument, result is tainted

5. **Apply language-specific propagation rules.** Handle constructs that differ by language:
   - **JavaScript:** Follow `.then()` and `await` chains (tainted promise resolves to tainted value); check prototype assignments (`Constructor.prototype.method = tainted` affects all instances)
   - **Go:** Follow channel send/receive (`ch <- taintedVal` means `val := <-ch` is tainted); track interface method dispatch through structural typing
   - **Python:** Follow generator `yield` (tainted yield means `next(gen)` is tainted); track decorator wrapping and `**kwargs` unpacking

6. **Check for sanitizers along each taint path.** A path is safe if taint passes through a recognized sanitizer before reaching the sink:
   - Parameterized queries / prepared statements neutralize SQL injection
   - `html.escape()`, `DOMPurify.sanitize()`, template auto-escaping neutralize XSS
   - URL validation / allowlists neutralize SSRF
   - `shlex.quote()`, `shellescape` neutralize command injection
   - If no sanitizer is found on the path, flag it as a vulnerability

7. **Classify each finding by severity and confidence.** Rate based on: (a) directness of the path (fewer intermediary functions = higher confidence), (b) whether the sink is exploitable without additional conditions, (c) whether partial sanitization exists but is insufficient.

8. **Report each vulnerability with the full taint path.** For each finding, output: the source location (file:line), every propagation step (variable assignments, function calls, field accesses), and the sink location. Include the vulnerability class (SQLi, RCE, SSRF, etc.) and a remediation recommendation.

9. **Cross-check for false positives.** Review flagged paths for: implicit sanitization not recognized (e.g., custom validation libraries, type coercion that restricts input), dead code paths, or framework-level protections (e.g., ORM default parameterization). Downgrade or remove findings accordingly.

10. **Summarize results grouped by severity and vulnerability class.** Provide an actionable report with the most critical findings first, including code snippets showing the vulnerable path and the recommended fix.

## Concrete Examples

**Example 1: Python Flask SQL Injection**

User: "Check this Flask endpoint for SQL injection vulnerabilities."

```python
# app.py
@app.route('/users')
def get_users():
    name = request.args.get('name')        # SOURCE: user-controlled query param
    filter_str = build_filter(name)          # propagation through function call
    users = db.execute(f"SELECT * FROM users WHERE {filter_str}")  # SINK: raw SQL
    return jsonify(users)

def build_filter(value):
    return f"name = '{value}'"               # taint passes through: returns tainted string
```

Approach:
1. Source identified: `request.args.get('name')` at app.py:4 -- Flask query parameter
2. Propagation: `name` passed to `build_filter()` at app.py:5. Inside `build_filter`, `value` is tainted, and the f-string return at app.py:9 propagates taint to the return value
3. `filter_str` receives tainted return value at app.py:5
4. Sink reached: `db.execute()` at app.py:6 with tainted f-string -- no parameterization
5. No sanitizer found on path

Output:
```
VULNERABILITY: SQL Injection (High Severity, High Confidence)
Path: request.args.get('name') [app.py:4]
  -> build_filter(name) [app.py:5]
  -> f"name = '{value}'" [app.py:9]
  -> db.execute(f"SELECT * FROM users WHERE {filter_str}") [app.py:6]
Remediation: Use parameterized queries:
  db.execute("SELECT * FROM users WHERE name = ?", (name,))
```

**Example 2: Go Gin Command Injection via Channel**

User: "Audit this Go service that processes jobs through a channel."

```go
// handler.go
func HandleJob(c *gin.Context) {
    cmd := c.Query("cmd")              // SOURCE: user-controlled query param
    jobChan <- cmd                      // taint propagates into channel
}

// worker.go
func Worker(jobChan <-chan string) {
    for job := range jobChan {
        exec.Command("sh", "-c", job).Run()  // SINK: OS command execution
    }
}
```

Approach:
1. Source: `c.Query("cmd")` at handler.go:3 -- Gin query parameter
2. Go channel propagation: `jobChan <- cmd` sends tainted value into channel at handler.go:4
3. Channel receive: `job := range jobChan` at worker.go:3 receives tainted value (Go channel rule: send taints channel, receive from tainted channel taints receiver)
4. Sink: `exec.Command("sh", "-c", job).Run()` at worker.go:4 -- tainted string in shell command
5. No sanitizer on path

Output:
```
VULNERABILITY: Command Injection (Critical Severity, High Confidence)
Path: c.Query("cmd") [handler.go:3]
  -> jobChan <- cmd [handler.go:4]  (channel send)
  -> job := range jobChan [worker.go:3]  (channel receive)
  -> exec.Command("sh", "-c", job).Run() [worker.go:4]
Remediation: Validate cmd against an allowlist of permitted commands.
  Never pass user input directly to shell execution.
```

**Example 3: JavaScript Express SSRF via Promise Chain**

User: "Is there an SSRF risk in this Express middleware?"

```javascript
// middleware.js
app.get('/proxy', async (req, res) => {
    const url = req.query.target;                   // SOURCE
    const validated = validateFormat(url);
    const response = await fetch(validated);         // SINK: HTTP request with tainted URL
    const data = await response.json();
    res.json(data);
});

function validateFormat(input) {
    return input.startsWith('http') ? input : `http://${input}`;  // NOT a real sanitizer
}
```

Approach:
1. Source: `req.query.target` at middleware.js:3
2. Propagation: passed to `validateFormat()` which prepends `http://` but does not restrict the host -- taint persists through return value
3. `validated` is tainted at middleware.js:4
4. Sink: `fetch(validated)` at middleware.js:5 -- HTTP request to user-controlled URL
5. `validateFormat` is NOT a valid sanitizer (does not check against allowlist or block internal IPs)

Output:
```
VULNERABILITY: Server-Side Request Forgery (High Severity, High Confidence)
Path: req.query.target [middleware.js:3]
  -> validateFormat(url) [middleware.js:4]
  -> return input (not sanitized) [middleware.js:11]
  -> fetch(validated) [middleware.js:5]
Remediation: Validate URL against a domain allowlist and block
  internal/private IP ranges (127.0.0.0/8, 10.0.0.0/8, 172.16.0.0/12, 169.254.0.0/16).
```

## Best Practices

- **Do:** Always start by identifying the framework to determine the correct source definitions. A Spring `@RequestBody` and a Flask `request.json` are semantically equivalent sources, but you must recognize each framework's idiom.
- **Do:** Follow taint through function boundaries interprocedurally. A common pattern is user input passed to a "helper" or "builder" function that returns a tainted string -- the taint does not stop at function boundaries.
- **Do:** Apply language-specific propagation rules. Go channels, JavaScript promises/prototypes, and Python generators are real propagation vectors that single-language mental models miss.
- **Do:** Distinguish real sanitizers from format validators. A function that checks string format (e.g., `startsWith('http')`) is not a sanitizer. Parameterized queries, HTML escaping libraries, and allowlist validation are sanitizers.
- **Avoid:** Flagging paths that pass through ORM parameterized queries (e.g., SQLAlchemy `filter_by()`, GORM `Where("name = ?", val)`) as SQL injection. These are built-in sanitizers.
- **Avoid:** Treating all string operations as taint propagation without checking semantics. Integer parsing (`parseInt`, `strconv.Atoi`) that rejects non-numeric input effectively sanitizes command/SQL injection for that path.

## Error Handling

- **Unresolvable call targets (dynamic dispatch, reflection):** When a function call cannot be statically resolved (e.g., `getattr(obj, method_name)(user_input)`), flag the path as "potential vulnerability -- requires manual review" with a note about the unresolvable dispatch.
- **Third-party library black boxes:** When taint passes through an external library function with unknown internals, assume taint propagates through (conservative/sound analysis). Note the assumption in the finding so reviewers can verify.
- **Circular taint paths:** If taint analysis encounters a cycle (e.g., recursive function processing tainted data), bound the analysis depth and report the path up to the depth limit with a note.
- **Framework version differences:** Source/sink definitions change between framework versions (e.g., Express 4 vs 5). If version is unknown, apply the union of known source/sink definitions and note the ambiguity.
- **Partial codebase analysis:** When only reviewing a subset of files (e.g., a PR diff), note that taint sources or sinks may exist outside the reviewed scope, and flag inter-file data flows that cross the analysis boundary.

## Limitations

- **Custom sanitizers are the primary source of false positives.** YASA's own deployment found that 80.6% of false positives came from unrecognized sanitization mechanisms. When analyzing code, Claude similarly may not recognize project-specific validation libraries or custom escaping functions.
- **Dynamic language features limit precision.** `eval()`, `getattr()` with dynamic strings, JavaScript `Proxy` objects, and Go reflection create paths that static analysis cannot fully resolve.
- **Cross-service taint tracking is not covered.** If tainted data flows from a Python microservice to a Go microservice via an HTTP API or message queue, each service must be analyzed separately with manual bridging of the taint at service boundaries.
- **This is a manual analysis methodology, not a tool.** Claude performs the analysis by reading code and reasoning about data flow. It does not execute the code or use a formal solver. For large codebases (>10K lines), focus the analysis on entry points and known dangerous patterns rather than attempting exhaustive coverage.
- **Implicit framework sanitization may be missed.** Some frameworks auto-escape template output (e.g., Jinja2, Go `html/template`) or auto-parameterize queries. Claude should check framework documentation when uncertain.

## Reference

[YASA: Scalable Multi-Language Taint Analysis on the Unified AST at Ant Group](https://arxiv.org/abs/2601.17390v1) -- Look for: the UAST node taxonomy (Table 2), taint propagation rules (Section 4.3), framework-specific checker definitions (Section 4.4), and the language-specific semantic models for JavaScript prototype/promise propagation and Go channel propagation (Section 4.2).