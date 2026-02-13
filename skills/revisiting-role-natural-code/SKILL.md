---
name: "revisiting-role-natural-code"
description: "Comment-augmented code translation (COMMENTRA) that uses targeted natural language comment injection to significantly improve LLM-based cross-language code translation. Use when: 'translate this Java to Python', 'convert C++ code to Go', 'port this Python function to C', 'translate code between languages', 'fix a failed code translation', 'improve code translation accuracy'."
---

# COMMENTRA: Comment-Augmented Code Translation

This skill enables Claude to perform high-accuracy code translation between programming languages (C, C++, Go, Java, Python) by strategically injecting natural language comments into source code before translation. Based on a large-scale empirical study of 80,000+ translations, the COMMENTRA technique can double translation accuracy by first generating short, descriptive comments that explain what the code does, then using the commented code as input for translation. The key insight: comments that describe overall purpose dramatically improve translation, while verbose or multi-intent comments add noise and can degrade quality.

## When to Use

- When the user asks to translate/convert/port code from one programming language to another (e.g., Java to Python, C++ to Go, Python to C)
- When a previous code translation attempt produced compilation errors, runtime failures, or incorrect output
- When translating algorithmic code that relies on language-specific idioms the target language handles differently
- When converting code between statically-typed (C, C++, Java, Go) and dynamically-typed (Python) languages
- When the user has code with complex logic that needs careful preservation during translation
- When porting code from repositories or competitive programming solutions across language boundaries

## Key Technique

COMMENTRA is an iterative, comment-injection approach to code translation. Instead of translating raw source code in a single pass, it first attempts an uncommented translation. If that translation fails (compilation error, runtime error, or wrong output), it injects short descriptive comments into the original source code and retranslates. This targeted approach avoids unnecessary comment overhead on code that translates correctly without help, while rescuing failed translations through semantic clarification.

The critical finding is that **comment intent matters enormously**. Comments must be *descriptive* -- short statements of what the code does ("find the longest increasing subsequence", "swap elements at indices i and j"). Other intents actively hurt: *explanatory* comments (describing the algorithm approach), *analytical* comments (mentioning time complexity), and *precautionary* comments (warning about edge cases) all introduce noise that misleads the LLM. The ideal comment is a brief, inline annotation next to code blocks explaining their functional purpose.

Placement also matters: inline comments placed adjacent to specific code blocks outperform method-level summary docstrings and pseudocode-style annotations. The LLM benefits most from localized semantic context that clarifies what each section of code accomplishes, not from a top-level summary of the entire function.

## Step-by-Step Workflow

1. **Receive the source code and identify the source and target languages.** Confirm the translation pair (e.g., Java to Python). Identify language-specific constructs that will need careful handling: memory management (C/C++), type systems (Go interfaces vs Java generics), concurrency primitives, and standard library differences.

2. **Attempt a direct translation without comments first.** Translate the raw source code to the target language. This is the baseline attempt -- many straightforward translations succeed without comment augmentation.

3. **Validate the translation.** Check the output for: (a) syntax/compilation errors, (b) obvious logic errors from language idiom mismatches, (c) incorrect handling of data structures or standard library equivalents. If the user provides test cases, mentally trace through them.

4. **If the translation fails or looks suspicious, inject descriptive inline comments into the original source code.** For each logical block (loop body, conditional branch, function section), add a short comment describing *what it does*, not how or why. Use this format:
   - `// find the maximum element in the array` (good -- descriptive)
   - `// using dynamic programming approach with O(n^2) complexity` (bad -- explanatory/analytical noise)

5. **Keep comments short, single-intent, and in English.** Each comment should be one clause describing the functional purpose of the adjacent code. Do not comment every line -- target logical blocks (3-8 lines) that perform a cohesive operation. English comments consistently outperform comments in other natural languages.

6. **Retranslate the commented source code.** Use the commented version as input. The descriptive comments provide semantic anchors that guide the LLM to select correct target-language idioms and library functions.

7. **Validate the retranslation.** Check for the same categories of errors. If failures persist, examine which code blocks caused the issue and refine comments to be more precisely descriptive of those specific blocks.

8. **If a second attempt fails, try varying the comment style.** Replace terse comments with slightly more detailed (but still descriptive) versions. Focus comments specifically on the code blocks that map to the error location in the failed translation.

9. **Strip comments from the final translated code.** Once the translation succeeds, remove the injected comments from the output. The comments were a translation aid, not part of the deliverable.

10. **Preserve any original comments from the source.** If the source code had meaningful comments written by the author, translate those into appropriate target-language comment style and retain them.

## Concrete Examples

**Example 1: Java to Python -- Algorithm with language-specific idioms**

User: "Translate this Java code to Python"
```java
public static int longestIncreasingSubsequence(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1);
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
    }
    return Arrays.stream(dp).max().orElse(0);
}
```

Approach:
1. Attempt direct translation -- this is straightforward enough it may succeed.
2. If needed, inject descriptive comments into the Java source:

```java
public static int longestIncreasingSubsequence(int[] nums) {
    int n = nums.length;
    // initialize dp array where dp[i] stores LIS length ending at index i
    int[] dp = new int[n];
    Arrays.fill(dp, 1);
    // for each element, check all previous elements for valid extensions
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            // extend the subsequence if current element is larger
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
    }
    // return the maximum LIS length found
    return Arrays.stream(dp).max().orElse(0);
}
```

3. Translate the commented version:

```python
def longest_increasing_subsequence(nums):
    n = len(nums)
    # initialize dp array where dp[i] stores LIS length ending at index i
    dp = [1] * n
    # for each element, check all previous elements for valid extensions
    for i in range(1, n):
        for j in range(i):
            # extend the subsequence if current element is larger
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)
    # return the maximum LIS length found
    return max(dp) if dp else 0
```

4. Strip injected comments from final output, delivering clean Python.

**Example 2: C++ to Go -- Memory and type system differences**

User: "Convert this C++ to Go"
```cpp
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int, int> seen;
    for (int i = 0; i < nums.size(); i++) {
        int complement = target - nums[i];
        if (seen.count(complement)) {
            return {seen[complement], i};
        }
        seen[nums[i]] = i;
    }
    return {};
}
```

Approach:
1. Direct translation attempt likely works here but may mishandle return types.
2. If the Go translation has issues (e.g., wrong map syntax, slice return), add comments:

```cpp
// find two indices whose values sum to target using a hash map lookup
vector<int> twoSum(vector<int>& nums, int target) {
    // map from value to its index for O(1) complement lookup
    unordered_map<int, int> seen;
    for (int i = 0; i < nums.size(); i++) {
        // calculate the value needed to reach the target
        int complement = target - nums[i];
        // check if complement was seen in a previous iteration
        if (seen.count(complement)) {
            return {seen[complement], i};
        }
        // record current value and its index
        seen[nums[i]] = i;
    }
    // return empty if no pair found
    return {};
}
```

3. Retranslate to produce correct Go:

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int)
    for i, num := range nums {
        complement := target - num
        if j, ok := seen[complement]; ok {
            return []int{j, i}
        }
        seen[num] = i
    }
    return []int{}
}
```

**Example 3: Rescuing a failed Python to C translation**

User: "This Python to C translation has runtime errors, can you fix it?"

Approach:
1. Read the original Python source and the failed C translation.
2. Identify where the C translation diverges (e.g., missing memory allocation, wrong array bounds, incorrect string handling).
3. Add descriptive comments to the Python source at the failure points:
   - `# allocate result array of size n` next to list comprehensions
   - `# read space-separated integers from stdin` next to `input().split()`
   - `# build adjacency list for undirected graph` next to dict-of-lists construction
4. Retranslate the commented Python to C, producing correct `malloc`/`free` pairs, proper `scanf` patterns, and struct-based adjacency lists.
5. Validate the new translation against the reported errors.

## Best Practices

- **Do:** Write comments that describe *what* the code does in plain English ("sort the array by frequency", "merge overlapping intervals"). These are the highest-value comments for translation.
- **Do:** Focus comments on code blocks that use language-specific idioms (list comprehensions, Java streams, Go goroutines, C pointer arithmetic) since these are the hardest to translate correctly.
- **Do:** Keep each comment to a single short clause. One-line descriptive comments consistently outperform multi-sentence explanations.
- **Do:** Apply the iterative approach -- try without comments first, then add comments only to failed translations. This avoids unnecessary overhead and prevents comments from introducing noise into already-successful translations.
- **Avoid:** Writing comments about algorithm complexity, design rationale, or potential edge cases. These "analytical" and "precautionary" intents actively mislead the translation LLM.
- **Avoid:** Mentioning specific variable names, data structure names, or implementation details in comments that differ from what the target language would use. A comment saying "iterate over the grid" when the variable is named `matrix` can cause the LLM to introduce a spurious `grid` variable.

## Error Handling

**Comment-induced variable hallucination:** If a comment mentions a term (e.g., "grid", "tree", "buffer") that doesn't correspond to an actual variable in the source, the LLM may invent that variable in the translated code. Always ensure comment nouns match the actual code identifiers.

**Over-commenting degrades quality:** Adding comments to every single line creates noise. If a retranslation with comments performs worse than without, reduce comment density to target only the complex or idiomatic code blocks.

**Type system mismatches:** When translating between dynamically-typed (Python) and statically-typed languages (C, Java, Go), comments should explicitly state the expected types: "stores a list of integer pairs" rather than just "stores the results." This prevents the LLM from choosing wrong container types.

**Failed iterations:** If two rounds of comment-augmented translation still fail, the problem is likely not solvable by comments alone. Look for fundamental language capability gaps (e.g., Python generators to C, Go channels to Java) that require structural redesign rather than comment-guided translation.

## Limitations

- Comment augmentation helps most with algorithmic and data-processing code. System-level code with heavy OS/runtime dependencies (file I/O, networking, threading) often fails for reasons comments cannot address.
- The technique assumes the source code is functionally correct. Comments cannot fix bugs in the original -- they will faithfully describe broken logic and produce a broken translation.
- Translation between languages with vastly different paradigms (e.g., Python to C for memory-managed code) may need structural refactoring beyond what comment injection can guide.
- Comments in languages other than English provide less benefit. Stick to English-language comments for maximum translation improvement.
- The doubling of performance reported in the paper was measured on competitive programming benchmarks (AVATAR, CodeNet). Real-world codebases with framework dependencies, build systems, and external libraries present challenges that comments alone cannot resolve.

## Reference

[Revisiting the Role of Natural Language Code Comments in Code Translation](https://arxiv.org/abs/2601.16661v1) -- Gupta et al., 2026. Key sections: Section 4 (COMMENTRA algorithm and iterative workflow), Section 3.2 (comment intent taxonomy: descriptive > explanatory > analytical > precautionary), Table 4 (quantitative gains per iteration across language pairs).