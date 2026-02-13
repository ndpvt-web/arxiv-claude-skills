---
name: "pull-requests-as-training"
description: "Apply the Clean-PR agentless repo-level code editing protocol: decompose issues into file localization, fine-grained navigation via AST, and minimal Search/Replace patch generation. Triggers: 'fix this issue across the repo', 'edit multiple files for this bug', 'agentless code edit', 'search replace patch', 'repo-level fix', 'locate and patch this bug'"
---

# Pull-Request-Driven Repo-Level Code Editing

This skill implements the agentless three-stage protocol from the Clean-PR paper for repository-level code editing. Instead of relying on iterative agent loops with tool calls, it decomposes an issue into three deterministic steps: (1) file localization from the repo tree, (2) fine-grained navigation to the exact function/class using AST-level reasoning, and (3) generation of minimal Search/Replace edit blocks that use unique context anchors rather than fragile line numbers. This approach produces precise multi-file patches with fewer hallucinations and without complex scaffolding.

## When to Use

- When the user describes a bug or feature request and expects edits across multiple files in a large repository
- When asked to "fix this issue" or "implement this change" and the relevant files are not yet identified
- When generating patches that must apply cleanly — the Search/Replace format guarantees deterministic application
- When working on SWE-bench-style tasks: issue description in, code patch out
- When the user wants a systematic approach to repo-wide changes rather than ad-hoc file-by-file edits
- When editing a codebase with thousands of files and only 1-3 need modification (needle-in-a-haystack localization)

## Key Technique

The Clean-PR method treats real-world GitHub pull requests as structured training signals. Raw PR diffs are noisy — 38% contain non-core changes (docs, configs), 25% are bot-generated, and 24% were never merged. The pipeline filters these, then converts surviving diffs into **minimal Search/Replace edit blocks**: the shortest contiguous span of original code needed to uniquely anchor an edit location within a file, paired with the replacement code. This avoids line-number fragility and ensures patches apply deterministically. A round-trip verification step discards any example where re-applying the generated blocks doesn't bit-wise match the ground-truth post-PR state.

At inference time, the protocol is **agentless** — no tool-use loops, no retry scaffolding. It decomposes the task into three sequential stages: (1) **File Localization** maps the issue description + repo tree to a ranked list of file paths, (2) **Fine-Grained Navigation** uses AST parsing to map within each file to the enclosing function or class definition that needs editing, and (3) **Patch Generation** produces Search/Replace blocks scoped to the localized context. Error-driven data augmentation improves robustness by training on hard negatives: semantically similar but incorrect files (Stage 2) and distractor code regions near the bug site that require no editing (Stage 3).

## Step-by-Step Workflow

1. **Gather the issue description.** Extract the bug report, feature request, or change specification. If linked issues exist, concatenate their titles and descriptions for fuller context.

2. **Build the repo tree.** Generate a flat listing of all source files in the repository (exclude non-code artifacts like `.md`, `.txt`, `.json`, images, and config files unless they are explicitly relevant). This is the haystack for file localization.

3. **File Localization.** Given the issue text and the repo tree, identify the 1-5 source files most likely to require modification. Rank them by relevance. Filter aggressively — average real-world issues touch only 1-2 files even in repos with 3,000+ files.

4. **Read candidate files.** Retrieve the full content of each localized file. If a file exceeds practical context limits, apply context windowing centered on the most relevant sections (imports, class definitions matching the issue keywords).

5. **Fine-Grained Navigation via AST reasoning.** For each file, identify the specific function(s) or class definition(s) that enclose the code needing modification. Use structural reasoning (indentation, class/function boundaries, decorator patterns) to narrow to the exact scope. Include hard-negative awareness: note nearby code regions that look related but should NOT be edited.

6. **Generate Search/Replace edit blocks.** For each edit site, produce a block in this format:
   - **Search**: The minimal unique contiguous span of existing code that unambiguously identifies the edit location within the file. Must match exactly one location.
   - **Replace**: The corrected or new code that replaces the search span.
   Keep search spans as short as possible while remaining unique. Never use line numbers as anchors.

7. **Round-trip verify.** Mentally (or actually) apply each Search/Replace block to the original file content. Confirm the search string matches exactly one location. Confirm the result matches intent. If ambiguous, expand the search context until unique.

8. **Assemble the patch.** Combine all Search/Replace blocks into a single structured patch, organized by file path. Present in a clear, deterministic format.

9. **Validate the patch against the issue.** Re-read the original issue and confirm every requirement is addressed. Check that no unrelated code was modified.

10. **Handle multi-language repos.** The technique works across Python, Java, TypeScript, Go, C/C++, Ruby, Rust, PHP, Scala, Swift, and Kotlin. Use language-appropriate AST reasoning for navigation.

## Concrete Examples

**Example 1: Bug fix in a Django web application**

User: "The `/api/users/` endpoint returns a 500 error when the `email` field is null. Fix it."

Approach:
1. Build repo tree — identify `views.py`, `serializers.py`, `models.py`, `tests/` under the `users/` app
2. File Localization — rank `users/serializers.py` and `users/views.py` as top candidates (serializer validation is the likely null-handling site)
3. Read `users/serializers.py` — find `UserSerializer` class with `email = serializers.EmailField()`
4. Fine-Grained Navigation — the `email` field declaration and any `validate_email` method are the edit scope
5. Generate Search/Replace block:

```
File: users/serializers.py

<<<< SEARCH
email = serializers.EmailField()
==== REPLACE
email = serializers.EmailField(required=False, allow_null=True, allow_blank=True)
>>>>
```

