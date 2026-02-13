---
name: "agentskiller-scaling-generalist-agent"
description: "Synthesize multi-turn agent interaction data across semantically linked domains using DAG-based pipelines, domain ontologies, Person-Centric Entity Graphs, Service Blueprints for MCP servers, and cross-domain fusion. Use when: 'generate training data for tool-calling agents', 'build multi-domain MCP servers', 'create synthetic agent environments', 'design cross-domain service fusion', 'scaffold domain ontology for agent tasks', 'synthesize execution-validated agent trajectories'."
---

This skill enables Claude to apply the AgentSkiller framework for synthesizing high-quality, multi-turn agent interaction data across realistic, semantically linked domains. Rather than relying on scarce API logs or scripted interactions, AgentSkiller uses a fully automated 18-step DAG pipeline to build domain ontologies, generate tool interfaces as MCP servers, populate consistent databases with strict policies, fuse services across domains, and produce execution-validated agent trajectories. Apply this when designing agent training data, building multi-domain tool environments, or constructing MCP server ecosystems that need consistent state management and cross-service reasoning.

## When to Use

- When the user needs to generate synthetic multi-turn interaction data for training tool-calling LLM agents
- When building a suite of MCP servers that must share consistent entities and enforce domain-specific policies
- When designing a domain ontology that links multiple service areas (e.g., healthcare + scheduling + billing)
- When creating execution-validated task trajectories where every tool call is verified against database state
- When the user wants to fuse two or more single-domain services into cross-domain tasks with shared entities
- When scaffolding realistic mock environments with databases, policies, and tool interfaces for agent evaluation
- When the user asks to "scale agent training data" or "create diverse tool-calling datasets"

## Key Technique

AgentSkiller addresses the data bottleneck for generalist LLM agents by replacing manual data collection with a fully automated synthesis pipeline. The core insight is that realistic agent data requires **semantic grounding**: tools must operate on consistent databases, policies must constrain behavior deterministically, and cross-domain tasks must be linked through shared entities rather than arbitrarily stitched together. The framework achieves this through a DAG of 18 atomic steps organized into 5 phases, each producing artifacts that downstream steps consume.

The pipeline starts by expanding a small set of seed domains (e.g., 15) into hundreds (e.g., 300) using five expansion dimensions: horizontal association, diagonal association, temporal derivation, hierarchical refinement, and agent-centric mapping. For each domain, it builds a **Person-Centric Entity Graph** where actor nodes (Student, Clinician) connect to resource nodes (Course, Appointment) via directed edges. **Service Blueprints** then define the relational schemas and tool interfaces for each actor-resource subgraph, enforcing a star-schema constraint (the core entity appears as a foreign key in all relationships). These blueprints are compiled into functional MCP servers with unit tests and a self-correction loop (up to 5 iterations of pytest + automated patching).

The differentiating mechanism is **cross-domain fusion**: two single-domain trajectories are linked through shared core entities (e.g., a Patient entity bridges a hospital booking service and an insurance claims service) with a natural-language Motivation Descriptor explaining why the cross-domain task exists. Policies from each domain are harmonized via bridge rules (e.g., "Payment must initiate only after Booking reaches 'confirmed'"). Tasks are then instantiated with concrete database values, validated via dual-path execution (LLM-generated scripts for novel trajectories, cached scripts for known templates), and filtered with an all-or-nothing scoring criterion where failure in any dimension means the entire sample is discarded. A Persona-based Simulator generates realistic user queries by applying an information-hiding principle: the initial query contains only a single pain point, forcing the agent to actively gather missing parameters.

## Step-by-Step Workflow

1. **Define seed domains and expand the ontology.** Start with 10-20 seed domains relevant to the target application. Expand each using five dimensions: horizontal (parallel sectors like `hotel_booking` -> `car_rental`), diagonal (cross-context like `hospital` -> `medical_billing`), temporal (lifecycle like `order_placement` -> `order_tracking`), hierarchical (specialization like `healthcare` -> `dental_services`), and agent-centric (role-based like `hospital` -> `nurse_scheduling`). Use snake_case, 1-3 terms, and deduplicate against a global set. Target 200-300 domains.

