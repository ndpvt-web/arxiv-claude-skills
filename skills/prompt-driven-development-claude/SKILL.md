---
name: "prompt-driven-development-claude"
description: |
  Orchestrate prompt-driven development of large multi-module systems through structured, iterative natural-language workflows.
  Uses a phased development methodology with prompt taxonomies (feature requests, bug fixes, information sharing, architectural guidance)
  to build and maintain architectural coherence across thousands of lines of code without the user writing code manually.
  Trigger phrases:
  - "Build this entire system using prompt-driven development"
  - "Help me develop a complete framework/library through prompts only"
  - "I want to build a large project iteratively without writing code"
  - "Guide me through prompt-driven development of this application"
  - "Create a multi-module system using the PDD workflow"
  - "Develop this TUI/GUI/CLI framework step by step through prompts"
---

# Prompt-Driven Development with Claude Code

This skill enables Claude to serve as the primary code author in building large, multi-module software systems (1,000-10,000+ lines) through a structured prompt-driven workflow. Based on empirical research where a 7,420-line TUI framework was built through 107 categorized prompts across five development phases, this methodology treats the human as architect/validator and the LLM as implementer. The human specifies requirements, validates behavior, and issues corrective prompts — never writing code directly. Architectural coherence is maintained through phased development, explicit information sharing, and a disciplined prompt taxonomy.

## When to Use

- When a user wants to build a complete library, framework, or multi-module application from scratch using only natural-language prompts
- When developing software for a language or ecosystem where the user is not fluent but can describe desired behavior
- When building a system with 5+ interconnected modules (e.g., a windowing subsystem, event architecture, widget library, layout engine)
- When the user says "I want to build X without writing any code myself" or "develop this entirely through prompts"
- When iteratively constructing a large codebase where architectural coherence across files is critical
- When prototyping a production-grade tool for a niche or emerging programming language where training data is sparse

## Key Technique

Prompt-Driven Development (PDD) structures the human-LLM interaction into a **five-phase development lifecycle** with a **typed prompt taxonomy**. Rather than treating AI code generation as one-shot or ad-hoc, PDD recognizes that building large systems requires distinct phases: (1) foundational infrastructure and core abstractions, (2) primary subsystem construction (e.g., window management), (3) controls and widget expansion, (4) complex UI systems integration, and (5) polish, documentation, and stabilization. Each phase has a different ratio of prompt types — early phases are heavy on feature requests and architectural guidance, while middle phases shift toward bug fixes and corrective iteration.

The prompt taxonomy classifies every interaction into one of five types: **feature requests** (new capabilities), **bug fix prompts** (corrective behavior), **information sharing** (providing documentation, API references, or language-specific knowledge the LLM lacks), **architectural guidance** (steering structural decisions), and **documentation generation**. The empirical data shows that bug fix prompts dominate (~67%), feature requests comprise ~20%, and information sharing ~8%. This distribution reveals that PDD is fundamentally an iterative refinement process — the LLM generates plausible first drafts, and the human steers through rapid corrective cycles. Most prompts are short (1-2 sentences), enabling a tight feedback loop where validation and correction happen in near-real-time.

The critical insight is that **architectural coherence emerges from phased structure and prompt discipline, not from the LLM's memory alone**. The human must front-load architectural decisions in early phases, provide explicit information when the LLM lacks domain knowledge, and resist the urge to skip ahead to features before infrastructure is solid. Bug prompts should be specific and observable ("the window redraws incorrectly when resized" not "it looks wrong"), and feature requests should describe behavior, not implementation.

## Step-by-Step Workflow

1. **Define the system scope and decompose into phases.** Before writing any code, outline 3-5 development phases ordered by dependency. Phase 1 is always foundational infrastructure (core data structures, base classes, configuration). Later phases build on earlier ones. Write this phase plan as a document the LLM can reference.

2. **Establish the prompt taxonomy for the session.** Decide which prompt types you will use: feature requests (FR), bug fixes (BF), information sharing (IS), architectural guidance (AG), and documentation (DOC). Label prompts explicitly when communicating — this helps both the human and the LLM track what kind of response is expected.

3. **Begin Phase 1 with architectural guidance prompts.** Start with 2-4 AG prompts that establish core abstractions, module boundaries, naming conventions, and the event/data flow architecture. These prompts should be the most detailed of the entire session. Example: "Create a base Widget class with properties for position, size, visibility, and focus state. All widgets must support an event handler callback pattern. Use a parent-child hierarchy for layout."

4. **Issue feature requests for Phase 1 primitives.** Request foundational capabilities one at a time. Each FR prompt should describe the desired behavior and its relationship to existing code. Validate each feature before moving to the next — do not batch unvalidated features.

