---
name: "discoverllm-executing-intents-discovering"
description: "Help users discover and form their intents through adaptive diverge-converge interaction, rather than just asking clarifying questions. Use when the user gives a vague or open-ended request such as 'build me a website', 'write something creative', 'make a CLI tool', 'design a component', 'help me refactor this', or 'create a visualization'."
---

# DiscoverLLM: Adaptive Intent Discovery for Ambiguous Requests

This skill teaches Claude to help users **discover what they actually want** when their requests are vague or open-ended. Instead of asking clarifying questions the user cannot answer ("What tone do you want?"), Claude adaptively **diverges** (presents concrete options to explore) when intent is unclear and **converges** (refines and implements) when the user's preferences crystallize. This is based on the DiscoverLLM framework, which achieves 10%+ better task outcomes while reducing conversation length by up to 40% compared to standard clarification-based approaches.

## When to Use

- When the user gives a broad, underspecified request like "build me an app" or "write a story" and you sense they haven't fully formed their vision
- When the user responds to clarifying questions with uncertainty ("I'm not sure", "whatever you think is best", "I don't know yet")
- When the task is creative or design-oriented: writing, UI components, data visualizations, CLI tools, API design
- When the user keeps revising after each output, suggesting they're discovering preferences through seeing results
- When a request has many valid interpretations and the user would benefit from seeing possibilities before committing
- When the user says things like "surprise me", "explore some options", or "I'll know it when I see it"

## Key Technique: The Diverge-Converge Pattern

The core insight from DiscoverLLM is that user ambiguity often stems from **incomplete self-knowledge**, not poor communication. Users frequently cannot articulate preferences they haven't formed yet. Asking "what style do you want?" fails when they don't know what styles exist or which they'd prefer. The solution is to **surface concrete options** that let users react and discover their own preferences.

The technique models user intent as a **hierarchy that progressively concretizes**. At the top are broad goals ("I want a website"). Below are increasingly specific preferences ("minimalist design" -> "monochrome palette" -> "dark mode with accent color"). Users start knowing only the root-level goals. Deeper preferences are *latent*: they exist as dispositions but become accessible only when the user encounters relevant options. Each time you present options and the user reacts, undiscovered intents emerge and the hierarchy fills in.

The behavioral pattern is adaptive **diverge-converge cycling**. Diverge when intents are abstract: present 2-3 distinct concrete alternatives spanning the space of reasonable interpretations. Converge when intents crystallize: integrate discovered preferences into a single focused output. The key is balance -- research shows base models converge too early (91% consecutive convergent turns), while naive "ask more questions" approaches diverge too much and waste time. Optimal behavior alternates fluidly: diverge -> user reacts -> converge on what resonated -> diverge on remaining unknowns -> converge toward final output.

## Step-by-Step Workflow

1. **Assess intent concreteness on intake.** When receiving a request, mentally categorize each aspect as concrete (user specified it) or abstract (user left it open). A request like "build a REST API for user management" has a concrete domain but abstract decisions around auth strategy, database choice, error handling patterns, and response formats.

2. **Identify the 2-3 highest-value divergence points.** Don't try to resolve everything at once. Pick the decisions that most fundamentally shape the output. For a CLI tool, this might be: interactive vs. non-interactive, config file vs. flags, output format. Prioritize decisions where seeing options would genuinely help the user.

3. **Diverge with concrete artifacts, not abstract questions.** Instead of asking "what style do you want?", produce 2-3 small concrete examples that embody distinct choices. Show a snippet of each approach. Let the user react to tangible things. Each option should be genuinely different -- spanning the space, not minor variations.

4. **Label each option with its trade-offs.** Briefly annotate what each option optimizes for ("Option A: Minimal -- fewest dependencies, easiest to understand" / "Option B: Batteries-included -- handles edge cases, heavier setup"). This helps users discover which trade-offs they care about.

5. **Read the user's reaction as an intent signal.** When the user responds, extract both explicit preferences ("I like option B") and implicit ones (they mentioned "clean" twice -- they value simplicity). Update your mental model of their intent hierarchy. Note which aspects are now concrete and which remain abstract.

6. **Converge on discovered intents immediately.** Take what you've learned and produce a focused output that integrates all discovered preferences. Don't ask more questions about things already resolved. Show progress by building on what's been decided.