2. **Extract domain-invariant entities for each domain.** For each domain, identify entities that pass three tests: (a) existential independence (the entity exists on its own, not as a relationship), (b) semantic stability (meaning is consistent across contexts), and (c) mid-level granularity (`Student` is acceptable, `MiddleSchoolStudent` is too specific). Classify each as an actor (is_person=true) or a resource.

3. **Build the Person-Centric Entity Graph.** Construct a directed graph G=(V,L) where actor nodes connect to resource nodes via semantic edges inferred by zero-shot LLM reasoning. Use hub-and-spoke topology around each actor. Store as NetworkX-compatible JSON. Batch edge inference in groups of 10 with parallelized threads and atomic progress logging.

4. **Generate Service Blueprints from the entity graph.** For each actor node, sample k distinct subgraphs of its neighbors (1-N resource entities). Each blueprint B = (C, P, R, F) defines: Core entity C (the actor), Peripheral entities P (resources), Relational schemas R (star-schema with C as FK in all tables), and Executable functions F (tool signatures). Validate that no relationship-to-relationship references exist. Normalize all foreign keys to the format `{entity_name}_id`.

5. **Implement MCP servers with test-driven self-correction.** For each Service Blueprint, generate a Python MCP server using a skeletal template. Include: a unified invocation router, session management, a `confirm` parameter on all state-mutating operations, and policy assertion hooks. Generate a pytest suite alongside the implementation. Run the test loop: execute tests -> diagnose failures (implementation bug / test defect / data issue) -> generate search-and-replace patches -> re-run, up to K=5 iterations.

6. **Populate databases and generate Domain Policies.** For each schema in the blueprint, generate synthetic data using Faker with domain-appropriate providers. Define policies as computational contracts with four dimensions: preconditions (required system state), input validation (parameter format constraints), permission outcomes (decision tree mapping state to access), and side effects (exact state mutations). Enforce the closed-world constraint: policies may only reference attributes that exist in the schema. Run a Policy Auditor to detect hallucinated attributes and auto-patch.

7. **Model semantic tool dependencies as a DAG.** Analyze each MCP server's tool set and build a tool dependency graph. Enforce two constraints: reachability (all tools reachable from a source node) and acyclicity (no circular dependencies). This graph determines valid execution orderings for trajectory generation.

8. **Fuse domains via shared entities and bridge rules.** Identify pairs of domains that share a core entity type (e.g., Patient in both `hospital_booking` and `insurance_claims`). For each pair, define: trajectory A from domain A, trajectory B from domain B, and a Motivation Descriptor M explaining the cross-domain need. Unify databases with namespace isolation (prefix relationship names as `{server}_{table}`). Harmonize policies by synthesizing bridge rules that govern inter-domain sequencing.

9. **Synthesize and validate task templates.** Enumerate trajectory templates from the tool dependency DAG. For each template, ground it with concrete database values, then validate via dual-path execution: use an LLM to generate a Python execution script for novel trajectories, or use cached scripts for known patterns. Apply all-or-nothing scoring: if any tool call fails policy checks, entity validation, or state consistency checks, discard the entire sample. Prune redundant read-only operations from validated trajectories.

10. **Generate multi-turn dialogues with the Persona Simulator.** For each validated task, create three artifacts: a grounded instruction (fully instantiated), a motivation context (plausible backstory), and a startup query (single pain point, information-hidden). Run the interactive loop: user simulator sends startup query, agent responds with tool calls, MCP servers execute against the mock environment, continue until goal satisfaction or termination. Export as JSON Lines with user prompt, hidden system prompt, and ground-truth trajectory hash.

## Concrete Examples

**Example 1: Building a Healthcare + Billing Cross-Domain Environment**

User: "I need to generate training data for an agent that handles hospital appointment booking and insurance claim filing together."

Approach:
1. Define seed domains: `hospital_appointment`, `insurance_claims`
2. Extract entities: Actor=Patient, Resources=Appointment, Doctor, InsurancePlan, Claim
3. Build entity graph with Patient as hub connecting to all resources
4. Generate Service Blueprints:
   - Blueprint A: Patient manages Appointments with Doctor (tools: `book_appointment`, `cancel_appointment`, `get_available_slots`)
   - Blueprint B: Patient manages Claims against InsurancePlan (tools: `file_claim`, `check_claim_status`, `get_coverage_details`)
