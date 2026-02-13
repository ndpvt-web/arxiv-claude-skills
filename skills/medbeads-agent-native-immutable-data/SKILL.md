---
name: "medbeads-agent-native-immutable-data"
description: "Build immutable, agent-native medical data pipelines using Merkle DAG structures (MedBeads pattern). Converts mutable EMR/FHIR records into cryptographically-linked, causally-ordered bead graphs that LLMs can traverse deterministically instead of relying on probabilistic RAG. Use when: 'build an immutable clinical data store', 'convert FHIR to a causal DAG', 'create tamper-evident medical records', 'agent-native healthcare data pipeline', 'deterministic context retrieval for medical AI', 'MedBeads implementation'."
---

# MedBeads: Agent-Native, Immutable Data Substrate for Medical AI

This skill teaches Claude to implement the MedBeads architecture -- an immutable, causally-linked data substrate that replaces probabilistic RAG retrieval with deterministic graph traversal for clinical AI agents. The core idea: instead of storing medical data in flat, mutable FHIR resources that force LLMs to guess context via vector search, encode every clinical event as a cryptographically-sealed "Bead" in a Merkle DAG where causal relationships (why something happened) are explicit edges, not inferred metadata. This eliminates hallucination from context reconstruction and makes every AI decision fully auditable.

## When to Use

- When building a medical records system that AI agents will consume directly (not just human clinicians)
- When converting existing FHIR R4 resources into a causally-linked, append-only graph
- When implementing tamper-evident audit trails for clinical data where integrity must be mathematically verifiable
- When replacing RAG-based clinical context retrieval with deterministic BFS graph traversal
- When designing multi-agent clinical workflows where agents need shared, immutable patient context
- When building an MCP server that exposes patient history as a traversable DAG to LLM agents
- When the user needs content-addressable storage for medical records with cryptographic verification

## Key Technique

**The Context Mismatch Problem.** Traditional EMR/FHIR stores data for humans: flat, mutable resources where causality is implicit in timestamps and narrative notes. When an LLM agent needs to understand "why was this medication prescribed?", it must use RAG to retrieve fragments and probabilistically reconstruct the causal chain. This introduces semantic confusion (e.g., confusing "congestive heart failure" with "cardiac arrest" because both match a "heart failure" query) and makes hallucination undetectable.

**The Bead + Merkle DAG Solution.** MedBeads encodes each clinical event as a Bead: a seven-component tuple `B = {T, τ, A, P, C, E, Σ}` containing timestamp, type, author (DID), parent bead hashes, JSON content payload, evidence references, and a digital signature. Each Bead's ID is `SHA-256(CanonicalJSON(B))`, so any mutation breaks the hash chain. Parent links create a Merkle DAG where traversal from any node upward answers "why did this happen?" and downward answers "what happened next?" -- both in O(V+E) time via BFS, with zero probabilistic inference.

**Separation of Concerns.** The architecture guarantees *what* context the LLM receives (deterministic, complete, tamper-evident) while the LLM decides *how* to interpret it. This means you can swap LLM providers without restructuring your data layer. The append-only design means errors are corrected by appending Correction Beads (like Git commits), never by mutating history.

## Step-by-Step Workflow

1. **Define the Bead schema.** Create a JSON structure with seven fields: `id` (SHA-256 hash, computed), `timestamp` (ISO 8601), `type` (one of: `patient_registration`, `fhir_encounter`, `fhir_condition`, `fhir_observation`, `fhir_medicationrequest`, `medical_note`, `reasoning`, `correction`), `author` (DID string like `did:medbeads:doctor:12345`), `parents` (array of parent bead hash IDs), `content` (arbitrary JSON payload), `evidence` (array of `{uri, content_hash}` for external binaries like DICOM), and `signature` (base64 digital signature).

2. **Implement content-addressable storage (CAS).** Store each Bead as a file at `./objects/{hash[0:2]}/{hash[2:]}` (two-character prefix sharding into ~256 subdirectories). Files are write-once, never modified. Verification is recomputing the hash and comparing to the filename.

3. **Build the Bead creation pipeline.** On POST, canonicalize the Bead JSON (sorted keys, no whitespace), compute `SHA-256(canonical)` as the ID, write to CAS, then update the SQLite index with `bead_id`, `type`, `timestamp`, `patient_id`, and populate a `bead_edges` table mapping parent-child relationships. The SQLite index is a rebuildable cache -- include a `ReindexStorage()` function that reconstructs it by scanning the CAS directory.

4. **Map FHIR resources to Beads with explicit causal parents.** Patient becomes a genesis bead (root, no parents). Encounter parents to Patient. Condition parents to Encounter. Observation parents to Encounter. MedicationRequest parents to both Encounter and Condition (dual causality). Map fields: `Encounter.period.start` to `timestamp`, `Condition.code.text` to `content.condition_name`, `Observation.valueQuantity` to `content.value` + `content.unit`.

