---
name: generative-ontology-structured-knowledge
description: >
  Constrain LLM generation with executable Pydantic schemas and multi-agent pipelines
  to produce structurally valid, domain-rich artifacts. Uses ontology-as-grammar to
  eliminate hallucinated structures while preserving creative output. Trigger phrases:
  "generate a valid game design", "schema-constrained generation", "build a multi-agent
  pipeline with Pydantic validation", "ontology-driven content generation",
  "structured creative generation with DSPy", "generate artifacts that pass domain validation".
---

# Generative Ontology: Schema-Constrained Multi-Agent Generation

This skill enables Claude to build systems where **domain ontologies encoded as Pydantic schemas constrain LLM generation**, while **multi-agent specialization drives creative quality**. The core insight from the paper "Generative Ontology" (Cheung, 2026) is that ontology provides the grammar and the LLM provides the creativity — combining executable type constraints with specialized agent roles eliminates structural hallucinations (effect size d=4.78) while producing the largest gains in output quality (fun d=1.12, depth d=1.59).

## When to Use

- When the user wants to generate complex structured artifacts (game designs, recipes, legal documents, software architectures) that must conform to domain rules
- When building a multi-agent pipeline where each agent has a specialized role and validates its output against a shared schema
- When the user needs to constrain LLM outputs to a domain vocabulary — e.g., only valid game mechanisms, legal clause types, or architectural patterns
- When implementing retrieval-augmented generation grounded in domain exemplars with ontology-based filtering
- When the user asks to build a DSPy pipeline with Pydantic output validation
- When designing systems that need both structural correctness and creative richness — not just one or the other

## Key Technique

**Generative Ontology** treats domain knowledge as executable code rather than passive documentation. A domain ontology (the valid concepts, relationships, and constraints of a field) is encoded as nested Pydantic `BaseModel` classes with `Literal` types, `Enum` constraints, `min_length` validators, and cross-field validators. These schemas become the output specification for DSPy signatures, meaning the LLM *must* produce output that parses and validates — or the framework retries with error feedback. This eliminates an entire class of failures: mechanisms without components, goals without end conditions, recipes without cooking methods.

**Multi-agent specialization** is the second pillar. Rather than one LLM call doing everything, a sequential pipeline assigns distinct professional roles — each with a defined "anxiety" (a persistent concern that prevents shallow agreement). For example, a Balance Critic asks "What breaks when optimized?" and a Theme Weaver asks "Does the theme feel alive in every mechanism?" The ablation study showed that schema validation alone eliminates structural errors but does *not* improve creative quality; multi-agent specialization produces the largest creative gains. Both are needed together.

**RAG grounding** forms the third pillar. Existing exemplars (e.g., published board games from BoardGameGeek) are embedded and indexed. Retrieval uses a two-phase strategy: first filter by ontology categories (e.g., matching mechanism types), then rank by semantic similarity to the theme. Retrieved exemplars demonstrate successful patterns — how mechanisms combine, which themes pair with which structures — giving the LLM concrete precedents rather than generating from scratch.

## Step-by-Step Workflow

1. **Identify the domain ontology.** List the core concepts, their valid values, and the relationships between them. Ask: what would a domain expert say makes an artifact *structurally valid*? For a board game: mechanisms, components, victory conditions, player dynamics. For a recipe: ingredients, techniques, equipment, timing constraints.

