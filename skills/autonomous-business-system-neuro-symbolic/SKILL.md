---
name: "autonomous-business-system-neuro-symbolic"
description: |
  Design and implement neuro-symbolic business automation systems that combine LLM agents with predicate-logic programming and knowledge graphs.
  Applies the AUTOBUS architecture: tasks modeled as networks with pre/post conditions, enterprise data as logic facts, and AI agents that generate
  executable logic programs from natural-language instructions.
  Trigger phrases:
  - "Build a business process automation system"
  - "Create a neuro-symbolic workflow engine"
  - "Design a task orchestration system with logic constraints"
  - "Model business processes as knowledge graphs with AI agents"
  - "Implement an enterprise system that combines LLMs with rule engines"
  - "Build a system where AI agents generate executable business logic"
---

# Autonomous Business System via Neuro-Symbolic AI (AUTOBUS Pattern)

This skill enables Claude to design and implement business automation systems using the AUTOBUS neuro-symbolic architecture from Pang & Sayama (2026). The core idea: LLM-based agents translate natural-language business instructions into predicate-logic programs that execute against a knowledge graph of enterprise data. This combines the flexibility of LLMs (understanding intent, handling ambiguity) with the determinism of logic programming (verifiable constraints, auditable execution). The result is a system where business users define *what* should happen in plain language, and the system generates *how* to execute it with formal guarantees.

## When to Use This Skill

- When the user asks to build a workflow engine or business process automation that needs both flexibility and formal correctness
- When designing a system that must enforce business rules (approval chains, compliance checks, SLA constraints) while adapting to unstructured inputs
- When the user wants to model enterprise processes as task networks with explicit dependencies, preconditions, and postconditions
- When building a knowledge-graph-backed system where entities, relationships, and policies drive automated decision-making
- When the user needs AI agents that generate executable logic (not just text) from business instructions and available tools
- When implementing human-in-the-loop automation where high-impact or ambiguous decisions are escalated to humans
- When refactoring hard-coded business workflows into a declarative, data-driven orchestration layer

## Key Technique: Neuro-Symbolic Task Orchestration

The AUTOBUS architecture solves a fundamental tension: LLMs are excellent at interpreting natural language and unstructured data but cannot guarantee deterministic, verifiable execution of business logic. Conversely, rule engines and logic programming provide determinism but are brittle and require expert programming. AUTOBUS bridges this gap with a three-layer architecture:

**Layer 1 — Enterprise Knowledge Graph as Logic Facts.** All enterprise data (customers, products, policies, organizational structure) is organized as a knowledge graph. Entities become logic facts (`customer("acme", tier: "gold")`), relationships become relational predicates (`reports_to("alice", "bob")`), and business constraints become foundational rules (`requires_approval(order) :- order.amount > 50000`). This semantic grounding means the logic engine reasons over the *actual state* of the business, not stale snapshots.

**Layer 2 — AI Agents Generate Task-Specific Logic Programs.** For each task in a business initiative, an AI agent receives three inputs: (1) task instructions in natural language, (2) relevant enterprise semantics from the knowledge graph, and (3) a catalog of available tools/APIs. The agent synthesizes these into a task-specific logic program — a set of predicates, rules, constraints, and action directives that the logic engine can execute. This is where the "neuro" meets the "symbolic": the LLM's language understanding produces formally executable code.

**Layer 3 — Logic Engine Execution with Human Oversight.** A logic engine (e.g., Prolog-style reasoner, Datalog, or ASP solver) executes the generated programs. It enforces constraints, resolves dependencies between tasks, coordinates API calls, and produces auditable execution traces. Humans define semantics and policies, curate the tool catalog, and are consulted for high-impact or ambiguous decisions — maintaining accountability without bottlenecking routine automation.

## Step-by-Step Workflow

1. **Model the business initiative as a task network.** Decompose the end-to-end process into discrete tasks. For each task, define: a natural-language description, preconditions (what must be true before execution), postconditions (what must be true after), required data entities, evaluation rules (how to judge success/failure), and API-level actions (what the task actually does).

