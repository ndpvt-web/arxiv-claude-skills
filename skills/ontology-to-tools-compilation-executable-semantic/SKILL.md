---
name: "ontology-to-tools-compilation-executable-semantic-"
description: "Compile domain ontologies (OWL/RDFS/JSON-LD schemas) into executable tool interfaces with embedded semantic constraints, so LLM agents enforce domain rules during generation rather than post-hoc. Use when: 'compile my ontology into tools', 'enforce schema constraints in agent tools', 'generate MCP tools from OWL', 'build knowledge graph extraction pipeline', 'ontology-driven tool generation', 'semantic constraint enforcement for agents'."
---

# Ontology-to-Tools Compilation for Executable Semantic Constraint Enforcement

This skill enables Claude to compile formal domain ontologies (OWL, RDFS, JSON Schema, or any structured schema) into executable tool interfaces that embed semantic constraints directly into their input schemas. Instead of letting an LLM generate unconstrained output and then validating it after the fact, this approach makes constraint violations structurally impossible at the tool-call boundary. The technique originates from The World Avatar framework and generalizes to any workflow where domain knowledge must govern LLM behavior -- knowledge graph population, structured data extraction, scientific literature mining, or API generation from data models.

## When to Use

- When the user has an OWL/RDFS ontology or JSON-LD context and wants to generate tool definitions (MCP tools, OpenAPI endpoints, or function-calling schemas) that enforce the ontology's constraints
- When building an agent pipeline that extracts structured data from unstructured text (papers, reports, logs) and the extracted data must conform to a known domain model
- When the user asks to "generate MCP tools from my schema" or "compile my data model into agent tools"
- When designing a knowledge graph ingestion workflow where LLM agents create/modify RDF instances and must respect cardinality, domain/range, and datatype restrictions
- When the user wants to reduce prompt engineering by encoding domain rules into tool schemas rather than natural-language instructions
- When building a validate-and-repair loop where an agent iteratively fixes constraint violations until output is semantically valid

## Key Technique

**Compile-time constraint embedding, not runtime validation.** Traditional approaches let an LLM generate JSON or RDF triples freely, then run a SHACL or JSON Schema validator to catch errors. This paper inverts that: the ontology is compiled *before* the agent runs into tool schemas whose input parameters structurally encode every constraint. Cardinality restrictions become `minItems`/`maxItems` on arrays. Range restrictions become `enum` values or `$ref` pointers to nested object schemas. Domain constraints determine which tools accept which entity types. The LLM literally cannot call a tool with invalid arguments because the schema rejects them at the function-call layer.

**The compilation pipeline has three phases.** (1) *Parse* the ontology to extract classes, object properties, datatype properties, restrictions (cardinality, allValuesFrom, someValuesFrom, hasValue), and class hierarchies. (2) *Generate* one tool per ontology class (for instance creation) and one tool per complex operation (linking instances, updating properties), where each tool's JSON Schema input encodes the parsed constraints. (3) *Expose* the tools via MCP server (or any tool-use protocol) so agents discover and invoke them at runtime.

**The agent workflow is extract-validate-repair.** Given unstructured input (e.g., a scientific paper), the agent: reads the text, selects the appropriate ontology-compiled tool, fills in parameters by extracting information, receives immediate feedback if constraints are violated (e.g., missing required field, wrong datatype, cardinality exceeded), then repairs its extraction and retries. This loop converges because each iteration narrows the space of valid completions.

## Step-by-Step Workflow

1. **Ingest the ontology or schema.** Read the user's OWL/TTL file, RDFS, JSON-LD context, or even a well-structured JSON Schema. Identify all classes (entities), object properties (relationships between entities), datatype properties (literal attributes), and restrictions (cardinality, value constraints, domain/range).

2. **Build a constraint map per class.** For each class, collect: (a) required properties (min cardinality >= 1), (b) optional properties (min cardinality = 0), (c) max cardinalities, (d) range types for each property (another class, or a datatype like `xsd:string`, `xsd:float`), (e) enumerated allowed values (`oneOf`), (f) value restrictions (`minInclusive`, `maxInclusive`, `pattern`).

3. **Generate one "create" tool per class.** The tool name follows the pattern `create_{ClassName}`. Its input schema is a JSON Schema object where each property maps to an ontology property, with `type`, `enum`, `minimum`, `maximum`, `minItems`, `maxItems`, `pattern`, and `required` fields derived from the constraint map. Nest object properties as `$ref` to other class schemas.

4. **Generate relationship and update tools.** For each object property that links two classes, generate a `link_{PropertyName}` tool that accepts source and target IRIs and validates domain/range. For mutable properties, generate `update_{ClassName}_{property}` tools.