5. **Implement BFS context retrieval in two directions.** Ancestor traversal (upward through `parents` links) answers "why did this event occur?" Descendant traversal (downward from patient root through `bead_edges`) answers "what happened after?" Accept a `depth` parameter to limit traversal scope. Use `O(1)` lookup on the `bead_edges` index table for parent-child resolution.

6. **Build the REST API with five endpoints.** `POST /beads` creates a bead and returns its hash. `GET /beads?id={hash}` retrieves a single bead. `GET /beads/context?id={hash}&depth=N&direction=ancestors|descendants` returns the causal subgraph. `GET /patients` lists all patient root beads. `GET /patients/{id}/beads` returns all beads for a patient via full descendant traversal.

7. **Serialize the DAG subgraph for LLM prompt injection.** The Python middleware retrieves the relevant subgraph, sorts beads chronologically, serializes them as structured text (not raw JSON -- use a compact, token-efficient format), and injects into the LLM prompt alongside the query. This is the "agent-native" part: the LLM receives a complete, ordered, causally-linked context window.

8. **Embed role-based clearance rules per Bead.** Each Bead can include a `clearance` field with deny-list rules (e.g., `{"deny": ["Insurance", "Family"]}`). Sensitive psychiatric notes deny Insurance access; emergency beads grant Emergency role access. Enforce at the API layer before returning beads.

9. **Implement tamper-evidence verification.** To verify integrity of a patient's entire record: traverse all beads from root, recompute each bead's hash from its content, and confirm it matches the stored ID and all child `parents` references. Any mismatch indicates tampering. This is a pure mathematical operation with no trust dependency.

10. **Dockerize as a three-tier stack.** Go core engine (~800 lines) for CAS, indexing, REST API, and graph traversal. Python FastAPI middleware (~200 lines) for FHIR conversion and LLM prompt serialization. React frontend for timeline and DAG visualization. Deploy via `docker-compose`.

## Concrete Examples

**Example 1: Converting a FHIR Patient Bundle to a MedBeads DAG**

User: "I have Synthea-generated FHIR bundles. Convert them into an immutable bead graph."

Approach:
1. Parse the FHIR Bundle JSON, extracting resources by type (Patient, Encounter, Condition, Observation, MedicationRequest)
2. Create the genesis bead from the Patient resource
3. Process Encounters, parenting each to the Patient bead
4. Process Conditions, parenting each to its referenced Encounter
5. Process Observations and MedicationRequests with appropriate parent links
6. Store each bead in CAS and index in SQLite

Output:
```json
{
  "id": "a3f8c2...sha256",
  "type": "fhir_condition",
  "timestamp": "2025-03-15T09:30:00Z",
  "author": "did:medbeads:synthea:converter",
  "parents": ["b7e1d4...encounter_hash"],
  "content": {
    "condition_name": "Essential Hypertension",
    "icd10": "I10",
    "clinical_status": "active",
    "verification_status": "confirmed"
  },
  "evidence": [],
  "signature": "base64..."
}
```
The resulting DAG: `Patient(root) -> Encounter_1 -> [Condition_HTN, Observation_BP, MedRequest_Lisinopril]`

**Example 2: Deterministic Context Retrieval for a Clinical Agent**

User: "An LLM agent needs to evaluate whether to adjust a patient's medication. Build the context retrieval."

Approach:
1. Start from the MedicationRequest bead for the current prescription
2. Traverse ancestors (BFS upward through `parents`) to depth 5 to get: the prescribing encounter, the condition that prompted it, prior observations, and the patient root
3. Traverse descendants from the condition bead to find: all observations monitoring the condition, any correction beads, and subsequent medication changes
4. Serialize the subgraph chronologically into a structured prompt block

Output (injected into LLM prompt):
```
=== CLINICAL CONTEXT (7 beads, deterministic retrieval) ===
[2024-01-10] patient_registration: Jane Doe, DOB 1965-03-22
  hash: a1b2c3... | root
[2024-06-15] fhir_encounter: Annual checkup, Dr. Smith
  hash: d4e5f6... | parent: a1b2c3
[2024-06-15] fhir_observation: BP 158/95 mmHg
  hash: g7h8i9... | parent: d4e5f6
[2024-06-15] fhir_condition: Essential Hypertension (I10)
  hash: j0k1l2... | parent: d4e5f6
[2024-06-15] fhir_medicationrequest: Lisinopril 10mg daily
  hash: m3n4o5... | parents: d4e5f6, j0k1l2
[2024-12-01] fhir_observation: BP 142/88 mmHg
  hash: p6q7r8... | parent: d4e5f6
[2025-01-15] fhir_encounter: Follow-up, Dr. Smith
  hash: s9t0u1... | parent: a1b2c3
=== END CONTEXT ===

Query: Should Lisinopril dosage be increased given the latest BP reading?
```