2. **Encode the ontology as Pydantic schemas.** Create a hierarchy of `BaseModel` classes. Use `Literal` types and `Enum` classes to restrict values to the domain vocabulary. Add `Field(min_length=...)` constraints to prevent empty placeholders. Use nested models to enforce hierarchical coherence (e.g., a `ComponentSet` inside a `GameOntology`).

   ```python
   from pydantic import BaseModel, Field
   from enum import Enum
   from typing import List, Literal

   class MechanismType(str, Enum):
       WORKER_PLACEMENT = "worker_placement"
       DECK_BUILDING = "deck_building"
       AREA_CONTROL = "area_control"
       ENGINE_BUILDING = "engine_building"
       # ... domain-specific vocabulary

   class ComponentSet(BaseModel):
       card_types: List[str] = Field(default_factory=list)
       board_description: str = Field(default="", min_length=0)
       tokens: List[str] = Field(default_factory=list)

   class GameOntology(BaseModel):
       title: str = Field(min_length=3)
       theme: str = Field(min_length=20)
       game_type: Literal["cooperative", "competitive", "semi-cooperative"]
       goal: str = Field(min_length=20, description="Victory condition")
       end_condition: str = Field(min_length=10, description="When the game terminates")
       primary_mechanisms: List[MechanismType] = Field(min_items=2, max_items=4)
       components: ComponentSet
   ```

3. **Add cross-field validators** to enforce domain coherence — e.g., if `DECK_BUILDING` is a mechanism, `card_types` must be non-empty. These catch semantic inconsistencies that type constraints alone miss.

   ```python
   from pydantic import model_validator

   class GameOntology(BaseModel):
       # ... fields above ...

       @model_validator(mode="after")
       def check_mechanism_components(self):
           requirements = {
               MechanismType.DECK_BUILDING: ("card_types", "Deck building requires card types"),
               MechanismType.AREA_CONTROL: ("board_description", "Area control requires a board"),
               MechanismType.WORKER_PLACEMENT: ("tokens", "Worker placement requires tokens"),
           }
           for mech in self.primary_mechanisms:
               if mech in requirements:
                   field, msg = requirements[mech]
                   if not getattr(self.components, field, None):
                       raise ValueError(msg)
           return self
   ```

4. **Define DSPy signatures with the schema as the output type.** The signature's docstring becomes the system prompt; field descriptions guide generation. Use `dspy.ChainOfThought` to add reasoning before structured output.

   ```python
   import dspy

   class DesignSignature(dspy.Signature):
       """You are an expert tabletop game designer. Produce complete,
       specific, mechanically coherent designs."""
       theme = dspy.InputField(desc="The thematic concept for the game")
       constraints = dspy.InputField(desc="Design requirements", default="")
       game_design: GameOntology = dspy.OutputField()
   ```

5. **Build the multi-agent pipeline with professional anxieties.** Define 3-5 specialized agents, each with a distinct domain concern. Wire them sequentially so each agent receives the cumulative output of prior agents.

   | Agent | Responsibility | Professional Anxiety |
   |-------|---------------|---------------------|
   | Mechanics Architect | Core systems, turn structure | "Is there meaningful player agency?" |
   | Theme Weaver | Narrative integration | "Does theme feel alive in every mechanism?" |
   | Component Designer | Physical elements | "Can players manipulate this smoothly?" |
   | Balance Critic | Exploit detection | "What breaks when optimized?" |
   | Fun Factor Judge | Engagement assessment | "Would I want to play this again?" |

6. **Implement RAG grounding.** Embed existing exemplars using a sentence-transformer model, store in a vector database (ChromaDB works well). At generation time, filter by ontology categories first (mechanism types, domain tags), then rank by semantic similarity to the input theme. Inject top-k exemplars into agent context.

7. **Wire the pipeline: sequential generation → critical evaluation → refinement.** Phase 1: Mechanics Architect → Theme Weaver → Component Designer (sequential, each builds on prior output). Phase 2: Balance Critic evaluates the assembled design. Phase 3: If issues found, a refinement pass addresses them. Phase 4: Fun Factor Judge scores the result.

8. **Add retry logic with error feedback.** When Pydantic validation fails, capture the `ValidationError`, include it in the retry prompt, and re-run. DSPy's `Assert` mechanism automates this. Typically 1-2 retries suffice.

9. **Validate the final output end-to-end.** Run the complete schema validation plus cross-field checks. Log any warnings. Return the validated, typed artifact.

