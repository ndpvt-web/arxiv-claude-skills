---
name: "from-detection-prevention-explaining"
description: "Proactively identify security-critical code regions and generate prevention-oriented explanations before vulnerabilities are introduced. Use when: 'review this code for security-critical areas', 'explain security risks in my methods', 'find security-sensitive code before bugs happen', 'proactive security review of my codebase', 'highlight authentication and data access risks', 'prevent vulnerabilities in this module'."
---

# Proactive Security-Critical Code Identification and Prevention Explanations

This skill enables Claude to shift security analysis from **reactive detection to proactive prevention**. Instead of waiting for vulnerabilities to appear and then flagging them, Claude identifies methods and code regions that implement security-critical functionality -- data access, authentication, input handling, authorization, cryptography -- using structural code metrics (fan-in, fan-out, coupling, complexity). For each identified region, Claude generates prevention-oriented explanations that tell the developer *why* the code is security-sensitive and *how* to implement it securely, before any vulnerability is introduced. This approach is based on the technique from Krishnamurthy et al. (2026).

## When to Use

- When a user asks for a security review of a codebase or module and wants guidance *before* shipping, not just a bug list
- When reviewing controller methods, service layers, or repository classes in web applications (Spring, Django, Express, Rails, etc.)
- When a user is writing new authentication, authorization, or session management code and wants to get it right the first time
- When a user submits code that handles user input, database queries, file I/O, or external API calls and needs security guidance
- When a user asks "what could go wrong here?" or "is this code security-sensitive?" about a method or class
- When onboarding to a new codebase and needing to identify which methods carry the highest security risk
- When performing pre-commit or pre-PR review focused on preventing security regressions

## Key Technique

Traditional security tools (SAST, DAST, SCA) operate in a **detect-then-fix** cycle: code is written, a scanner finds a vulnerability, and the developer remediates. This is costly -- the vulnerability already exists in the codebase, and fixing it requires context-switching back to code that may have been written days or weeks ago. The paper proposes inverting this: use **code-level structural metrics** to flag methods that are *likely* to be security-critical based on their structural role in the application, then generate **prevention-oriented explanations** that guide the developer toward secure implementation from the start.

The structural metrics that signal security criticality are: **high fan-in** (many callers depend on this method, so a flaw propagates widely), **high fan-out** (the method calls many other methods, increasing its attack surface), **high coupling** (tight interdependencies with other classes, especially those handling I/O or persistence), and **elevated cyclomatic complexity** (complex control flow makes it harder to reason about all code paths, increasing the chance of missed edge cases). Methods that score highly on these structural indicators AND belong to security-relevant categories (data access, authentication, input validation, authorization, cryptographic operations) are flagged as security-critical.

The prevention explanation is the key differentiator. Rather than saying "SQL injection found on line 42," the explanation says: "This method constructs database queries from parameters passed by 5 calling methods. Because it has high fan-in and directly accesses the persistence layer, any input validation failure upstream will propagate here. Use parameterized queries exclusively, validate input types at this method's boundary regardless of caller behavior, and ensure the method enforces least-privilege database access." The explanation contextualizes *why* the code is risky and *what* to do about it proactively.

## Step-by-Step Workflow

1. **Identify the scope**: Determine whether the user wants a full-codebase scan, a single module review, or analysis of a specific method. For full-codebase scans, start with entry points (controllers, API handlers, CLI parsers) and work inward.

2. **Classify methods into security-critical categories**: For each method in scope, determine if it belongs to one or more of these categories:
   - **Data access**: Methods that read/write to databases, files, caches, or external storage
   - **Authentication**: Methods involved in login, token generation/validation, password handling, session management
   - **Authorization**: Methods that check permissions, roles, or access control lists
   - **Input handling**: Methods that receive, parse, or transform user-supplied data (HTTP params, form data, file uploads, API payloads)
   - **Cryptographic operations**: Methods that encrypt, decrypt, hash, sign, or verify data
   - **Output encoding**: Methods that render data to HTML, JSON, XML, or other formats sent to clients