5. Implement two MCP servers with pytest validation
6. Fuse via shared Patient entity; bridge rule: "Claim can only be filed after Appointment status='completed'"
7. Generate task: "Patient needs to book a cardiology appointment and file an insurance claim for the visit"

Output structure:
```json
{
  "domains": ["hospital_appointment", "insurance_claims"],
  "shared_entity": "Patient",
  "bridge_rules": [
    "file_claim requires appointment.status == 'completed'",
    "claim.provider_id must match appointment.doctor.provider_network_id"
  ],
  "trajectory": [
    {"tool": "get_available_slots", "server": "hospital_appointment", "params": {"specialty": "cardiology"}},
    {"tool": "book_appointment", "server": "hospital_appointment", "params": {"slot_id": "S-1042", "patient_id": "P-7721", "confirm": true}},
    {"tool": "get_coverage_details", "server": "insurance_claims", "params": {"patient_id": "P-7721", "procedure_code": "CARD-001"}},
    {"tool": "file_claim", "server": "insurance_claims", "params": {"patient_id": "P-7721", "appointment_id": "A-3301", "confirm": true}}
  ],
  "startup_query": "I just had a cardiology checkup and need to get this covered by my insurance. Can you help?"
}
```

**Example 2: Scaffolding an E-Commerce MCP Server from a Service Blueprint**

User: "Create an MCP server for a retail catalog and checkout service with proper policies."

Approach:
1. Define blueprint: Core=Customer, Peripherals=[Product, Cart, Order]
2. Design star-schema: Cart table has `customer_id` FK, Order table has `customer_id` FK, CartItem links Cart and Product
3. Define tools: `search_products`, `add_to_cart`, `remove_from_cart`, `checkout`, `get_order_status`
4. State-mutating tools (`add_to_cart`, `remove_from_cart`, `checkout`) get `confirm` parameter
5. Generate policies:
   - `checkout` precondition: cart must contain at least 1 item
   - `checkout` input validation: payment_method must be in ['credit_card', 'debit_card', 'wallet']
   - `checkout` side effects: creates Order, clears Cart, decrements Product.stock
6. Implement server, run pytest loop

Output (Service Blueprint):
```yaml
blueprint:
  core_entity: Customer
  peripheral_entities: [Product, Cart, Order, CartItem]
  schemas:
    Customer: {customer_id: uuid, name: str, email: str}
    Product: {product_id: uuid, name: str, price: float, stock: int}
    Cart: {cart_id: uuid, customer_id: uuid, created_at: datetime}
    CartItem: {item_id: uuid, cart_id: uuid, product_id: uuid, quantity: int}
    Order: {order_id: uuid, customer_id: uuid, total: float, status: str}
  tools:
    - name: search_products
      params: {query: str, category: str|null}
      mutates_state: false
    - name: add_to_cart
      params: {customer_id: uuid, product_id: uuid, quantity: int, confirm: bool}
      mutates_state: true
      precondition: "Product.stock >= quantity"
    - name: checkout
      params: {customer_id: uuid, payment_method: str, confirm: bool}
      mutates_state: true
      precondition: "Cart must contain >= 1 item"
      side_effects: ["Create Order", "Clear Cart", "Decrement Product.stock"]
```

**Example 3: Generating a Persona-Driven Multi-Turn Dialogue**

User: "Generate a realistic multi-turn conversation where an agent helps a user with travel booking."

Approach:
1. Select validated trajectory: `search_flights` -> `book_flight` -> `reserve_hotel` -> `arrange_airport_transfer`
2. Create persona: Business traveler, urgent timeline, specific preferences
3. Apply information hiding: startup query reveals only the immediate need
4. Simulate the multi-turn exchange