10. **Evaluate with domain-specific metrics.** Define 5-9 evaluation dimensions relevant to the domain (for games: fun, strategic depth, thematic cohesion, elegance, tension, social interaction, player agency, replayability). Use LLM-as-judge with test-retest reliability checks (ICC > 0.75 is the target).

## Concrete Examples

**Example 1: Board Game Generation Pipeline**

User: "Build a Python system that generates complete board game designs from a theme prompt, validated against a game design ontology."

Approach:
1. Define `GameOntology` Pydantic schema with enums for `MechanismType`, `GameType`, nested `ComponentSet` and `PlayerDynamics` models
2. Create DSPy signatures for each agent role with the schema as the output type
3. Implement the 5-agent pipeline: Mechanics Architect → Theme Weaver → Component Designer → Balance Critic → Fun Factor Judge
4. Add cross-field validators (deck building requires cards, area control requires a board)
5. Wire retry logic: on `ValidationError`, feed error message back to LLM and retry (max 3 attempts)

Output structure:
```python
# Usage
pipeline = GameGenerationPipeline(model="claude-sonnet-4-20250514")
result = pipeline.generate(
    theme="Ancient Egyptian tomb raiders competing to assemble cursed artifacts",
    constraints="2-4 players, 60-90 minutes, medium complexity"
)

# result is a validated GameOntology instance
print(result.title)                # "Curse of the Pharaohs"
print(result.primary_mechanisms)   # [DECK_BUILDING, SET_COLLECTION, HIDDEN_INFO]
print(result.components.card_types)  # ["Artifact Fragment", "Curse", "Tool", "Guardian"]
# Validation guarantees: no mechanisms without components, no empty goals
```

**Example 2: Recipe Generation with Culinary Ontology**

User: "I want to generate recipes that always have valid technique-equipment pairings and proper timing sequences."

Approach:
1. Define a `CulinaryOntology` schema: `TechniqueType` enum (sautee, braise, ferment...), `EquipmentSet`, `IngredientList` with dietary constraint flags, `TimingSequence` with step ordering
2. Add validators: braising requires a Dutch oven or similar; fermentation requires time > 4 hours; deep frying requires oil and a thermometer
3. Build 3-agent pipeline: Flavor Architect (anxiety: "Are the flavor pairings actually complementary?"), Technique Specialist (anxiety: "Is this technique achievable with home equipment?"), Nutrition Critic (anxiety: "Does this meal have balanced macros?")
4. RAG ground in a corpus of published recipes filtered by cuisine type and technique

Output:
```python
class CulinaryOntology(BaseModel):
    title: str = Field(min_length=3)
    cuisine: Literal["french", "japanese", "mexican", "indian", "italian"]
    techniques: List[TechniqueType] = Field(min_items=1, max_items=4)
    equipment: EquipmentSet
    ingredients: List[Ingredient]
    steps: List[Step]  # ordered, with time estimates

    @model_validator(mode="after")
    def check_technique_equipment(self):
        requirements = {
            TechniqueType.BRAISE: ("dutch_oven", "Braising requires a Dutch oven"),
            TechniqueType.DEEP_FRY: ("thermometer", "Deep frying requires a thermometer"),
        }
        # ... validation logic
```

**Example 3: Adapting to Software Architecture Domain**

User: "Generate microservice architecture designs that are structurally valid — no service can depend on a capability that doesn't exist."

Approach:
1. Define `ArchitectureOntology`: `ServiceType` enum (api_gateway, event_bus, data_store...), `CommunicationPattern` (sync_rest, async_event, grpc), `Service` model with `depends_on` and `provides` capability lists
2. Cross-field validator: every `depends_on` entry must match a `provides` entry from another service in the architecture
3. Agents: API Architect (anxiety: "Can this handle 10x traffic?"), Security Reviewer (anxiety: "What's the blast radius of a compromised service?"), Data Flow Analyst (anxiety: "Is there an undetected circular dependency?")
4. RAG ground in published architecture case studies