5. **Share domain-specific information proactively.** When the LLM lacks knowledge about the target language, framework, or domain (e.g., Ring language syntax, terminal escape codes, platform-specific APIs), provide documentation excerpts, code samples, or API signatures as IS prompts before requesting features that depend on them.

6. **Transition to bug fix iteration within each phase.** After each feature is generated, test it. When bugs appear, issue specific BF prompts that describe: (a) the observed behavior, (b) the expected behavior, and (c) the reproduction steps. Categorize bugs mentally: redraw/rendering issues, event handling faults, runtime errors, or layout inconsistencies. This helps you write sharper corrective prompts.

7. **Progress to subsequent phases only when the current phase is stable.** Do not start Phase 3 (e.g., widget expansion) before Phase 2 (e.g., window manager) is functionally correct. Each phase depends on the previous one. Premature advancement causes cascading bugs that are harder to diagnose.

8. **Use the ~67/20/8 prompt distribution as a health metric.** If your bug fix ratio is much higher than 67%, the LLM may be struggling with the domain — provide more IS prompts. If feature requests dominate and you have few bug prompts, you may not be testing thoroughly enough. If architectural guidance prompts are needed late in the project, revisit whether early abstractions were well-defined.

9. **Maintain a running changelog of completed features and known issues.** After every 10-15 prompts, summarize what has been built and what remains. This serves as a context anchor for the LLM and prevents architectural drift in long sessions.

10. **Close with documentation generation and final stabilization.** Reserve the last phase for DOC prompts (generating README, API docs, usage examples) and a final pass of BF prompts to resolve edge cases discovered during documentation writing.

## Concrete Examples

**Example 1: Building a CLI dashboard framework**

```
User: "I want to build a terminal dashboard framework in Python that supports
multiple panels, real-time data updates, keyboard navigation, and theming.
I don't want to write any code — guide me through prompt-driven development."

Approach:
Phase Plan:
  Phase 1: Core rendering engine (screen buffer, terminal abstraction, color system)
  Phase 2: Layout system (panels, splits, grid positioning)
  Phase 3: Widget library (text, charts, tables, progress bars)
  Phase 4: Event system (keyboard input, focus management, real-time refresh)
  Phase 5: Theming, configuration, and documentation

Session Start (AG prompts):
  Prompt 1 [AG]: "Create a Screen class that wraps terminal I/O. It should
  maintain an internal character buffer matching terminal dimensions, support
  writing characters at (x,y) positions with foreground/background colors,
  and flush changes to stdout using ANSI escape codes. Use a dirty-region
  tracking system to minimize redraws."

  Prompt 2 [AG]: "Define a Widget base class with: position (x, y), size
  (width, height), visible flag, focus state, and a render(screen) method.
  Widgets must register with a parent container. Use a compositor pattern
  where the root container triggers top-down render passes."

Phase 1 Feature Requests:
  Prompt 3 [FR]: "Add a Color class supporting 16 ANSI colors, 256-color
  mode, and RGB. The Screen class should accept Color objects for fg/bg."

  Prompt 4 [BF]: "The screen buffer doesn't clear properly when the terminal
  is resized. After resize, old content persists in areas outside the new
  dimensions. Expected: full clear and re-render on SIGWINCH."

Output after Phase 1 (~15 prompts):
  - screen.py: Screen class with buffer, dirty tracking, ANSI rendering (180 lines)
  - widget.py: Widget base class, Container, compositor (120 lines)
  - color.py: Color system with ANSI/256/RGB support (60 lines)
```

**Example 2: Iterative bug fix cycle for an event system**

```
User: "The tab key should cycle focus between widgets in the active panel,
but pressing tab does nothing."

This is a BF prompt. The effective corrective sequence:

  Prompt 1 [BF]: "Tab key press is not being captured by the event loop.
  I verified the terminal is in raw mode. Expected: pressing Tab triggers
  a KEY_TAB event in the event queue. Observed: no event appears."

  Claude identifies the issue: Tab (0x09) was being filtered as whitespace
  in the input parser. Fix applied.

  Prompt 2 [BF]: "Tab events now fire, but focus doesn't move. The
  FocusManager.next() method is called but the highlight doesn't change
  visually. The focused widget's border should turn bright white."

  Claude traces the issue: render() was reading stale focus state because
  the focus change happened after the render pass. Fix: trigger re-render
  after focus change.

  Prompt 3 [BF]: "Focus cycling works but wraps incorrectly — after the
  last widget it focuses an invisible widget instead of returning to the
  first. Expected: skip widgets where visible=False during focus cycling."

  Three targeted BF prompts resolve what initially seemed like one bug but
  was actually three distinct issues across input, state, and rendering.
```