7. **Diverge again only on remaining abstract aspects.** If the converged output reveals new decision points, present options for those specifically. Each diverge-converge cycle should narrow the remaining unknowns. The user should feel the output getting closer to what they want with each turn.

8. **Detect convergence readiness and commit.** When the user's feedback becomes minor tweaks rather than directional changes, stop exploring and produce the final output. Don't over-explore -- once core intents are discovered, converge fully and deliver.

9. **Maintain a running summary of discovered intents.** Internally track what's been decided so far. This prevents re-asking resolved questions and lets you produce coherent outputs that honor all accumulated preferences.

10. **Adapt cycle speed to the user.** Some users discover quickly (one diverge is enough). Others need multiple rounds. If a user immediately says "option A, go with that", converge fully. If they say "hmm, these are all close but not quite", diverge more finely within that space.

## Concrete Examples

**Example 1: Vague CLI Tool Request**

```
User: "Build me a CLI tool for managing my todos"

Assessment: Domain is concrete (todo management), but interaction model,
storage, features, and output format are all abstract.

Diverge (Turn 1):
"Here are three different directions this could go. Each is a working
skeleton you can react to:

**Option A: Single-file simplicity**
- Flat text file storage (~/.todos)
- `todo add "Buy milk"` / `todo done 1` / `todo list`
- Zero dependencies, works anywhere

**Option B: Rich interactive TUI**
- SQLite storage with tags, priorities, due dates
- `todo` launches an interactive terminal UI with vim-like keybindings
- Filtering, sorting, color-coded priorities

**Option C: Git-integrated project tracker**
- Stores todos as markdown in .todos/ directory
- `todo add --tag=bug "Fix login"` / `todo board` for kanban view
- Syncs with git, supports team workflows

Which of these feels closest to what you're after? Or grab pieces
from multiple."

User: "I like A's simplicity but I want due dates from B. No TUI though."

Intent discovered: simple interface, file-based storage, due dates,
no interactive mode. Still abstract: file format, date input syntax,
list display format.

Converge (Turn 2): Produce working implementation with flat file storage,
due date support, simple CLI interface. Present the actual code.

Diverge within output: "For due dates, I went with `todo add "Buy milk"
--due=tomorrow`. The list view shows overdue items first. Two things
I could go either way on -- want me to show alternatives?"
- Date display: relative ("in 2 days") vs absolute ("2026-02-15")
- Overdue handling: highlight in red vs separate section

User: "Relative dates, and red highlighting. Ship it."

Converge (Turn 3): Final complete implementation incorporating all
discovered preferences.
```

**Example 2: Open-Ended Component Design**

```
User: "Create a React notification component"

Assessment: Component type is concrete, but behavior, styling, positioning,
animation, and API surface are all abstract.

Diverge (Turn 1):
"Three different takes on notifications -- each implies a different UX
philosophy:

**Option A: Toast stack (bottom-right)**
```tsx
notify.success("Saved!")          // auto-dismisses in 3s
notify.error("Failed", {persist: true})  // stays until dismissed
```
Minimal API. Stacks vertically. Good for transient feedback.

**Option B: Inline alert banners (top of container)**
```tsx
<NotificationBanner type="warning" dismissible>
  Your trial expires in 3 days. <a href="/upgrade">Upgrade</a>
</NotificationBanner>
```
Content-rich. Supports actions and links. Good for persistent messages.

**Option C: Notification center (bell icon + drawer)**
```tsx
<NotificationCenter>  // renders bell icon with unread count
  // drawer slides out with notification history
</NotificationCenter>
```
Persistent history. Read/unread state. Good for async events.

Which pattern fits your use case?"

User: "Toast stack, but I need it to support action buttons too,
like an undo action."

Intent discovered: toast pattern, auto-dismiss, action button support,
stacking behavior. Still abstract: animation style, theme integration,
max visible count.

Converge (Turn 2): Build the full toast component with action button
support. Make reasonable defaults for unknowns (fade animation,
max 5 visible, 3s auto-dismiss).

User: "This is great. Can the animation be a slide-in from the right
instead of fade?"

Converge (Turn 3): Update animation. Intent hierarchy is now mostly
concrete -- deliver final polished component.
```

**Example 3: Writing Task with Unknown Direction**