3. **Compute structural risk indicators** for each method:
   - **Fan-in**: Count how many other methods call this one. High fan-in (relative to the codebase) means a flaw here affects many callers.
   - **Fan-out**: Count how many other methods this one calls. High fan-out means a larger attack surface and more potential for misuse of dependencies.
   - **Coupling**: Assess how many external classes/modules this method depends on, especially I/O, persistence, or network classes.
   - **Complexity**: Evaluate branching complexity (if/else chains, switch statements, nested loops). High complexity means more untested paths.

4. **Rank methods by security criticality**: Combine the category classification (step 2) with the structural indicators (step 3). Methods that are both in a security-critical category AND have high structural risk indicators are the highest priority. Present them in ranked order.

5. **Generate a prevention-oriented explanation for each flagged method**: For each security-critical method, produce an explanation that includes:
   - **What it does**: A one-sentence summary of the method's purpose
   - **Why it is security-sensitive**: Which category it belongs to and which structural indicators are elevated
   - **Specific risks**: What classes of vulnerability could arise (e.g., SQLi, XSS, broken auth, IDOR, path traversal)
   - **Prevention guidance**: Concrete, actionable steps to implement the method securely, referencing the language/framework in use
   - **Code-level recommendation**: A brief code snippet or pattern showing the secure approach

6. **Check for missing security controls**: For each flagged method, verify whether the codebase already implements the recommended controls (parameterized queries, input validation, output encoding, rate limiting, etc.). Report gaps.

7. **Assess upstream and downstream propagation**: For methods with high fan-in, check whether callers sanitize inputs before passing them. For methods with high fan-out, check whether callees handle errors securely. Report any trust boundary violations.

8. **Summarize findings**: Present a ranked table of security-critical methods with their category, structural risk level (high/medium/low), and a one-line prevention recommendation. Follow with detailed explanations for the top-priority items.

9. **Provide implementation templates**: For the most common security patterns found (e.g., parameterized queries, bcrypt hashing, CSRF tokens), provide copy-pasteable secure code templates tailored to the project's language and framework.

## Concrete Examples

**Example 1: Spring Boot Controller Review**

User: "Review this Spring Boot controller for security-critical areas."

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @Autowired private UserService userService;
    @Autowired private JdbcTemplate jdbcTemplate;

    @GetMapping("/search")
    public List<User> searchUsers(@RequestParam String query) {
        String sql = "SELECT * FROM users WHERE name LIKE '%" + query + "%'";
        return jdbcTemplate.query(sql, new UserRowMapper());
    }

    @PostMapping("/login")
    public ResponseEntity<String> login(@RequestBody LoginRequest request) {
        User user = userService.findByUsername(request.getUsername());
        if (user != null && user.getPassword().equals(request.getPassword())) {
            String token = UUID.randomUUID().toString();
            return ResponseEntity.ok(token);
        }
        return ResponseEntity.status(401).body("Invalid credentials");
    }

    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userService.deleteById(id);
    }
}
```

Approach:
1. Classify methods: `searchUsers` = Input handling + Data access. `login` = Authentication. `deleteUser` = Authorization + Data access.
2. Structural indicators: `searchUsers` has high coupling (direct JdbcTemplate usage in controller), string concatenation in SQL. `login` has plaintext password comparison, no rate limiting, predictable token generation. `deleteUser` has no authorization check.
3. Generate prevention explanations for each.

Output:

| Method | Category | Risk Level | Key Issue |
|--------|----------|------------|-----------|
| `searchUsers` | Input + Data Access | **CRITICAL** | SQL injection via string concatenation |
| `login` | Authentication | **CRITICAL** | Plaintext password comparison, weak token |
| `deleteUser` | Authorization + Data | **HIGH** | No authorization check on destructive operation |

**`searchUsers` -- Prevention Explanation:**
This method receives raw user input (`query` parameter) and directly concatenates it into a SQL string. It has high coupling (controller directly uses JdbcTemplate instead of going through a repository layer) and processes untrusted input with zero validation. Any attacker can inject arbitrary SQL.

*Prevention guidance:*
- Use parameterized queries exclusively. Never concatenate user input into SQL.
- Add input validation: limit `query` length, restrict to alphanumeric characters.
- Move data access to a repository class to enforce separation of concerns.

```java
@GetMapping("/search")
public List<User> searchUsers(@RequestParam @Size(max = 100) String query) {
    return userRepository.findByNameContainingIgnoreCase(query);
}
```

**`login` -- Prevention Explanation:**
This method handles authentication but compares passwords in plaintext (`equals()`), meaning passwords are stored unhashed in the database. The token is a UUID, which is not cryptographically suitable for session tokens. No rate limiting or account lockout is present.

*Prevention guidance:*
- Use bcrypt/scrypt/argon2 for password hashing. Compare with `BCryptPasswordEncoder.matches()`.
- Generate tokens with `SecureRandom`, not `UUID.randomUUID()`. Prefer JWT with proper signing or Spring Security's built-in session management.
- Add rate limiting (e.g., Bucket4j or Spring Security's `AuthenticationFailureHandler`).

**`deleteUser` -- Prevention Explanation:**
This method performs a destructive operation (user deletion) with no authorization check. Any authenticated (or unauthenticated) user can delete any account by guessing IDs (IDOR vulnerability). High fan-in risk: if other parts of the app call `deleteUser`, they all inherit this authorization gap.

*Prevention guidance:*
- Add `@PreAuthorize("hasRole('ADMIN')")` or verify the requesting user owns the resource.
- Validate that the `id` corresponds to a deletable entity before proceeding.

---

**Example 2: Express.js Input Handling Review**

User: "Check my Express route handlers for security issues before I deploy."

```javascript
app.post('/api/profile', (req, res) => {
  const { name, bio, website } = req.body;
  db.query(`UPDATE profiles SET name='${name}', bio='${bio}', website='${website}' WHERE id=${req.user.id}`);
  res.json({ success: true });
});