5. **Add a `validate_instance` tool.** This tool accepts a class name and a candidate JSON object, runs full constraint checking, and returns a structured list of violations with human-readable messages. This enables the repair loop.

6. **Compose the MCP server manifest (or function-calling tool list).** Bundle all generated tools into a single MCP `tools` array with names, descriptions (auto-generated from ontology labels/comments), and input schemas. Write this to a `tools.json` or serve it via an MCP endpoint.

7. **Wire the extract-validate-repair agent loop.** Build an agent prompt that: (a) reads unstructured input, (b) identifies which ontology classes are relevant, (c) calls the corresponding `create_` tool with extracted data, (d) if the call fails validation, reads the error, fixes the extraction, and retries up to N times.

8. **Persist valid instances to the knowledge graph.** On successful tool invocation, serialize the validated instance as RDF triples (or JSON-LD) and insert into a triplestore (e.g., Blazegraph, Fuseki) or append to a local graph file.

9. **Iterate for multi-entity documents.** For documents containing multiple entities and relationships, the agent processes entities in dependency order (referenced entities first), then links them using relationship tools.

10. **Audit and report.** After processing, generate a summary of: entities created, constraint violations encountered and repaired, any unresolved extraction failures, and coverage statistics.

## Concrete Examples

**Example 1: Compiling a chemistry ontology into MCP tools**

User: "I have an OWL ontology for metal-organic polyhedra (MOP) synthesis. Compile it into tools my agent can use to extract synthesis data from papers."

Approach:
1. Parse the OWL file. Identify classes: `MOP`, `SynthesisMethod`, `Ligand`, `MetalCenter`, `SolventSystem`.
2. Extract restrictions: `MOP` has exactly 1 `MetalCenter` (cardinality = 1), 1..* `Ligand` (min 1), optional `SolventSystem`, a required `xsd:string` for `formulaNotation`.
3. Generate tools.

Output (partial `tools.json`):
```json
{
  "tools": [
    {
      "name": "create_MOP",
      "description": "Create a Metal-Organic Polyhedron instance. Enforces: exactly one metal center, at least one ligand, required formula notation.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "formulaNotation": {
            "type": "string",
            "description": "Chemical formula notation for the MOP"
          },
          "metalCenter": {
            "$ref": "#/$defs/MetalCenterRef",
            "description": "IRI of the metal center (exactly one required)"
          },
          "ligands": {
            "type": "array",
            "items": { "$ref": "#/$defs/LigandRef" },
            "minItems": 1,
            "description": "Ligands involved in the MOP (at least one)"
          },
          "solventSystem": {
            "$ref": "#/$defs/SolventSystemRef",
            "description": "Optional solvent system used in synthesis"
          },
          "yieldPercent": {
            "type": "number",
            "minimum": 0,
            "maximum": 100,
            "description": "Synthesis yield as percentage"
          }
        },
        "required": ["formulaNotation", "metalCenter", "ligands"]
      }
    },
    {
      "name": "create_Ligand",
      "description": "Create a Ligand instance with SMILES notation and name.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "name": { "type": "string" },
          "smiles": { "type": "string", "pattern": "^[A-Za-z0-9@+\\-\\[\\]\\(\\)=#$/\\\\.]+$" }
        },
        "required": ["name", "smiles"]
      }
    }
  ]
}
```

**Example 2: Generating API tools from a product catalog schema**

User: "Here's my JSON Schema for an e-commerce product catalog. Generate constrained tool definitions so my agent can only create valid products."

Approach:
1. Read the JSON Schema. Identify: `Product` has required `name` (string, maxLength 200), `price` (number, minimum 0.01), `category` (enum: ["electronics", "clothing", "home"]), optional `tags` (array, maxItems 10), `weight_kg` (number, minimum 0).
2. Generate a `create_Product` tool whose input schema mirrors these constraints exactly.
3. Generate an `update_Product_price` tool that accepts a product ID and a new price (with the same minimum constraint).

Output:
```json
{
  "name": "create_Product",
  "inputSchema": {
    "type": "object",
    "properties": {
      "name": { "type": "string", "maxLength": 200 },
      "price": { "type": "number", "minimum": 0.01 },
      "category": { "type": "string", "enum": ["electronics", "clothing", "home"] },
      "tags": { "type": "array", "items": { "type": "string" }, "maxItems": 10 },
      "weight_kg": { "type": "number", "minimum": 0 }
    },
    "required": ["name", "price", "category"]
  }
}
```

**Example 3: Extract-validate-repair loop on a research paper**

