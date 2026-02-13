---
name: "constitutional-spec-driven-development-enforcing"
description: "Enforce security by construction in AI-generated code using Constitutional Spec-Driven Development (CSDD). Creates a versioned security constitution document mapping CWE/OWASP vulnerabilities to enforceable constraints, then generates code that satisfies those constraints with full traceability. Triggers: 'generate secure code', 'create a security constitution', 'build with security by construction', 'CSDD for my project', 'enforce CWE constraints', 'secure code generation with traceability'"
---

# Constitutional Spec-Driven Development: Enforcing Security by Construction

This skill enables Claude to apply Constitutional Spec-Driven Development (CSDD) — a methodology that embeds non-negotiable security principles into the specification layer *before* code generation begins. Instead of generating code first and scanning for vulnerabilities later, CSDD produces a machine-readable "constitution" of security constraints derived from CWE/MITRE Top 25 and regulatory frameworks, then generates code that satisfies those constraints by construction. The result is code with 73% fewer security defects and a full compliance traceability matrix mapping every principle to specific code locations.

## When to Use

- When the user asks to generate a new service, API, or microservice and security matters (especially in regulated domains like finance, healthcare, government)
- When the user says "generate secure code" or "build this securely" without specifying a security methodology
- When building authentication, authorization, payment processing, or user data handling features
- When the user wants to enforce OWASP Top 10 or CWE Top 25 protections systematically
- When the user asks for a security review framework or compliance traceability for generated code
- When migrating from "vibe coding" to a more disciplined secure development process
- When the user needs to demonstrate security compliance with traceability from requirements to code

## Key Technique

CSDD inverts the traditional generate-then-scan workflow. A **security constitution** is authored first — a structured document where each principle has an identifier (e.g., SEC-002), a CWE reference, an RFC 2119 enforcement level (MUST/SHOULD/MAY), a concrete constraint, an implementation pattern, and a rationale. This constitution becomes the governing document for all subsequent code generation. When generating code, only 3-5 relevant principles are selected per generation request to stay within context limits, and the AI is instructed to satisfy each constraint using the specified implementation pattern.

The critical insight is that **proactive specification outperforms reactive verification**. Rather than generating unconstrained code and running static analysis to find 11+ CWE violations, CSDD front-loads the security thinking into a reusable artifact. The constitution is versioned alongside source code, enabling governance — any change to security constraints goes through the same review process as code changes. The methodology produces a **compliance traceability matrix** that maps each principle to exact file paths and line numbers, giving auditors and reviewers a complete chain from requirement to implementation.

The methodology is domain-agnostic. While the paper demonstrates it through banking microservices, the constitutional framework extends to architectural principles (layered separation, dependency inversion), design patterns (repository, factory), organizational guidelines (naming, logging), and performance constraints (pagination, circuit breakers).

## Step-by-Step Workflow

1. **Identify the domain's threat model.** Determine which CWE vulnerabilities and regulatory requirements apply. For a web API, start with CWE-79 (XSS), CWE-89 (SQL injection), CWE-287 (authentication), CWE-862 (authorization), CWE-20 (input validation). For financial systems, add CWE-190 (integer overflow), CWE-312 (cleartext storage), CWE-522 (weak credentials).

2. **Author the security constitution.** For each identified threat, write a constitutional principle using this exact template:
   ```
   Principle: SEC-NNN
   CWE: CWE-XXX
   Level: MUST | SHOULD | MAY
   Constraint: [What code must/must not do]
   Pattern: [Specific implementation technique]
   Rationale: [Why this constraint exists]
   ```
   Aim for 10-20 principles. Be specific — "parameterized queries via ORM exclusively" not "prevent SQL injection."

3. **Select the technology stack with constitutional rationale.** Choose each technology because it *enables* specific constitutional principles. Document why: e.g., "SQLAlchemy 2.0 — parameterized queries satisfy SEC-002; Pydantic v2 — declarative schema validation satisfies SEC-006; passlib+bcrypt — adaptive hashing with cost factor ≥12 satisfies SEC-009."

4. **Write feature specifications (spec.md).** For each feature, describe functional requirements alongside the constitutional constraints that apply. Explicitly reference principle IDs: "Account retrieval MUST verify customer ownership per SEC-010."

5. **Generate implementation plans (plan.md).** Break each feature into components and map each component to the constitutional principles it must satisfy. Identify all code locations where security enforcement will occur.

6. **Generate code in constitutional context.** For each generation request, include only the 3-5 relevant constitutional principles in the prompt context. Instruct the model: "Generate code satisfying the following constitutional constraints" followed by the selected principles with their patterns.

7. **Verify constitutional compliance inline.** After generating each code unit, check that every referenced principle is satisfied. Look for the specific implementation pattern — not just the absence of a vulnerability, but the presence of the mandated technique (e.g., confirm bcrypt with cost ≥12, not just "password is hashed").

8. **Build the compliance traceability matrix.** Create a table mapping each principle to: source file, line numbers, implementation technique used. Every principle must have at least one code location. Zero gaps is the target.

