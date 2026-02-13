---
name: "world-workflows-benchmark-bringing"
description: "Build world models for enterprise systems with hidden workflows and cascading database effects. Applies the probe-observe-model pattern from the World of Workflows paper to safely operate in interconnected databases where actions trigger invisible side effects. Use when: 'help me understand what side effects this DB change will cause', 'build a world model of our workflow system', 'audit what cascading changes happen when I update this record', 'predict hidden effects of this enterprise action', 'detect constraint violations in our workflow', 'map out hidden triggers in our database system'."
---

# World-Model-Driven Enterprise Workflow Reasoning

This skill enables Claude to act as a world-model-aware agent when operating within enterprise systems that have hidden workflows, business rules, and cascading database side effects. Rather than blindly executing CRUD operations and trusting direct API responses, Claude builds an internal model of how actions propagate through interconnected tables, predicts indirect state changes before they happen, and detects silent constraint violations that surface-level observation would miss. The core technique comes from the World of Workflows (WoW) benchmark, which demonstrated that frontier LLMs suffer from "dynamics blindness" -- they consistently fail to predict invisible cascading effects -- and that grounded world modeling closes this gap.

## When to Use

- When the user asks to modify records in an enterprise system (ServiceNow, Salesforce, SAP, custom ERP) and needs to understand downstream effects before committing
- When building or debugging database triggers, business rules, or workflow automations that create cascading updates across tables
- When the user wants to audit what hidden side effects a particular action (e.g., assigning an asset, closing an incident, onboarding an employee) will produce
- When designing agents that interact with enterprise APIs and need to anticipate state changes beyond the direct API response
- When debugging unexpected data mutations and tracing them back to hidden workflow triggers
- When the user asks to validate whether a planned sequence of actions will violate any system constraints, especially constraints enforced by invisible business rules
- When writing integration tests for enterprise workflows that must assert on cascading state changes, not just primary outcomes

## Key Technique: Grounded World Modeling for Opaque Systems

Enterprise systems like ServiceNow contain thousands of business rules and workflows that execute invisibly when records change. A single `UPDATE` on one table can trigger a chain of modifications across dozens of related tables. The WoW paper formalizes this as a Partially Observable Markov Decision Process (POMDP): the agent sees direct tool responses but not the full state transition function. Three specific gaps cause agent failure:

**Representation Gap (73.5% of errors):** LLMs confuse human-readable names with database identifiers. An agent predicts `username: "john.doe"` when the system actually stores `sys_id: "a8f3b2c1..."`. The fix: always resolve entities to their canonical database identifiers before reasoning about state.

**Dynamics Gap:** LLMs under-predict the set of affected tables. Creating an incident silently updates `metric_instance`, `sys_audit`, and notification tables. The fix: build an explicit transition model by probing the system -- execute an action, diff the database state before and after, record every table/column that changed.

**Causal Gap:** LLMs plan greedily without multi-step projection. Assigning a 4th asset to a user triggers a clearance decrement, which makes a subsequent action impossible. The fix: simulate state forward multiple steps (causal rollout) before executing, checking for downstream constraint violations.

The practical technique is **probe-observe-model**: issue controlled test actions, capture the full state diff (not just the API response), build a map of trigger conditions and their cascading effects, then use that map to predict and validate future operations.

## Step-by-Step Workflow

1. **Inventory the schema and relationships.** Query the database schema to enumerate all tables, their columns, foreign key relationships, and any documented triggers or business rules. Produce a dependency graph showing which tables reference which others.

2. **Identify the action under analysis.** Clarify exactly which operation the user wants to perform (e.g., "update the `state` column of the `incident` table from `In Progress` to `Resolved`"). Pin down the table, column, old value, and new value.

3. **Resolve entities to canonical identifiers.** Replace all human-readable names with database-native identifiers (sys_id, primary keys, UUIDs). This eliminates the representation gap. Query the database to confirm the mapping: `SELECT sys_id FROM user WHERE user_name = 'john.doe'`.

4. **Capture baseline state snapshot.** Before any action, query all potentially affected tables and record their current state. Include tables directly referenced by foreign keys AND tables known to be updated by workflows. Store as structured tuples: `(TableName, ColumnName, CurrentValue)`.

5. **Execute a probe action in a safe environment.** If a staging/dev environment exists, perform the action there. If not, use read-only queries and workflow documentation to simulate. The goal is to observe every state change, not just the direct one.

6. **Diff the state to extract the full transition.** Compare post-action state against the baseline. Record every change as a delta tuple: `(TableName, ColumnName, OldValue, NewValue)`. This is the ground truth transition function for this action. Pay special attention to tables the user did NOT expect to change.