User: "Extract all catalyst entities from this paper abstract and populate my chemistry knowledge graph. Use the ontology tools we compiled."

Approach:
1. Read the abstract text. Identify candidate entities: a catalyst name, its composition, reaction conditions.
2. Call `create_Catalyst` with extracted fields. If the tool rejects (e.g., missing required `activationEnergy` field), parse the error.
3. Re-read the abstract for the missing field. If not present in text, call with explicit `null` if optional, or flag as incomplete.
4. On success, call `link_catalyzes` to connect the catalyst to the reaction entity.

Output (agent trace):
```
Step 1: Extracted candidate — name: "Pd/C", support: "activated carbon"
Step 2: Called create_Catalyst(name="Pd/C", support="activated carbon")
  → REJECTED: missing required field "metalLoading" (minCardinality=1)
Step 3: Re-scanned text. Found "5 wt% Pd loading"
Step 4: Called create_Catalyst(name="Pd/C", support="activated carbon", metalLoading=5.0, metalLoadingUnit="wt%")
  → ACCEPTED: instance iri:catalyst_001 created
Step 5: Called link_catalyzes(catalyst="iri:catalyst_001", reaction="iri:rxn_042")
  → ACCEPTED
```

## Best Practices

- **Do:** Map every OWL restriction to a concrete JSON Schema keyword. `owl:minCardinality 1` becomes `"required"` + `"minItems": 1`. `owl:maxCardinality 1` becomes a singular value (not array) or `"maxItems": 1`. `owl:allValuesFrom` becomes a `$ref` or `enum`. Leave no constraint as a natural-language description only.
- **Do:** Generate human-readable `description` fields on every tool parameter by pulling `rdfs:label` and `rdfs:comment` from the ontology. This gives the LLM contextual understanding of what each field means.
- **Do:** Use `$defs` and `$ref` for shared class schemas so that the same entity definition is reused across tools (e.g., a `Ligand` schema referenced by both `create_MOP` and `create_Reaction`).
- **Avoid:** Encoding complex OWL axioms (disjointness, property chains, SWRL rules) as prompt instructions. If they cannot be expressed in JSON Schema, implement them as server-side validation logic inside the tool handler, not as LLM instructions.
- **Avoid:** Creating a single monolithic "create anything" tool. One tool per class ensures the LLM selects the correct schema and cannot mix fields from unrelated entities.
- **Avoid:** Skipping the repair loop. Set a retry budget (3-5 attempts) per entity extraction. Log all constraint violations for downstream analysis of extraction quality.

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| Schema validation failure on tool call | LLM extracted wrong type or missing field | Return structured error listing each violation. Agent re-extracts from source. |
| Ontology parsing failure | Malformed OWL/TTL syntax | Run a syntax checker (e.g., `rapper -c`) before compilation. Report line numbers. |
| Circular `$ref` in generated schema | Mutual ontology class references | Break cycles with IRI-string references instead of inline object nesting. |
| Agent exhausts retry budget | Information genuinely absent from source text | Mark entity as `incomplete`, log which constraints could not be satisfied, continue to next entity. |
| Tool explosion (too many tools) | Ontology has hundreds of classes | Group tools by namespace/module. Expose only the subset relevant to the current task via MCP resource filtering. |

## Limitations

- **Expressivity ceiling.** JSON Schema cannot represent all OWL 2 axioms. Property chains, disjoint unions, and complex class expressions (intersectionOf with nested restrictions) require custom server-side validation beyond what the tool schema alone can enforce.
- **Ontology quality dependency.** If the source ontology has vague or missing constraints (e.g., no cardinality restrictions, overly broad ranges), the compiled tools will be correspondingly permissive. Garbage-in, garbage-out.
- **Scale limits.** Ontologies with 500+ classes produce large tool manifests that may exceed LLM context windows. Requires partitioning or dynamic tool loading.
- **No inference.** The compiled tools enforce *asserted* constraints. They do not run OWL reasoners to infer implicit constraints (e.g., subclass transitivity). If inferred constraints matter, run a reasoner on the ontology first and compile the inferred version.
- **Extraction quality.** The constraint enforcement catches structural errors but cannot verify factual correctness of extracted values (e.g., a yield of 99% that is actually 9.9% in the paper).

## Reference

**Paper:** [Ontology-to-tools compilation for executable semantic constraint enforcement in LLM agents](https://arxiv.org/abs/2602.03439v1) (Zhou et al., 2026). Look for: the three-phase compilation pipeline (parse, generate, expose), the constraint-to-JSON-Schema mapping table, and the extract-validate-repair agent loop architecture.