**Example 3: Implementing Tamper-Evidence Verification**

User: "Add an integrity check endpoint that verifies no bead in a patient's history has been modified."

Approach:
1. Retrieve all beads for the patient via descendant traversal from root
2. For each bead, recompute `SHA-256(CanonicalJSON(bead_without_id))`
3. Verify the computed hash matches the stored bead ID
4. Verify every `parents` reference in each bead points to a bead that exists and whose hash is valid
5. Return a verification report

Output:
```json
{
  "patient_id": "a1b2c3...",
  "total_beads": 42,
  "verified": 42,
  "failed": 0,
  "integrity": "PASSED",
  "chain_depth": 8,
  "orphaned_references": []
}
```
If tampering is detected:
```json
{
  "integrity": "FAILED",
  "failures": [
    {
      "bead_id": "g7h8i9...",
      "expected_hash": "g7h8i9...",
      "computed_hash": "x9y8z7...",
      "reason": "Content modified: BP value changed from 158 to 128"
    }
  ]
}
```

## Best Practices

- **Do:** Use canonical JSON (sorted keys, no extra whitespace) before hashing. Non-deterministic serialization will produce different hashes for identical content and break the entire chain.
- **Do:** Treat the SQLite index as a disposable cache. Always implement `ReindexStorage()` that rebuilds the database by scanning the CAS directory. The CAS files are the source of truth.
- **Do:** Store all text content inline in the Bead's `content` field so the LLM can access it without additional retrieval calls. Only use `evidence` references for large binaries (DICOM images, PDFs).
- **Do:** Append Correction Beads to fix errors rather than modifying existing beads. Parent the correction to the erroneous bead so the causal link is explicit.
- **Avoid:** Using vector/embedding-based search to retrieve clinical context. The entire point of MedBeads is that BFS traversal on explicit causal links replaces probabilistic retrieval.
- **Avoid:** Making the bead store mutable. Never implement UPDATE or DELETE operations on beads. If you need to revoke a bead, append a revocation bead that references it as a parent.
- **Avoid:** Putting access control logic outside the bead. Clearance rules are embedded per-bead because sensitivity is context-dependent (a psychiatric note needs different access than a blood pressure reading).

## Error Handling

- **Hash mismatch on write:** If the computed hash of a newly created bead collides with an existing CAS file, compare content byte-for-byte. Identical content is an idempotent no-op (content-addressable deduplication). Different content with same hash is a SHA-256 collision -- log a critical alert (astronomically unlikely but must be handled).
- **Broken parent reference:** If a bead references a parent hash that doesn't exist in CAS, flag it during verification. During FHIR import, process resources in dependency order (Patient first, then Encounters, then Conditions/Observations) to prevent dangling references.
- **SQLite index corruption:** Detect by comparing index row count against CAS file count. If mismatched, run `ReindexStorage()` to rebuild from CAS. This should be a standard recovery operation, not an emergency.
- **Oversized context window:** If a patient's full DAG exceeds the LLM's context limit, use the `depth` parameter on context retrieval to limit traversal scope. Prefer ancestor traversal (more causally relevant) over full descendant traversal.
- **FHIR resource without clear parent mapping:** Some FHIR resources (e.g., AllergyIntolerance) lack explicit encounter references. Default to parenting these to the Patient root bead and document the heuristic.

## Limitations

- **Not a replacement for a certified EMR.** MedBeads is a data substrate layer; it does not handle scheduling, billing, clinical decision support rules, or regulatory compliance workflows.
- **Append-only storage grows unboundedly.** For long-lived patient records, the CAS directory will grow continuously. Archival and tiering strategies are outside the current design.
- **Single-institution scope in the prototype.** Cross-institutional DAG merging (when a patient moves between hospitals) is described conceptually but not implemented. Merging two DAGs with shared ancestry requires careful conflict resolution.
- **Synthetic data validation only.** The paper validates against Synthea-generated FHIR bundles. Real-world EMR data has messier references, missing fields, and inconsistent coding that will require robust fallback logic during FHIR-to-Bead conversion.
- **No built-in consensus mechanism.** Unlike blockchain, MedBeads is a single-writer Merkle DAG. Multi-writer scenarios (multiple clinicians writing simultaneously) need external coordination to avoid conflicting bead chains.

## Reference

**Paper:** [MedBeads: An Agent-Native, Immutable Data Substrate for Trustworthy Medical AI](https://arxiv.org/abs/2602.01086v1) (Nakajima, 2026). Focus on Section 3 (Bead schema and Merkle DAG construction), Section 4 (FHIR-to-DAG mapping table), and Section 5 (BFS context retrieval algorithm with complexity analysis). The Go/Python/React prototype is open-source under Apache 2.0.