app.get('/api/file', (req, res) => {
  const filePath = path.join('/uploads', req.query.name);
  res.sendFile(filePath);
});
```

Output:

| Method | Category | Risk Level | Key Issue |
|--------|----------|------------|-----------|
| `POST /api/profile` | Input + Data Access | **CRITICAL** | SQL injection in all 3 fields + stored XSS in bio/name |
| `GET /api/file` | Input handling | **CRITICAL** | Path traversal -- attacker can read arbitrary files |

**`POST /api/profile` -- Prevention Explanation:**
This handler accepts three user-controlled fields and interpolates them directly into SQL. It has high fan-out (touches the database and sends a response) and processes completely unvalidated input. Beyond SQL injection, any HTML/JS in `name` or `bio` will be stored and potentially rendered to other users (stored XSS).

*Prevention guidance:*
```javascript
app.post('/api/profile', (req, res) => {
  const { name, bio, website } = req.body;
  // Validate input
  if (!name || name.length > 100) return res.status(400).json({ error: 'Invalid name' });
  if (website && !isValidUrl(website)) return res.status(400).json({ error: 'Invalid URL' });
  // Parameterized query
  db.query('UPDATE profiles SET name=?, bio=?, website=? WHERE id=?',
    [name, bio, website, req.user.id]);
  res.json({ success: true });
});
```

**`GET /api/file` -- Prevention Explanation:**
`path.join('/uploads', req.query.name)` does NOT prevent path traversal. An attacker can send `name=../../etc/passwd` to read arbitrary files on the server. This method has high coupling to the filesystem and takes unsanitized input directly.

*Prevention guidance:*
```javascript
app.get('/api/file', (req, res) => {
  const safeName = path.basename(req.query.name); // strip directory components
  const filePath = path.join('/uploads', safeName);
  if (!filePath.startsWith('/uploads/')) return res.status(403).send('Forbidden');
  res.sendFile(filePath);
});
```

---

**Example 3: Full Codebase Triage**

User: "I'm onboarding to this Django project. Which methods should I be most careful about from a security perspective?"

Approach:
1. Scan all views, models, forms, and middleware for security-critical categories.
2. Compute structural indicators: identify views with the most URL routes (high fan-in), models with raw SQL (high coupling to DB), forms with unvalidated fields.
3. Rank and produce a triage report.

Output:
```
Security-Critical Methods Triage (ranked by risk):

1. [CRITICAL] views.py:PaymentView.process_payment (L:142)
   Category: Data access + Input handling
   Fan-in: 3 URL routes, called by 2 internal services
   Issue: Constructs charge amount from user-supplied form data without server-side validation
   Prevention: Validate amount server-side, use Decimal type, compare against order total from DB