```
User: "Help me write the intro for my project's README"

Assessment: Content type is concrete, but tone, audience, length,
and what to emphasize are abstract.

Diverge (Turn 1):
"Three openings, each pitching differently:

**Option A: Problem-first**
> Deploying ML models shouldn't require a PhD in DevOps. ModelShip
> eliminates the gap between `model.fit()` and production with a
> single command.

**Option B: Feature-first**
> ModelShip: GPU-optimized model serving with auto-scaling,
> A/B testing, and one-line deployment. Supports PyTorch,
> TensorFlow, and JAX.

**Option C: Story-first**
> We spent six months deploying our first model. Then we built
> ModelShip so nobody else has to. It's a deployment tool that
> treats ML engineers as first-class citizens.

React to any of these and I'll develop it further."

User: "A is the right vibe. But less casual -- this is for enterprise
teams, not solo devs."

Converge (Turn 2): Rewrite with the problem-first structure but
enterprise-appropriate tone, mentioning team workflows, compliance,
and reliability.
```

## Best Practices

- **Do:** Present options as concrete artifacts (code snippets, prose samples, design sketches), not as abstract descriptions. Users discover preferences by reacting to real things, not by answering hypothetical questions.
- **Do:** Keep divergent options genuinely distinct. Three minor variations don't help discovery. Span the space of reasonable interpretations so each option teaches the user something about their own preferences.
- **Do:** Converge eagerly once you have signal. One round of divergence is often enough. Don't keep exploring when the user has given you a clear direction.
- **Do:** Track accumulated decisions across turns. Never re-ask something already resolved. Each turn should build on everything discovered so far.
- **Avoid:** Asking open-ended clarifying questions like "what do you want?" when the user has already shown they don't know. Present options instead.
- **Avoid:** Presenting more than 3 options at once. Research shows this overwhelms rather than aids discovery. Two to three options is the sweet spot.
- **Avoid:** Diverging on trivial decisions. Reserve exploration for choices that meaningfully shape the output. Make sensible defaults for everything else.
- **Avoid:** Converging too early on the first interpretation that seems plausible. If the request is genuinely ambiguous, one round of divergence will save multiple rounds of revision.

## Error Handling

**User picks "none of the above":** This is valuable signal -- it means your options didn't span the space correctly. Ask what was wrong with each option specifically. Their critiques will reveal latent intents more effectively than the options themselves did.

**User wants to backtrack after convergence:** This contradicts the monotonic refinement assumption. Handle it gracefully: acknowledge the change, update your intent model, and re-diverge from the new starting point. Don't treat it as an error.

**User gives contradictory signals:** ("I want it simple but also feature-rich") Surface the tension explicitly: "These two pull in different directions. Here's what simple-with-one-rich-feature looks like vs. rich-with-simple-defaults." Help them discover which they actually prioritize.

**User just wants you to decide:** Some users don't want to explore. If they say "just pick one", converge immediately on the best default and note what you chose and why. Respect the meta-preference for delegation.

**Options feel too similar:** If you can't generate genuinely distinct options, the decision space may be too narrow to warrant divergence. Converge on the obvious approach and diverge on the next decision point instead.

## Limitations

- **Works best for open-ended creative/design tasks.** For well-specified technical problems ("fix this null pointer exception"), standard direct execution is better. Don't force diverge-converge on tasks with clear right answers.
- **Assumes intents exist as latent dispositions.** If the user genuinely has no preferences at all (not even latent ones), divergence may not converge. This is rare in practice -- most people react once they see options.
- **Monotonic refinement assumption.** The framework assumes preferences only narrow over time. Users who fundamentally change direction mid-conversation may need a reset rather than incremental refinement.
- **Adds one turn of latency.** The initial diverge takes a turn that direct execution wouldn't. For users who already know exactly what they want, this is wasted time. Read the signals -- if the request is specific and detailed, skip divergence.
- **Option quality depends on domain knowledge.** Divergence is only as good as the options presented. In unfamiliar domains, the options may not span the space effectively.

## Reference

**Paper:** [DiscoverLLM: From Executing Intents to Discovering Them](https://arxiv.org/abs/2602.03429v1) (Kim et al., 2026)
**Key takeaway:** Users are often ambiguous because they haven't formed their intents, not because they're hiding them. The optimal strategy adaptively alternates between divergent exploration (presenting concrete options) and convergent refinement (implementing discovered preferences), achieving better outcomes in fewer turns than either pure clarification or pure execution.