6. Verify — the search string `email = serializers.EmailField()` is unique in the file. The replacement adds null tolerance. Round-trip confirmed.

**Example 2: Multi-file refactor — rename a utility function**

User: "Rename `compute_hash` to `calculate_content_hash` everywhere in the repo."

Approach:
1. Grep for `compute_hash` across the repo tree to find all usage sites
2. File Localization — identify `utils/hashing.py` (definition), `services/upload.py`, `services/cache.py`, `tests/test_hashing.py` (usages)
3. For each file, navigate to the exact function call or definition site
4. Generate Search/Replace blocks for each file:

```
File: utils/hashing.py

<<<< SEARCH
def compute_hash(content: bytes) -> str:
==== REPLACE
def calculate_content_hash(content: bytes) -> str:
>>>>

File: services/upload.py

<<<< SEARCH
from utils.hashing import compute_hash
==== REPLACE
from utils.hashing import calculate_content_hash
>>>>

File: services/upload.py

<<<< SEARCH
    file_hash = compute_hash(file_content)
==== REPLACE
    file_hash = calculate_content_hash(file_content)
>>>>

File: services/cache.py

<<<< SEARCH
from utils.hashing import compute_hash
==== REPLACE
from utils.hashing import calculate_content_hash
>>>>

File: services/cache.py

<<<< SEARCH
    key = compute_hash(payload.encode())
==== REPLACE
    key = calculate_content_hash(payload.encode())
>>>>
```

5. Verify — each search string is unique within its respective file. All import and call sites are covered.

**Example 3: Feature addition across backend and frontend**

User: "Add a `last_login` timestamp field to the User model and display it on the profile page."

Approach:
1. File Localization — `models/user.py` (model definition), `api/serializers.py` (API serialization), `templates/profile.html` or `frontend/src/components/Profile.tsx` (display)
2. Navigate to `User` class in model, serializer fields list, and profile template/component
3. Generate Search/Replace blocks:

```
File: models/user.py

<<<< SEARCH
    created_at = models.DateTimeField(auto_now_add=True)
==== REPLACE
    created_at = models.DateTimeField(auto_now_add=True)
    last_login = models.DateTimeField(null=True, blank=True)
>>>>

File: api/serializers.py

<<<< SEARCH
    fields = ["id", "username", "email", "created_at"]
==== REPLACE
    fields = ["id", "username", "email", "created_at", "last_login"]
>>>>

File: frontend/src/components/Profile.tsx

<<<< SEARCH
        <p>Member since: {user.created_at}</p>
      </div>
==== REPLACE
        <p>Member since: {user.created_at}</p>
        <p>Last login: {user.last_login ?? "Never"}</p>
      </div>
>>>>
```

4. Verify uniqueness of each search span. Confirm the model field, serializer, and UI are all consistent.

## Best Practices

- **Do:** Keep search spans minimal but unique. Include just enough surrounding context (a line above or below) to disambiguate if the target line appears multiple times.
- **Do:** Preserve exact whitespace and indentation in search strings. A single space mismatch causes the patch to fail.
- **Do:** Process files in dependency order — model changes before serializer changes before view changes — so the patch tells a coherent story.
- **Do:** Use AST-level reasoning (function boundaries, class scope) rather than keyword grep to navigate to the right edit site. Semantic neighbors that look similar but need no change are common distractors.
- **Avoid:** Using line numbers as anchors. They shift with every edit and across branches.
- **Avoid:** Including unchanged boilerplate in your search span. The search block should be the minimal anchor, not the entire function body.
- **Avoid:** Editing files that don't need changes. If a file is "close" to the bug but not causally involved, leave it alone. Over-editing is a major source of patch failure.

## Error Handling

- **Search string matches zero locations:** The code may have changed since the user's description. Re-read the file and regenerate the search span to match current content exactly.
- **Search string matches multiple locations:** Expand the search context by including one or more surrounding lines until the match is unique. Never guess which occurrence is correct.
- **Patch applies but introduces a syntax error:** Re-check indentation in the replace block. Python and YAML are especially sensitive. Verify bracket/brace balance.
- **File localization misses a required file:** If the initial patch doesn't fully resolve the issue, re-scan the repo tree with broader keywords. Check for indirect dependencies (imports, config references, test files).
- **Context window exceeded for large files:** Center the window on the function or class identified during fine-grained navigation. Include imports at the top of the file separately if needed.

## Limitations

- This agentless protocol does not iterate. If the first patch attempt is wrong, it requires human feedback or a separate retry — there is no built-in tool-use loop to self-correct.
- Works best when the issue description is specific. Vague requests like "make it faster" produce weak file localization signals.
- Cannot handle changes that require runtime information (e.g., "fix the bug that occurs when I click the button" without a stack trace or reproduction steps).
- Large-scale mechanical changes (hundreds of files) are better served by traditional refactoring tools (sed, codemod) than by generating individual Search/Replace blocks.
- The technique assumes the repository is in a consistent, buildable state. If the codebase has pre-existing conflicts or broken syntax, localization and patching reliability drops.

## Reference

**Paper:** [Pull Requests as a Training Signal for Repo-Level Code Editing](https://arxiv.org/abs/2602.07457v1) (Zhu et al., 2026)
**Key takeaway:** The three-stage agentless protocol (file localization → AST-guided navigation → minimal Search/Replace patch generation) with unique-context anchoring and round-trip verification produces deterministic, high-quality multi-file patches without iterative agent scaffolding.