7. **Build the causal chain.** From the state diff, reconstruct the cascade: "Action A modified table X, which triggered rule R1, which modified table Y, which triggered rule R2, which modified table Z." Document each link with the trigger condition (e.g., "when asset_count > 3, decrement clearance_level").

8. **Perform causal rollout for multi-step plans.** If the user plans a sequence of actions, simulate each step's full state diff in order. After each simulated step, check whether the resulting state satisfies preconditions for the next step. Flag any step where a cascading effect invalidates a later action.

9. **Check constraints against the projected final state.** Enumerate known constraints (role exclusions, field validations, referential integrity, business policies). Verify the projected end state satisfies all of them. Report any silent violations -- cases where the action "succeeds" but leaves the system in an invalid state.

10. **Report findings as a structured audit.** Present: (a) direct effects, (b) cascading effects with causal chains, (c) constraint violations detected, and (d) recommendations for safe execution order or prerequisite actions.

## Concrete Examples

**Example 1: Predicting side effects of an incident state change**

User: "What happens in ServiceNow when I resolve an incident that has child incidents?"

Approach:
1. Identify the action: `UPDATE incident SET state = 'Resolved' WHERE number = 'INC0010042'`
2. Resolve the incident's sys_id: query `incident` table for `INC0010042`
3. Snapshot related tables: `incident`, `task_sla`, `sys_audit`, `metric_instance`, `incident` (child records via `parent_incident`)
4. Trace known business rules for incident resolution:
   - Business rule "Close child incidents" triggers: all child incidents where `parent_incident = <sys_id>` get `state = 'Resolved'`
   - Each child resolution triggers its own SLA stop clock on `task_sla`
   - `metric_instance` records resolution time metric for parent AND each child
   - `sys_audit` receives one entry per field change per record
5. Causal rollout: if any child incident has `state = 'On Hold'`, the "Close child incidents" rule may conflict with the "Hold prevents closure" constraint

Output:
```
DIRECT EFFECTS:
  incident (INC0010042): state: In Progress -> Resolved
                         resolved_at: NULL -> 2026-01-29T14:30:00Z
                         resolved_by: NULL -> sys_id_of_current_user

CASCADING EFFECTS:
  incident (INC0010043, child): state: New -> Resolved  [via business rule: "close-child-incidents"]
  incident (INC0010044, child): state: New -> Resolved  [via business rule: "close-child-incidents"]
  task_sla (3 records): has_breached recalculated, stop_time set  [via workflow: "sla-stop-on-resolve"]
  metric_instance (3 records): new rows created with resolution_time  [via business rule: "calculate-resolution-metric"]
  sys_audit (9 entries): one per field change across all affected records

CONSTRAINT CHECK:
  WARNING: INC0010044 has state = 'On Hold'. Business rule "hold-prevents-closure"
  may block cascading resolution. Verify rule priority order or resolve the hold first.
```

**Example 2: Safe employee offboarding sequence**

User: "I need to offboard employee E1234 -- deactivate their account, unassign their assets, and remove role memberships. What order is safe?"

Approach:
1. Resolve E1234 to sys_id in `sys_user` table
2. Map the dependency graph:
   - `sys_user_has_role` references `sys_user.sys_id`
   - `alm_asset` has `assigned_to` referencing `sys_user.sys_id`
   - `sys_user.active` field triggers workflow "on-deactivation"
3. Probe the "on-deactivation" workflow behavior:
   - Automatically revokes VPN certificates (updates `vpn_certificate` table)
   - Sends notification to manager (creates `sys_email` record)
   - But does NOT unassign assets or remove roles
4. Check for hidden constraints:
   - Business rule: "Cannot unassign asset from inactive user" (requires active = true)
   - Business rule: "Role removal triggers access audit" (creates `access_audit` record)
5. Determine safe ordering via causal rollout

Output:
```
RECOMMENDED EXECUTION ORDER:
  1. Unassign assets (UPDATE alm_asset SET assigned_to = NULL WHERE assigned_to = <sys_id>)
     Reason: Must happen BEFORE deactivation. Hidden business rule blocks
     asset unassignment for inactive users.

  2. Remove role memberships (DELETE FROM sys_user_has_role WHERE user = <sys_id>)
     Cascading: Creates access_audit entries per role removed.
     Safe at this point: user is still active.

  3. Deactivate account (UPDATE sys_user SET active = false WHERE sys_id = <sys_id>)
     Cascading: Revokes VPN certs, sends manager notification.
     Safe last: no remaining assets or roles to conflict with.

UNSAFE ORDER WARNING:
  Deactivating first would make step 1 fail silently -- the asset remains
  assigned but the business rule swallows the error, leaving orphaned assets
  with no indication of failure in the API response.
```

**Example 3: Diagnosing an unexpected database mutation**