2. **Build the enterprise knowledge graph schema.** Define entity types (Customer, Order, Product, Employee, Policy), their attributes, and relationships. Express business constraints as rules. Use a format that maps cleanly to logic predicates — each entity instance becomes a fact, each constraint becomes a rule.

3. **Translate the knowledge graph into logic facts and foundational rules.** Convert entity instances to ground facts (e.g., `employee(id: "E001", name: "Alice", role: "manager", department: "finance")`). Convert constraints to inference rules (e.g., `can_approve(Person, Order) :- employee(Person, role: "manager"), order_amount(Order, Amt), Amt < approval_limit(Person)`).

4. **Define the tool/API catalog.** For each available tool or API, specify: name, description, input parameters with types, output schema, preconditions for invocation, and side effects. This catalog is what the AI agent draws from when constructing logic programs.

5. **Construct the agent prompt for logic program generation.** Provide the AI agent with: (a) the task instruction in natural language, (b) relevant facts and rules from the knowledge graph, (c) the tool catalog, and (d) a schema for the expected logic program output (predicates, rules, action directives, constraint checks). Instruct the agent to produce a valid logic program, not prose.

6. **Generate the task-specific logic program.** The AI agent outputs a structured logic program containing: data predicates (facts needed for this task), constraint rules (conditions that must hold), action directives (API calls to execute with bound parameters), evaluation predicates (how to assess the outcome), and escalation conditions (when to involve a human).

7. **Validate the generated logic program.** Before execution, check: all referenced predicates exist in the knowledge graph or are defined locally, all tool invocations match the catalog signatures, constraint rules are satisfiable given current facts, and no action directive violates declared policies. Reject and regenerate if validation fails.

8. **Execute via the logic engine.** Run the logic program against the knowledge graph. The engine resolves preconditions, binds variables, enforces constraints, invokes tools/APIs in the correct order, and records an execution trace. If a constraint is violated mid-execution, halt and trigger the escalation path.

9. **Evaluate postconditions and record outcomes.** After execution, verify that all postconditions hold. Log the execution trace (which rules fired, which tools were called, what data changed) for auditability. Update the knowledge graph with new facts produced by the task.

10. **Advance the initiative by checking downstream task preconditions.** With the current task complete, re-evaluate preconditions of dependent tasks in the network. Trigger the next eligible tasks, repeating from step 5 for each.

## Concrete Examples

### Example 1: Order Approval Workflow

**User:** "Build an order approval system where orders over $50K need VP approval, orders over $10K need manager approval, and smaller orders auto-approve. It should check inventory and notify the customer."

**Approach:**

1. Model as a task network: `validate_order → check_inventory → route_approval → notify_customer`
2. Define knowledge graph entities and translate to logic:

```prolog
% Enterprise facts
employee("alice", role: "vp", approval_limit: 200000).
employee("bob", role: "manager", approval_limit: 50000).
product("widget_x", stock: 150, price: 299).

% Foundational rules
approval_tier(Order, "auto") :- order_total(Order, Amt), Amt =< 10000.
approval_tier(Order, "manager") :- order_total(Order, Amt), Amt > 10000, Amt =< 50000.
approval_tier(Order, "vp") :- order_total(Order, Amt), Amt > 50000.

can_approve(Person, Order) :-
    approval_tier(Order, Tier),
    employee(Person, role: Tier),
    order_total(Order, Amt),
    employee(Person, approval_limit: Limit),
    Amt =< Limit.
```

3. AI agent generates task-specific logic program for `route_approval`:

```prolog
% Task: route_approval
% Precondition: inventory_checked(Order, Status), Status == "available"
% Postcondition: approval_decision(Order, Decision)

execute_approval(Order) :-
    approval_tier(Order, "auto"),
    assert(approval_decision(Order, "approved")),
    call_api(update_order_status, [order: Order, status: "approved"]).

execute_approval(Order) :-
    approval_tier(Order, Tier), Tier \= "auto",
    find_approver(Order, Approver),
    call_api(send_approval_request, [order: Order, approver: Approver]),
    escalate_to_human(Approver, Order, "Requires manual approval").

find_approver(Order, Approver) :-
    can_approve(Approver, Order).
```

4. Logic engine executes: resolves `approval_tier`, finds eligible approver, calls API or escalates.

**Output:** An auditable trace showing which rule path was taken, which approver was selected and why, and the resulting order status — all verifiable against the knowledge graph.

### Example 2: Employee Onboarding Automation

**User:** "Create an onboarding system that provisions accounts, assigns equipment, schedules training, and notifies the team — with dependencies between steps."

**Approach:**

1. Model the task network with dependencies:

```json
{
  "initiative": "employee_onboarding",
  "tasks": [
    {
      "id": "provision_accounts",
      "description": "Create email, Slack, and system accounts for new hire",
      "preconditions": ["employee_record_exists(Employee)"],
      "postconditions": ["accounts_provisioned(Employee)"],
      "tools": ["create_email_account", "create_slack_account", "create_system_account"]
    },
    {
      "id": "assign_equipment",
      "description": "Assign laptop and peripherals based on role",
      "preconditions": ["employee_record_exists(Employee)"],
      "postconditions": ["equipment_assigned(Employee)"],
      "tools": ["check_inventory", "create_equipment_request"]
    },
    {
      "id": "schedule_training",
      "description": "Enroll in required training based on department and role",
      "preconditions": ["accounts_provisioned(Employee)"],
      "postconditions": ["training_scheduled(Employee)"],
      "tools": ["get_required_courses", "enroll_in_course"]
    },
    {
      "id": "notify_team",
      "description": "Send welcome announcement to the team",
      "preconditions": ["accounts_provisioned(Employee)", "equipment_assigned(Employee)"],
      "postconditions": ["team_notified(Employee)"],
      "tools": ["send_slack_message", "send_email"]
    }
  ]
}
```

2. Knowledge graph facts for a specific onboarding:

```prolog
employee("E042", name: "Dana", role: "engineer", department: "platform").
department_training("platform", ["security_101", "platform_onboarding", "ci_cd_basics"]).
role_equipment("engineer", ["laptop_pro", "monitor_27", "keyboard_mech"]).
team_channel("platform", "#platform-team").
```

3. Agent generates logic program for `schedule_training`:

```prolog
execute_schedule_training(Employee) :-
    employee(Employee, department: Dept),
    department_training(Dept, Courses),
    accounts_provisioned(Employee),
    foreach(Course, Courses,
        call_api(enroll_in_course, [employee: Employee, course: Course])),
    assert(training_scheduled(Employee)).
```

4. The engine runs `provision_accounts` and `assign_equipment` in parallel (no dependency between them), then `schedule_training` (needs accounts), then `notify_team` (needs both accounts and equipment).

### Example 3: Compliance-Checked Data Access Request

**User:** "Build a data access request system where requests are checked against data classification policies, the requester's clearance level, and regulatory constraints before granting access."

**Approach:**

1. Define policy rules in the knowledge graph:

```prolog
data_classification("customer_pii", level: "confidential", regulations: ["gdpr", "ccpa"]).
data_classification("sales_metrics", level: "internal", regulations: []).
clearance("analyst", max_level: "internal").
clearance("data_engineer", max_level: "confidential").
regulation_requires("gdpr", "data_processing_agreement").

access_allowed(Requester, Dataset) :-
    employee(Requester, role: Role),
    clearance(Role, max_level: MaxLevel),
    data_classification(Dataset, level: Level),
    level_rank(Level, LR), level_rank(MaxLevel, MR), LR =< MR,
    regulation_check(Requester, Dataset).

regulation_check(Requester, Dataset) :-
    data_classification(Dataset, regulations: Regs),
    foreach(Reg, Regs,
        regulation_satisfied(Requester, Reg)).
```