## Best Practices

- **Do:** Start with the schema before writing any agent logic. The schema *is* the domain knowledge — get it right first. Field descriptions become prompt context, so write them carefully.
- **Do:** Assign each agent a specific "anxiety" — a professional concern that prevents rubber-stamping. Without anxiety, agents agree with each other and output quality degrades.
- **Do:** Use two-phase RAG retrieval: ontology-category filtering first, then semantic similarity. Pure semantic search returns thematically similar but structurally irrelevant exemplars.
- **Do:** Log validation errors and retry counts. If a particular field fails repeatedly, the schema constraint may be too strict or the field description unclear — tune the description before loosening the constraint.
- **Avoid:** Putting all domain knowledge in the prompt. Encode it in the schema. Prompts are suggestions; schemas are guarantees.
- **Avoid:** Running all agents in parallel. The sequential pipeline matters — each agent builds on prior output. Only the evaluation phase (Balance Critic) can run independently.
- **Avoid:** Skipping cross-field validators. Type constraints catch syntax errors; cross-field validators catch semantic incoherence (the most damaging kind of hallucination).

## Error Handling

- **Pydantic `ValidationError` on LLM output:** Capture the error, include the specific field failures in the retry prompt. DSPy's `Assert` automates this. Set max retries to 3 — if it still fails, the schema constraint likely conflicts with the prompt.
- **Agent produces output that validates but is low quality:** This is what the Balance Critic and evaluation agents catch. If quality is consistently low, check that agent anxieties are specific enough — generic concerns like "is this good?" produce generic output.
- **RAG retrieves irrelevant exemplars:** Verify the ontology-filtering step is active. Pure embedding similarity without category pre-filtering returns thematically similar but structurally unrelated results.
- **Token budget exceeded in multi-agent pipeline:** The full pipeline uses ~27k tokens per artifact (8.5x single-pass). If budget is constrained, drop the Fun Factor Judge first (smallest quality impact), then the Component Designer. Never drop the Balance Critic or schema validation.
- **Enum vocabulary too narrow:** If the LLM consistently tries to use values outside the enum, the domain vocabulary needs expansion. Add the missing terms to the `Enum` class rather than loosening to `str`.

## Limitations

- **Ontology development cost is front-loaded.** Building the Pydantic schema requires genuine domain expertise. The framework does not generate the ontology — you must supply it.
- **Creative ceiling.** The ablation study showed generated designs score 7-8 while published designs score 8-9 (fun gap d=1.86). Schema constraints ensure a high floor but may impose a ceiling on truly novel combinations.
- **Domains without clear validity criteria are poor candidates.** Poetry, abstract art, and open-ended creative writing lack the checkable constraints that make this approach work.
- **Cost scales with agent count.** Each additional agent adds an LLM call. The 5-agent pipeline costs ~8.5x a single call. Evaluate whether the quality gain justifies the cost for your domain.
- **Enum staleness.** Domain vocabularies evolve. The `MechanismType` enum needs periodic updates as new patterns emerge. Build a process for ontology maintenance.

## Reference

**Paper:** Cheung, B. (2026). "Generative Ontology: When Structured Knowledge Learns to Create." arXiv:2602.05636v2. [https://arxiv.org/abs/2602.05636v2](https://arxiv.org/abs/2602.05636v2)

**Code:** [https://github.com/bennycheung/GameGrammarCLI](https://github.com/bennycheung/GameGrammarCLI)

**What to look for:** Section 3 for the schema architecture and DSPy integration, Section 4 for the multi-agent pipeline and professional anxiety definitions, Section 5 for the ablation study proving that schema validation and multi-agent specialization address *different* failure modes (structural vs. creative), and Table 8 for the generalization checklist to new domains.