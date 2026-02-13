---
name: swe-bench-mobile-agents-develop
description: >
  Apply defensive programming and agent-architecture patterns from SWE-Bench Mobile to tackle
  production iOS/mobile development tasks. Optimizes how Claude navigates large mixed-language
  codebases, interprets multi-modal inputs (PRDs + Figma designs), and generates robust patches.
  Use when: "build this iOS feature from a PRD", "implement this Figma design in Swift",
  "fix this mobile app issue across multiple files", "generate a patch for this iOS codebase",
  "help me with production mobile development", "defensive programming for mobile code".
---

# SWE-Bench Mobile: Defensive Agent Strategy for Production Mobile Development

This skill equips Claude to tackle production-grade mobile (especially iOS) development tasks using
the Defensive Programming agent strategy identified in SWE-Bench Mobile research. The core insight:
simple prompts focused on edge-case robustness outperform complex multi-step checklists by 7.4%,
and agent architecture choices (tool integration, context management, iterative refinement) matter
as much as raw model capability -- the same model shows up to 6x performance variance across
different agent scaffolding. This skill encodes the winning patterns.

## When to Use

- When the user provides a PRD (Product Requirement Document) and/or Figma design and asks you to implement a mobile feature
- When working in a large mixed Swift/Objective-C iOS codebase (or any multi-language mobile project)
- When generating a unified diff patch for a production mobile app
- When the user asks to implement a UI component, data management feature, gesture handler, or networking layer in an iOS app
- When a mobile task requires modifying 3+ files across model, view, and controller layers
- When the user needs help structuring agent-assisted mobile development workflows
- When debugging why an AI-generated mobile patch fails tests or misses requirements

## Key Technique: Defensive Programming over Comprehensive Checklists

The SWE-Bench Mobile benchmark evaluated 22 agent-model configurations on 50 industry-level iOS
tasks (449 test cases). The highest-performing prompt strategy was **Defensive Programming**: a
focused instruction to write robust, production-ready code that handles edge cases gracefully --
nil values, empty data, network timeouts, concurrent operations. This simple strategy achieved
26.7% test pass rate vs. 19.3% baseline, while a verbose "Comprehensive" checklist approach
dropped to just 4% task success (vs. 10% for Defensive Programming).

Why does simplicity win? Overly detailed process instructions misdirect attention toward workflow
compliance rather than implementation correctness. The Defensive Programming prompt keeps the agent
focused on what matters: generating code that actually works under real conditions. The research
also found that **agent architecture is a first-class concern** -- Cursor achieved 12% task success
with Opus 4.5 while OpenCode achieved only 2% with the same model. The difference comes from
tool integration quality, context window management, and iterative self-correction loops.

The dominant failure modes reveal where to focus effort: missing feature flags (54% of failures),
missing data models (22%), incomplete file coverage (11-15%), and missing UI components (11-15%).
Tasks requiring 1-2 files achieved 18% success vs. only 2% for 7+ files, showing that cross-file
reasoning in unfamiliar language ecosystems is the key bottleneck.

## Step-by-Step Workflow

1. **Parse the requirement into atomic deliverables.** Extract every concrete output from the PRD
   or user description: data models, UI components, API integrations, navigation flows, feature
   flags. Create an explicit checklist -- the #1 failure mode is omitting required artifacts.

2. **Inventory the codebase architecture before writing code.** Map the project structure: find
   existing patterns for models, views, controllers/coordinators, networking layers, and feature
   flags. In Swift/ObjC codebases, identify bridging headers and mixed-language boundaries. Search
   for naming conventions, base classes, and dependency injection patterns.

3. **Identify ALL files that need modification.** For each deliverable from step 1, trace which
   files must change. Err on the side of including more files -- incomplete file coverage causes
   11-15% of failures. Check for: model definitions, view implementations, view models/presenters,
   coordinators/routers, dependency registration, feature flag declarations, and test targets.

4. **Apply the Defensive Programming mindset to every code block.** For each function or component:
   - Handle nil/optional values explicitly (guard let, if let, nil coalescing)
   - Account for empty collections and missing data
   - Add timeout handling for async operations
   - Consider thread safety for concurrent access
   - Respect iOS lifecycle (viewDidLoad vs. viewWillAppear, dealloc patterns)