2. The AI agent generates a logic program that checks `access_allowed`, and if any regulation check fails, escalates to a compliance officer with a structured explanation of exactly which rule blocked access.

**Output:** Either a granted-access action with full audit trail, or a denial with the specific predicate chain that failed — e.g., "Access denied: dataset `customer_pii` requires GDPR compliance; requester `E015` (role: analyst) lacks data_processing_agreement."

## Best Practices

- **Do:** Keep logic facts synchronized with the source-of-truth systems. The knowledge graph must reflect current enterprise state, not a stale export. Use event-driven updates or periodic sync with conflict detection.
- **Do:** Constrain the AI agent's output format strictly. Provide a grammar or schema for the logic program — predicates must follow declared signatures, actions must reference cataloged tools. This prevents the LLM from hallucinating non-existent APIs or malformed rules.
- **Do:** Make escalation conditions explicit in every task. Define exactly when a human must be consulted (ambiguous inputs, constraint violations, high-value decisions). Never allow silent failures.
- **Do:** Log complete execution traces. Every rule evaluation, variable binding, tool invocation, and constraint check should be recorded. This is essential for debugging, compliance auditing, and improving the system over time.
- **Avoid:** Letting the AI agent execute arbitrary code. The agent generates logic programs; the logic engine executes them. This separation is what provides safety guarantees. Never short-circuit by running LLM-generated code directly.
- **Avoid:** Modeling everything as a single monolithic logic program. Keep task-specific programs small and focused. The task network handles orchestration; individual programs handle single-task logic.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Agent generates invalid logic program | Validation step catches undefined predicates, type mismatches, or unsatisfiable constraints | Regenerate with error feedback appended to the agent prompt; limit to 3 retries |
| Precondition not met at execution time | Logic engine evaluates precondition predicates before running actions | Block task, log the unmet condition, check if upstream tasks need re-execution |
| API/tool call fails | Tool wrapper returns error status to logic engine | Apply retry policy from tool catalog; if exhausted, mark task as failed and escalate |
| Knowledge graph stale or inconsistent | Postcondition evaluation fails despite successful actions | Trigger knowledge graph refresh, re-evaluate; flag data integrity issue |
| Constraint violation mid-execution | Logic engine halts on violated constraint rule | Roll back any reversible actions, escalate to human with the specific constraint that failed |
| Human escalation timeout | Timer on escalation request | Re-notify, escalate to next level, or apply default policy if one is defined |

## Limitations

- **Logic programming expertise required for setup.** Defining the foundational rules and knowledge graph schema requires someone who understands predicate logic. The LLM generates task-specific programs, but the base ontology is human-authored.
- **Not suited for purely creative or exploratory tasks.** The architecture assumes tasks have definable preconditions, postconditions, and success criteria. Open-ended processes like strategic brainstorming don't fit the model.
- **LLM-generated logic programs need validation.** The generated programs can contain subtle logical errors (e.g., incorrect variable scoping, unintended rule interactions). The validation step is mandatory, not optional.
- **Scale depends on the logic engine.** Complex knowledge graphs with thousands of rules and millions of facts may require specialized engines (e.g., Datalog with incremental evaluation) rather than naive Prolog interpreters.
- **Human-in-the-loop latency.** Tasks requiring human approval create bottlenecks. Design the task network so that independent tasks can proceed in parallel while human-dependent tasks wait.

## Reference

**Paper:** Pang, C. & Sayama, H. (2026). "Autonomous Business System via Neuro-symbolic AI." Accepted to IEEE SysCon 2026. [arXiv:2601.15599](https://arxiv.org/abs/2601.15599)

**What to look for:** The architecture diagram showing the three-layer interaction (knowledge graph, AI agents, logic engine), the anatomy of agent-generated logic programs (Section on logic program structure), and the business initiative lifecycle showing how tasks flow through the system with human checkpoints.