Output:
```
Turn 1 - User: "I need to get to Chicago for a meeting next Tuesday."
Turn 2 - Agent: [calls search_flights(destination="ORD", date="2026-02-17")]
Turn 3 - Agent: "I found 3 flights. Do you have a preference for morning or afternoon?"
Turn 4 - User: "Morning, and I'll need a hotel near the Loop for two nights."
Turn 5 - Agent: [calls book_flight(flight_id="UA-441", passenger_id="T-8823", confirm=true)]
Turn 6 - Agent: [calls reserve_hotel(location="Chicago Loop", check_in="2026-02-17", nights=2)]
Turn 7 - Agent: "Booked UA-441 departing 7:15 AM and reserved the Palmer House. Want airport transfer?"
Turn 8 - User: "Yes please, arrange a car service."
Turn 9 - Agent: [calls arrange_airport_transfer(booking_id="B-5567", type="car_service", confirm=true)]
Turn 10 - Agent: "Done. Car service confirmed for arrival at ORD. Confirmation sent to your email."
```

## Best Practices

- **Do:** Enforce the star-schema constraint in every Service Blueprint. The core actor entity must appear as a foreign key in all relationship tables. This ensures consistent entity resolution across tool calls.
- **Do:** Apply the closed-world constraint when generating policies. Policies must only reference attributes that exist in the actual database schema. Run a Policy Auditor pass to catch hallucinated fields before proceeding.
- **Do:** Use namespace isolation (`{server}_{table}`) when merging databases across domains. Without this, identically named tables from different services will collide.
- **Do:** Validate every trajectory via execution against the actual mock database before including it in training data. Use all-or-nothing scoring: one failed tool call invalidates the entire sample.
- **Avoid:** Generating cross-domain fusions without a natural Motivation Descriptor. Arbitrary domain pairings produce incoherent tasks. Always justify why two domains interact through a shared entity and a plausible user need.
- **Avoid:** Including redundant read-only operations in finalized trajectories. Prune tool calls whose results are strict subsets of information already retrieved. This keeps training signal clean and focused on state-changing actions.

## Error Handling

- **Schema validation failures:** If a Service Blueprint references a relationship between two peripheral entities without routing through the core entity, reject it and regenerate with the star-schema constraint explicitly restated.
- **MCP server test failures:** Classify the root cause as implementation bug, test defect, or data issue. Apply search-and-replace patches and retry up to 5 times. If the loop exhausts retries, flag for manual review.
- **Policy hallucination:** When a Domain Policy references a database column that does not exist, the Policy Auditor should auto-generate a patch. If the hallucinated attribute is semantically necessary, add it to the schema instead.
- **Cross-domain fusion conflicts:** When two domains have contradictory policies (e.g., one requires immediate payment, the other allows deferred), synthesize a bridge rule that establishes sequencing. If irreconcilable, split into two separate single-domain tasks.
- **Trajectory validation failures:** When execution-based validation rejects a sample, check the error class. Policy violations suggest the task template is invalid. Invalid entity data suggests the database population step needs domain-specific Faker providers. Server bugs route back to the self-correction loop.
- **Low yield rate:** If more than 40% of generated samples fail validation, the domain ontology or entity graph likely has structural issues. Re-examine entity independence and granularity constraints before re-running downstream steps.

## Limitations

- The framework assumes domains can be meaningfully linked through shared entity types. Highly specialized or isolated domains (e.g., quantum computing simulation) may not fuse naturally with common service domains.
- Cross-domain fusion scales quadratically with the number of domains. For large ontologies (300+ domains), selective pairing based on entity overlap is necessary to keep computation tractable.
- Generated MCP servers are mock implementations backed by in-memory or file-based databases. They do not replicate the latency, failure modes, or concurrency behavior of production APIs.
- The all-or-nothing validation criterion is intentionally strict, yielding roughly 68% retention. This is a quality-over-quantity tradeoff; loosening it risks introducing subtly incorrect training signal.
- Persona-based simulation with information hiding produces realistic queries but cannot fully capture the unpredictability of real users who may change goals mid-conversation or provide contradictory information.
- The approach works best for structured, tool-calling agent tasks. It is less applicable to open-ended reasoning, creative generation, or tasks that do not involve discrete state transitions.

## Reference

[AgentSkiller: Scaling Generalist Agent Intelligence through Semantically Integrated Cross-Domain Data Synthesis](https://arxiv.org/abs/2602.09372v1) — Sun et al., 2026. Focus on Section 3 (the 18-step DAG pipeline), Section 4 (cross-domain fusion and policy harmonization), and Section 5 (execution-based validation with all-or-nothing scoring) for implementation details.