5. **Implement data models and feature flags FIRST.** These are prerequisite layers. Define structs/
   classes, Codable conformances, Core Data entities, or Realm objects before building UI or
   networking. Register feature flags in the project's existing flag system -- missing flags account
   for 54% of failures.

6. **Build UI components referencing Figma specs precisely.** Match spacing, colors, typography, and
   layout constraints to the design. Use Auto Layout or SwiftUI modifiers that correspond to the
   design system. Verify that dynamic content (variable-length text, missing images) degrades
   gracefully.

7. **Wire up networking and data flow with error boundaries.** Connect API calls, local persistence,
   and state management. Wrap each integration point in error handling that surfaces meaningful
   feedback rather than silent failures.

8. **Generate a minimal, correct unified diff patch.** Include only the files that must change.
   Verify the patch applies cleanly against the target branch. Each hunk should have sufficient
   context lines (3+) for unambiguous application.

9. **Self-review against the original requirements checklist.** Walk through every deliverable from
   step 1 and confirm it appears in the implementation. Check for the top failure modes: missing
   feature flags, missing data models, incomplete file coverage, missing UI components.

10. **Validate with available test infrastructure.** If tests exist, run them. If generating test-
    compatible output, ensure structural correctness (correct class names, method signatures,
    protocol conformances) since evaluation often uses diff-based structural analysis.

## Concrete Examples

**Example 1: Implementing a Profile Settings Screen from PRD**

User: "Here's the PRD for a new Profile Settings screen. It should show user avatar, name, email,
and a list of toggleable preferences. The Figma is attached. Implement this in our Swift codebase."

Approach:
1. Parse PRD deliverables: ProfileSettingsViewController, ProfileSettingsViewModel,
   UserPreference model, PreferenceCell, feature flag `profile_settings_v2_enabled`
2. Search codebase for existing patterns:
   ```bash
   # Find existing ViewControllers for pattern reference
   find . -name "*ViewController.swift" | head -20
   # Find feature flag registration
   grep -r "FeatureFlag" --include="*.swift" -l
   # Find existing table view cell patterns
   grep -r "UITableViewCell" --include="*.swift" -l | head -10
   ```
3. Identify files to create/modify: new model file, new VC, new VM, new cell,
   feature flag registration file, coordinator to add navigation route
4. Implement with defensive patterns:
   ```swift
   struct UserPreference: Codable {
       let id: String
       let title: String
       let isEnabled: Bool

       // Defensive: handle missing keys gracefully
       init(from decoder: Decoder) throws {
           let container = try decoder.container(keyedBy: CodingKeys.self)
           self.id = try container.decode(String.self, forKey: .id)
           self.title = try container.decodeIfPresent(String.self, forKey: .title) ?? "Unknown"
           self.isEnabled = try container.decodeIfPresent(Bool.self, forKey: .isEnabled) ?? false
       }
   }
   ```
5. Register feature flag, wire navigation, verify all 6 deliverables are covered

Output: Unified diff patch touching 6 files with defensive nil handling throughout.

**Example 2: Fixing a Gesture Interaction Bug Across Multiple Files**

User: "Our swipe-to-delete gesture on the Favorites list crashes when the list is empty and the
user swipes. Fix this in our mixed Swift/ObjC codebase."

Approach:
1. Locate the crash site -- search for swipe/delete gesture handling in Favorites:
   ```bash
   grep -r "swipe\|deleteRow\|commitEditingStyle" --include="*.swift" --include="*.m" -l
   ```
2. Identify the defensive gap: likely an unguarded array index access when data source is empty
3. Apply defensive fix with guard:
   ```swift
   func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCell.EditingStyle,
                  forRowAt indexPath: IndexPath) {
       guard editingStyle == .delete else { return }
       guard indexPath.row < favorites.count else { return }  // Defensive: bounds check

       let item = favorites[indexPath.row]
       favorites.remove(at: indexPath.row)

       tableView.performBatchUpdates({
           tableView.deleteRows(at: [indexPath], with: .automatic)
       }, completion: { _ in
           // Defensive: verify consistency after animation
           if self.favorites.isEmpty {
               self.showEmptyState()
           }
       })
   }
   ```