**Example 3: Information sharing for an unfamiliar language**

```
User wants to build a framework for an emerging language (e.g., Ring, Zig,
or a domain-specific language). The LLM has limited training data.

  Prompt 1 [IS]: "In Ring, classes are defined with: class ClassName
  [attributes list]. Methods use func keyword. String interpolation uses
  #{expr}. Here's the syntax for creating a class with inheritance:

  class Button from Widget
      title
      onclick
      func init(t) { title = t onclick = NULL }
      func render(scr) { scr.writeAt(x, y, '[' + title + ']') }
  "

  Prompt 2 [FR]: "Using the Ring class syntax I just showed you, create a
  TextInput widget that extends Widget. It should display a single-line
  text field with cursor, support character insertion and backspace, and
  emit an onChange event when the content changes."

  The IS prompt provides the syntactic foundation; the FR prompt builds on
  it. Without the IS prompt, the LLM would likely generate invalid syntax.
```

## Best Practices

- **Do:** Front-load architectural decisions in the first 5-10% of prompts. The AG prompts at the start determine whether the remaining 90% of prompts produce coherent code or spaghetti.
- **Do:** Keep bug fix prompts concrete and tripartite: observed behavior, expected behavior, reproduction context. Vague prompts like "it's broken" waste iteration cycles.
- **Do:** Share documentation and language references proactively before requesting features that depend on domain-specific knowledge. Don't wait for the LLM to guess wrong.
- **Do:** Test and validate each feature before requesting the next. Untested features create hidden dependencies that surface as confusing bugs later.
- **Avoid:** Batching multiple unrelated feature requests into a single prompt. One feature per prompt enables cleaner validation and easier rollback.
- **Avoid:** Skipping phases or building advanced features before foundational infrastructure is stable. A widget library built on a buggy rendering engine will require rewriting both.
- **Avoid:** Writing code yourself "just to fix this one thing." Every manual edit creates a divergence the LLM doesn't know about and can overwrite in the next generation.
- **Avoid:** Providing implementation instructions ("use a dict with keys...") instead of behavioral specifications ("it should support O(1) lookup by widget ID"). Let the LLM choose implementation; you specify behavior.

## Error Handling

- **Architectural drift after many prompts:** If the LLM starts generating code that contradicts earlier architectural decisions, issue a brief AG prompt restating the core patterns. Provide a summary of the current architecture as context reinforcement.
- **Persistent bug after 3+ fix attempts:** The LLM may be patching symptoms rather than root causes. Step back and issue an IS prompt explaining the underlying system behavior, then re-state the bug with more context about the subsystem interaction.
- **Domain knowledge gaps causing invalid code:** When the LLM generates syntactically invalid code for the target language, do not issue a BF prompt — issue an IS prompt with correct syntax examples first, then re-request the feature.
- **Feature works in isolation but breaks integration:** This signals a phase boundary problem. Validate inter-module contracts explicitly by issuing an AG prompt that restates the interface between the modules before fixing the integration bug.
- **Context window pressure in long sessions:** After 50+ prompts, the LLM may lose earlier context. Periodically re-anchor with a summary prompt: "Here is the current module structure and their responsibilities: [list]. Continue with the next feature in Phase 3."

## Limitations

- **Context window constraints:** PDD sessions for large systems (7,000+ lines) can exceed context limits. The methodology works best with tools that support extended context or file-based context (like Claude Code's file reading capabilities). Pure chat interfaces will struggle beyond ~3,000 lines.
- **Domain knowledge ceiling:** For languages or frameworks with minimal training data, the LLM's code quality depends heavily on the quantity and quality of IS prompts. If the user cannot provide good documentation excerpts, the bug fix ratio will spike unproductively.
- **Not suitable for performance-critical code:** PDD excels at structural/architectural code (UI frameworks, CRUD systems, API layers) but is less reliable for performance-sensitive algorithms, cryptographic implementations, or real-time systems where subtle bugs have severe consequences.
- **Requires a human who can validate behavior:** The methodology assumes the human can test and identify bugs accurately. If the human cannot run and evaluate the generated code, the corrective feedback loop breaks down.
- **Single-developer scaling:** The paper documents a single human working with one LLM instance. PDD for team-based development with multiple humans issuing prompts against the same codebase is unexplored.

## Reference

**Paper:** Fayed & Fayed, "Prompt Driven Development with Claude Code: Building a Complete TUI Framework for the Ring Programming Language" (arXiv:2601.17584, 2026). Look for: the five-phase development breakdown, prompt taxonomy distribution (Table 1), bug category analysis, and the finding that 67% of prompts are corrective — confirming PDD is an iterative refinement methodology, not a one-shot generation approach.