9. **Handle violations through regeneration, not patching.** If a generated code block violates a principle, regenerate with the constraint made more explicit — do not manually patch. The paper shows regeneration averages 1.4 iterations vs. 3.2 for manual patching.

10. **Version the constitution alongside code.** Store the constitution in the repository root (e.g., `CONSTITUTION.md` or `constitution.yaml`). Changes to constitutional principles require the same review rigor as production code changes.

## Concrete Examples

**Example 1: Secure Banking API**

User: "Build me a customer management API with registration, login, and account retrieval"

Approach:
1. Author constitution targeting CWE-89 (injection), CWE-287 (auth), CWE-522 (credentials), CWE-862 (authorization), CWE-20 (input validation), CWE-200 (info exposure):

```yaml
# constitution.yaml
principles:
  - id: SEC-001
    cwe: CWE-89
    level: MUST
    constraint: All database queries use parameterized statements via ORM
    pattern: SQLAlchemy ORM queries only; no raw SQL or f-string interpolation
    rationale: Prevents SQL injection attacks on customer data

  - id: SEC-002
    cwe: CWE-522
    level: MUST
    constraint: Passwords hashed with adaptive algorithm before storage
    pattern: bcrypt via passlib with cost factor >= 12
    rationale: Protects credentials if database is compromised

  - id: SEC-003
    cwe: CWE-287
    level: MUST
    constraint: API endpoints require valid JWT bearer token except /health and /register
    pattern: OAuth2 with python-jose HS256, 15-minute token expiry
    rationale: Prevents unauthorized access to customer resources

  - id: SEC-004
    cwe: CWE-862
    level: MUST
    constraint: Resource access requires ownership verification
    pattern: Compare authenticated customer_id against resource owner before returning data
    rationale: Prevents IDOR attacks on account data

  - id: SEC-005
    cwe: CWE-20
    level: MUST
    constraint: All API inputs validated against strict schemas
    pattern: Pydantic v2 models with field constraints, regex patterns, type enforcement
    rationale: Rejects malformed input before processing

  - id: SEC-006
    cwe: CWE-200
    level: MUST
    constraint: Error responses contain no implementation details
    pattern: Generic error messages with correlation IDs; details to server logs only
    rationale: Prevents information leakage to attackers
```

2. Generate registration endpoint satisfying SEC-001, SEC-002, SEC-005:

```python
# Satisfies SEC-005: Pydantic schema with strict constraints
class CustomerCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)
    full_name: str = Field(min_length=1, max_length=200)
    phone: str = Field(pattern=r"^\+?[1-9]\d{1,14}$")

# Satisfies SEC-002: bcrypt with cost factor 12
pwd_context = CryptContext(schemes=["bcrypt"], bcrypt__rounds=12)

@router.post("/customers", status_code=201)
async def register_customer(data: CustomerCreate, db: Session = Depends(get_db)):
    # SEC-001: ORM parameterized query, no raw SQL
    existing = db.query(Customer).filter(Customer.email == data.email).first()
    if existing:
        raise HTTPException(status_code=409, detail="Registration failed")  # SEC-006: no details
    customer = Customer(
        email=data.email,
        password_hash=pwd_context.hash(data.password),  # SEC-002
        full_name=data.full_name,
        phone=data.phone,
    )
    db.add(customer)
    db.commit()
    return {"id": customer.id, "email": customer.email}
```

3. Traceability matrix output:

```
| Principle | CWE     | File                | Lines   | Technique                    |
|-----------|---------|---------------------|---------|------------------------------|
| SEC-001   | CWE-89  | api/customers.py    | 18, 24  | SQLAlchemy ORM filter        |
| SEC-002   | CWE-522 | api/customers.py    | 12, 22  | passlib bcrypt rounds=12     |
| SEC-003   | CWE-287 | core/security.py    | 27-81   | python-jose HS256 JWT        |
| SEC-004   | CWE-862 | api/accounts.py     | 34-38   | customer_id ownership check  |
| SEC-005   | CWE-20  | schemas/customer.py | 5-12    | Pydantic v2 Field validators |
| SEC-006   | CWE-200 | api/customers.py    | 20      | Generic error message        |
```

**Example 2: Adding a Feature to an Existing Constitutionally-Governed Project**

User: "Add a transaction transfer endpoint to the banking API"

Approach:
1. Read the existing constitution. Identify relevant principles: SEC-001 (injection), SEC-003 (auth), SEC-004 (authorization), SEC-005 (validation). Add a new principle for financial amounts:

```yaml
  - id: SEC-007
    cwe: CWE-190
    level: MUST
    constraint: Financial amounts use decimal types with explicit precision
    pattern: "Decimal field with gt=0, le=1000000, decimal_places=2; never float"
    rationale: Prevents overflow and rounding errors in monetary calculations
```

2. Generate with SEC-001, SEC-003, SEC-004, SEC-005, SEC-007 in context:

```python
class TransferRequest(BaseModel):
    from_account: str = Field(pattern=r"^[A-Z0-9]{10}$")  # SEC-005
    to_account: str = Field(pattern=r"^[A-Z0-9]{10}$")    # SEC-005
    amount: Decimal = Field(gt=0, le=1000000, decimal_places=2)  # SEC-007

@router.post("/transfers", dependencies=[Depends(get_current_customer)])  # SEC-003
async def transfer(req: TransferRequest, customer_id: int = Depends(get_current_customer), db: Session = Depends(get_db)):
    # SEC-004: Verify ownership of source account
    source = db.query(Account).filter(
        Account.number == req.from_account,
        Account.customer_id == customer_id  # SEC-001 + SEC-004
    ).first()
    if not source:
        raise HTTPException(status_code=404, detail="Account not found")  # SEC-006
    # ... execute transfer with db transaction
```

3. Update the traceability matrix with new entries for the transfer endpoint.

**Example 3: Creating a Constitution for a Non-Financial Domain**

User: "I'm building a healthcare patient portal. Help me set up secure code generation."

Approach:
1. Identify domain-specific threats: HIPAA compliance, PHI exposure, access controls, audit logging.
2. Author constitution:

```yaml
principles:
  - id: PHI-001
    cwe: CWE-312
    level: MUST
    constraint: Patient health information encrypted at rest using AES-256
    pattern: Database column-level encryption via SQLAlchemy-Utils EncryptedType
    rationale: HIPAA §164.312(a)(2)(iv) requires encryption of ePHI

  - id: PHI-002
    cwe: CWE-532
    level: MUST
    constraint: Audit logs must never contain PHI or PII
    pattern: Log patient_id only; exclude names, SSN, diagnoses from all log statements
    rationale: HIPAA audit trail requirements without PHI exposure

  - id: PHI-003
    cwe: CWE-862
    level: MUST
    constraint: Provider can only access patients in their active care assignment
    pattern: Join through care_assignment table with active status check
    rationale: Minimum necessary access principle under HIPAA
```

3. Proceed with feature specs and code generation using these as governing constraints.

## Best Practices

- **Do:** Write constraints as specific implementation patterns, not goals. "bcrypt with cost ≥12" enforces; "use strong hashing" does not.
- **Do:** Include rationale for every principle. When edge cases arise, the rationale enables contextual judgment rather than blind compliance.
- **Do:** Select only 3-5 relevant principles per code generation request. Including the entire constitution wastes context and dilutes enforcement.
- **Do:** Regenerate violating code from scratch rather than manually patching. Regeneration produces more coherent and consistently secure output.
- **Avoid:** Using float types for any monetary or precision-sensitive values. The constitution should mandate Decimal with explicit precision.
- **Avoid:** Logging passwords, tokens, API keys, or PII in any form. Constitutional principles should explicitly exclude sensitive fields from audit logs.
- **Avoid:** Generic error messages that still leak information (e.g., "User not found" vs. "Invalid credentials"). The constitution should specify error response patterns.
- **Avoid:** Writing constitutional principles so broad they cannot be mechanically verified (e.g., "code should be secure"). Every principle must be checkable against specific code patterns.

## Error Handling

- **Principle conflicts:** If two constitutional principles create contradictory requirements (e.g., detailed audit logging vs. no PII in logs), resolve by adding a specific exception clause to one principle and documenting the resolution.
- **AI ignores a constraint:** Re-prompt with the specific violated principle quoted verbatim, preceded by "CRITICAL CONSTRAINT VIOLATION:" and the code that violated it. Regenerate the affected function entirely.
- **Missing coverage:** If the traceability matrix has a principle with no code location, either the feature requiring that principle hasn't been built yet (acceptable) or the implementation silently omitted it (create a task to add enforcement).
- **Context window overflow:** If the constitution exceeds context capacity, partition principles by domain (auth, input validation, data handling) and include only the relevant partition per request.
- **Prompt injection risk:** The constitution document itself can be a vector for prompt injection. Store it with the same access controls as production configuration. Never accept constitutional amendments from untrusted input.

## Limitations

- CSDD addresses **known vulnerability classes** from CWE/OWASP frameworks. Novel or zero-day attack patterns not cataloged in these frameworks will not be covered.
- The methodology handles **technical security flaws** but not business logic vulnerabilities (e.g., allowing negative transfers that credit the attacker's account requires domain-specific rules, not CWE mappings).
- Effectiveness depends on the AI model's ability to **understand and follow natural language constraints**. Complex or ambiguous principles may be inconsistently enforced.
- Constitution authoring requires **security expertise** to write effective principles. A poorly written constitution gives false confidence.
- The framework does not replace runtime security measures — WAFs, rate limiting, intrusion detection, and penetration testing remain necessary.

## Reference

**Paper:** [Constitutional Spec-Driven Development: Enforcing Security by Construction in AI-Assisted Code Generation](https://arxiv.org/abs/2602.02584v1) by Srinivas Rao Marri (2026). Look for: the five-phase development process, the 15 constitutional principles for banking, the compliance traceability matrix format, and the quantitative comparison showing 73% CWE violation reduction. Reference implementation: `github.com/srinivasraom/banking-ms-by-constitution`.