4. Check if the ObjC bridging header or any ObjC callers need updates
5. Verify empty-state UI is shown after last item deletion

Output: Patch modifying 2 files -- the gesture handler and the empty state view.

**Example 3: Structuring an Agent Workflow for a Complex Mobile Feature**

User: "I need to add offline caching for our feed. This touches networking, persistence, UI, and
sync logic. How should I structure the agent workflow?"

Approach:
1. Break into atomic tasks ordered by dependency:
   - Task 1: Define CachedFeedItem data model + Core Data schema
   - Task 2: Implement FeedCacheManager (write/read/evict)
   - Task 3: Add feature flag `offline_feed_cache_enabled`
   - Task 4: Modify FeedNetworkService to write-through to cache
   - Task 5: Update FeedViewController to load from cache on network failure
   - Task 6: Add sync conflict resolution logic
2. Recommend tackling each task as a separate focused prompt rather than one mega-prompt
   (simpler prompts outperform comprehensive ones by 7.4%)
3. For each task, apply defensive programming: handle empty cache, stale data, migration from
   no-cache to cached state, disk full errors, and concurrent read/write access
4. Verify each task independently before combining

Output: A structured task plan with 6 focused sub-tasks, each producing a testable patch.

## Best Practices

- **Do:** Keep implementation prompts focused on one concern at a time. The research shows simple,
  targeted prompts outperform complex checklists by a wide margin.
- **Do:** Always create an explicit deliverables checklist from the PRD before coding. Walk it
  after implementation. Missing artifacts are the #1 failure category.
- **Do:** Inventory the codebase first. Spend time reading existing patterns before generating
  code. Match naming conventions, architecture patterns, and dependency injection styles.
- **Do:** Treat feature flags as first-class deliverables. 54% of failures stem from missing
  production deployment toggles.
- **Avoid:** Writing a long, multi-part system prompt with step-by-step checklists for the agent.
  This misdirects attention toward process compliance rather than code correctness.
- **Avoid:** Attempting 7+ file changes in a single pass. Success drops from 18% (1-2 files) to
  2% (7+ files). Break large features into smaller, independently testable patches.

## Error Handling

| Failure Mode | Frequency | Mitigation |
|---|---|---|
| Missing feature flags | 54% | Always search for and match the project's feature flag system before submitting |
| Missing data models | 22% | Implement models first; treat them as blocking prerequisites |
| Incomplete file coverage | 11-15% | Trace every requirement to specific files; check coordinators, DI containers, and navigation |
| Missing UI components | 11-15% | Cross-reference Figma/PRD for every visual element; verify empty/loading/error states |
| Cross-language bridging errors | Common in ObjC/Swift | Check bridging headers, @objc annotations, NS_SWIFT_NAME macros |
| Async race conditions | Common in gesture/networking | Use DispatchQueue barriers, actors (Swift 5.5+), or serial queues for shared state |

## Limitations

- This approach is optimized for iOS (Swift/Objective-C) codebases. Android (Kotlin/Java) and
  cross-platform (React Native, Flutter) have different architectural patterns -- adapt the
  checklist accordingly.
- Even the best agent configuration achieves only 12% task success on the full benchmark.
  Production mobile development still requires substantial human review and iteration.
- Gesture and interaction tasks (8% avg pass rate) and media/asset tasks (9.8%) remain
  particularly difficult -- expect to need more manual intervention for these categories.
- The defensive programming approach helps most with data management (15.3% pass rate) and
  UI component tasks (12.5%). It has less impact on tasks requiring deep framework knowledge
  (e.g., custom Core Animation, Metal shaders).
- Multi-modal inputs (Figma designs) cannot be perfectly translated to code by current agents.
  Pixel-perfect implementation still requires human visual QA.

## Reference

**Paper:** [SWE-Bench Mobile: Can Large Language Model Agents Develop Industry-Level Mobile Applications?](https://arxiv.org/abs/2602.09540v1) (Tian et al., 2026)
**Leaderboard & Toolkit:** [swebenchmobile.com](https://swebenchmobile.com)

Key takeaway: Agent scaffolding design matters as much as model capability. Use simple, defensive
prompts focused on edge-case robustness rather than comprehensive process checklists, and break
large mobile features into small (1-3 file) independently testable patches.