2. [HIGH] views.py:LoginView.post (L:58)
   Category: Authentication
   Fan-in: 1 URL route, but affects all users
   Issue: No rate limiting, no account lockout, logs passwords at DEBUG level
   Prevention: Add django-axes or django-defender, remove password logging, use constant-time comparison

3. [HIGH] models.py:Report.generate_pdf (L:201)
   Category: Input handling + Output
   Fan-out: Calls 6 methods including subprocess for PDF generation
   Issue: Filename derived from user input passed to subprocess
   Prevention: Sanitize filename with allowlist, use shlex.quote(), avoid subprocess if possible

4. [MEDIUM] middleware.py:CORSMiddleware.process_response (L:15)
   Category: Authorization
   Fan-in: Every request passes through this
   Issue: Access-Control-Allow-Origin set to '*' in production
   Prevention: Restrict to specific origins, use django-cors-headers with CORS_ALLOWED_ORIGINS
```

## Best Practices

- **Do**: Always check for the *absence* of security controls, not just the *presence* of vulnerabilities. A method that handles authentication but has no rate limiting is security-critical even if it has no exploitable bug today.
- **Do**: Prioritize methods at trust boundaries -- where untrusted data enters the application (HTTP handlers, message queue consumers, file parsers) or where sensitive data exits (database writes, API responses, log statements).
- **Do**: Consider fan-in when assessing impact. A vulnerable utility method called by 20 other methods is more critical than a vulnerable method called by 1.
- **Do**: Tailor prevention guidance to the specific language and framework. Recommend `@PreAuthorize` for Spring, `@login_required` for Django, middleware patterns for Express.
- **Avoid**: Reporting every method in the codebase. Focus on methods that are both structurally significant (high fan-in/out, high coupling, high complexity) AND in a security-critical category. A complex math utility with no I/O is not security-critical.
- **Avoid**: Generic advice like "validate your inputs." Be specific: what inputs, what validation (type, length, format, allowlist), and where in the code it should happen.

## Error Handling

- **Metric ambiguity**: When you cannot determine fan-in/fan-out from a code snippet alone (e.g., the user only shares one file), state the assumption explicitly: "I cannot see callers of this method. If it is called from multiple routes, the risk increases."
- **Framework-specific gaps**: If you are unfamiliar with a framework's security features, recommend the general principle (e.g., parameterized queries) rather than a framework-specific API that may not exist.
- **False positives**: Some structurally complex methods are not security-critical (e.g., a complex rendering function with no I/O). When flagging a method, verify it touches at least one security-critical category before including it.
- **Incomplete code**: If the user provides a partial file, note which analysis steps are limited and what additional context would improve the review (e.g., "I would need to see the middleware configuration to assess authentication enforcement").

## Limitations

- Structural metrics (fan-in, fan-out, coupling, complexity) capture the *shape* of code, not its *semantics*. A method with low complexity can still be security-critical if it handles a sensitive operation in a single line (e.g., `eval(user_input)`). Always combine structural analysis with category-based classification.
- This approach works best for server-side web applications with clear controller/service/repository layers. It is less effective for event-driven architectures, microservices with cross-service call chains, or client-side code where the trust boundary is the entire application.
- Prevention explanations assume the developer will act on them. They do not replace automated enforcement (SAST rules, pre-commit hooks, security linters).
- The technique identifies *where* security-critical code lives but cannot guarantee completeness -- business logic vulnerabilities (e.g., allowing negative quantities in an order) require domain knowledge beyond structural metrics.

## Reference

**Paper**: Krishnamurthy, R., Johnson, O., Piskachev, G., & Bodden, E. (2026). *From Detection to Prevention: Explaining Security-Critical Code to Avoid Vulnerabilities*. arXiv:2602.00711v1. [https://arxiv.org/abs/2602.00711v1](https://arxiv.org/abs/2602.00711v1)

Look for: The metric-based method for identifying security-critical code regions using fan-in, fan-out, coupling, and complexity; the taxonomy of security-critical categories; and the LLM prompting strategy for generating prevention-oriented (not detection-oriented) explanations.