User: "A record in `metric_instance` appeared that nobody created. How do I trace where it came from?"

Approach:
1. Query the `metric_instance` record for its `sys_created_on` timestamp and `sys_created_by` (often `system` for workflow-generated records)
2. Query `sys_audit` for all changes within a 5-second window around that timestamp
3. Reconstruct the causal chain by working backwards:
   - `metric_instance` created at T+2s by `system`
   - `incident` state changed at T+0s by user `admin`
   - Business rule `calculate-resolution-metric` fires on incident state change to `Resolved`
4. Confirm by checking business rule conditions against the incident's state transition

Output:
```
ROOT CAUSE TRACE:
  T+0.0s  incident.state: In Progress -> Resolved  (by: admin, via: UI)
  T+0.1s  business rule "calculate-resolution-metric" fires
          condition: current.state == 'Resolved' && previous.state != 'Resolved'
  T+0.3s  metric_instance record created (by: system)
          fields: table=incident, id=<incident_sys_id>, metric_type=resolution_time

CONCLUSION: The metric_instance record was created by the business rule
"calculate-resolution-metric" which fires automatically when any incident
transitions to Resolved state. This is expected system behavior, not a bug.
```

## Best Practices

- **Do:** Always snapshot state BEFORE and AFTER any action. Compare the full diff, not just the fields you expect to change. Enterprise systems routinely modify 3-10x more fields than the direct action touches.
- **Do:** Resolve every entity to its canonical database identifier (sys_id, UUID, primary key) before reasoning. Human-readable names cause 73.5% of prediction errors according to the WoW benchmark.
- **Do:** Build and maintain a trigger map -- a lookup from (table, column, condition) to (affected_tables, changes). Populate it incrementally through probe actions. This is your world model.
- **Do:** Simulate multi-step plans forward before executing. Check each intermediate state for constraint satisfaction. A greedy step-by-step approach misses cases where step N makes step N+2 impossible.
- **Avoid:** Trusting API success responses as proof that no side effects occurred. Enterprise APIs return 200 OK while hidden workflows silently violate constraints in the background.
- **Avoid:** Assuming you know all business rules from documentation alone. Documentation is perpetually incomplete in enterprise systems. Always verify with probe-observe-diff when possible.
- **Avoid:** Treating each database table in isolation. The entire point of enterprise workflows is cross-table automation. Always ask: "What OTHER tables does this change affect?"

## Error Handling

- **Silent constraint violations:** The action completes successfully but leaves the system in an invalid state. Detection: compare the final state against all known constraints. Mitigation: pre-validate with causal rollout before execution.
- **Representation mismatches:** Your predicted state uses display values (`"John Doe"`) while the database stores internal IDs (`"a8f3b2c1"`). Detection: IoU between predicted and actual state diff drops to near zero. Mitigation: always resolve to canonical identifiers.
- **Incomplete cascade prediction:** You predicted 3 table changes but 7 actually occurred. Detection: state diff shows unexpected tables. Mitigation: expand your probe scope -- query more related tables in the baseline snapshot. Update your trigger map.
- **Temporal ordering failures:** A workflow fires asynchronously and your post-action snapshot was too early to capture it. Detection: re-query after a delay shows additional changes. Mitigation: implement a polling/retry pattern for state diff capture with configurable delay.
- **Permission-limited observability:** You cannot query certain tables due to access controls. Detection: permission errors on baseline snapshot queries. Mitigation: document the observability boundary explicitly and warn the user that cascade prediction is incomplete for those tables.

## Limitations

- This approach requires either a staging environment for probing or sufficient documentation/audit logs to reconstruct cascading behavior. In pure production with no safe probe capability, predictions rely entirely on documentation quality.
- The probe-observe-model pattern builds knowledge incrementally. On first encounter with a new enterprise system, coverage will be sparse. The world model improves with each probed action.
- Asynchronous workflows with long delays (minutes or hours) are difficult to capture in a single probe-diff cycle. Some enterprise systems batch workflow execution on schedules.
- Systems with thousands of business rules (the WoW benchmark uses 4,800+) may have rule priority conflicts that produce non-deterministic outcomes depending on execution order.
- This technique addresses hidden *automation* side effects. It does not address concurrent user modifications, race conditions, or distributed system consistency issues.

## Reference

**Paper:** [World of Workflows: A Benchmark for Bringing World Models to Enterprise Systems](https://arxiv.org/abs/2601.22130v2) -- Gupta et al., 2026. Look for: the POMDP formalization of enterprise agent interaction (Section 3), the three gap analysis (representation, dynamics, causal) in Section 6, the probe-observe-model paradigm in the discussion, and the evaluation metrics (TSRUC, Audit IoU) that define what "correct" enterprise operation looks like.