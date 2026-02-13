---
name: "environment-in-the-loop-rethinking-code-migration-"
description: "Perform code migrations (dependency upgrades, API changes, framework transitions) with integrated environment verification. Instead of migrating code then hoping it builds, this skill builds and tests inside a real environment at every step, using feedback loops to fix both code and configuration issues. Use when: 'migrate this project from X to Y', 'upgrade dependency version', 'port this codebase to a new framework', 'fix build after dependency update', 'help me upgrade NumPy/React/Django/Spring', 'automate this library migration'."
---

# Environment-in-the-Loop Code Migration

This skill enables Claude to perform code migrations — dependency upgrades, API adaptations, framework transitions — by treating the execution environment as a first-class participant rather than an afterthought. Based on the Environment-in-the-Loop (EITL) framework from Li et al. (2026), the core insight is that code and its environment co-evolve: migrating code without simultaneously constructing and verifying the target environment leads to hidden runtime failures, configuration drift, and prolonged rework cycles. This skill implements a tight feedback loop where every code change is immediately validated against a real build/test environment, and diagnostic output from that environment drives the next round of fixes.

## When to Use

- When the user asks to upgrade a dependency to a new major version (e.g., "upgrade NumPy from 1.x to 2.x", "migrate to React 18", "upgrade Spring Boot 2 to 3")
- When migrating a project between frameworks or runtimes (e.g., "port this Express app to Fastify", "migrate from Python 2 to Python 3")
- When a build is broken after dependency changes and the user needs systematic diagnosis and repair
- When the user needs to update API calls after a library deprecates or removes functions
- When upgrading language versions (e.g., Java 11 to Java 21, Node 16 to Node 22) and the ecosystem must follow
- When the user says "it builds locally but fails in CI" or "it worked before the upgrade" — classic environment-code mismatch symptoms

## Key Technique

Traditional LLM-assisted migration treats code transformation and environment setup as separate, sequential steps: first rewrite the code, then try to build it. This fails because many migration errors are **invisible to static analysis**. For example, NumPy 2.x changed internal constraints on `np.concatenate` while keeping the same function signature — code that looks correct fails at runtime with a type error. These version-dependent behavioral changes only surface when you actually execute the migrated code in the target environment.

The EITL framework solves this with a three-agent architecture operating in a continuous feedback loop. The **Migration Agent** generates candidate code changes and dependency specifications. The **Environment Agent** (the central hub) autonomously constructs an isolated build environment — installing dependencies, resolving version conflicts, configuring build systems, and executing the project. When builds or tests fail, the Environment Agent captures diagnostic logs and routes them: configuration errors (wrong Python version, missing system library) are self-repaired by the Environment Agent; semantic code errors are sent back to the Migration Agent for correction. The **Testsuite Agent** generates and runs regression tests within the verified environment, ensuring behavioral equivalence between old and new versions.

The critical innovation is that environment feedback is **continuous and structured**, not a one-shot pass/fail. Each iteration produces a diagnostic report classifying the failure type (dependency conflict, API incompatibility, configuration drift, runtime behavioral change) and prescribing the correction pathway. This eliminates the "blind retry" pattern where developers repeatedly tweak code without understanding root causes.

## Step-by-Step Workflow

1. **Audit the current environment state.** Read `package.json`, `requirements.txt`, `pom.xml`, `build.gradle`, `Dockerfile`, or equivalent manifests. Identify the current dependency versions, language runtime version, and build system. Run the existing build/test suite to establish a green baseline.

2. **Define the migration target explicitly.** Clarify exactly what is being upgraded (specific library version, language version, framework). Check the target library's changelog and migration guide for breaking changes. Document known incompatibilities as a checklist.

3. **Create an isolated environment for the migration.** Use Docker, a virtual environment, or a clean branch. The environment must be reproducible — write a Dockerfile or shell script that provisions it from scratch. Never mutate the user's working environment directly.

4. **Update dependency manifests first.** Change version numbers in lock files and manifests. Run the package manager's dependency resolution (`npm install`, `pip install`, `mvn dependency:resolve`) inside the isolated environment. Capture all output — version conflicts and resolution failures are the first feedback signal.

5. **Apply code transformations based on known breaking changes.** Using the migration guide and changelog from step 2, transform API calls, update import paths, replace deprecated patterns. Make changes file by file, grouping related changes.

6. **Build the project inside the target environment and capture diagnostics.** Run the full build. On failure, parse the error output to classify each issue:
   - **Dependency conflict**: version incompatibility between transitive dependencies → adjust version constraints
   - **API incompatibility**: removed/changed function signatures → update calling code
   - **Configuration drift**: wrong runtime version, missing system packages → fix environment provisioning
   - **Runtime behavioral change**: same API, different behavior → add explicit constraints or adapt logic

7. **Fix the highest-priority issue and rebuild.** Address one class of error at a time, starting with environment/configuration issues (they block everything else), then dependency conflicts, then API changes. Rebuild after each fix to get fresh diagnostics.

8. **Run regression tests inside the verified environment.** Execute existing tests. If tests are sparse, write targeted tests for the migrated code paths — especially around functions whose behavior changed across versions. Compare test results against the baseline from step 1.

9. **Iterate the feedback loop until green.** Repeat steps 6-8. Each cycle should resolve at least one class of error. If a fix introduces new failures, classify them and prioritize. Track progress explicitly — the error count should monotonically decrease.

10. **Produce a migration artifact.** Output the final set of code changes, updated manifests, updated Dockerfile/environment script, and a summary of what changed and why. This artifact should allow anyone to reproduce the migration from scratch.

## Concrete Examples

**Example 1: Upgrading NumPy from 1.x to 2.x in a data science project**

User: "Upgrade this project from NumPy 1.24 to NumPy 2.0"

Approach:
1. Read `requirements.txt` and `setup.py`. Identify all NumPy-dependent code by grepping for `import numpy` and `np.`.
2. Check NumPy 2.0 migration guide — note removed aliases (`np.bool`, `np.int`, `np.float`), changed `concatenate` behavior, removed `np.mat`.
3. Create a virtualenv with Python 3.11 and install `numpy==2.0.0` plus all other dependencies at their current versions.
4. Update `requirements.txt` to pin `numpy>=2.0.0,<3.0.0`.
5. Run `pip install -r requirements.txt` — capture any dependency conflicts (e.g., `scipy 1.10` requires `numpy<1.27`). Resolve by upgrading `scipy` to a 2.0-compatible version.
6. Replace all `np.bool` with `np.bool_`, `np.int` with `np.int_`, `np.float` with `np.float64` across the codebase.
7. Build and run tests. A test using `np.concatenate` on mixed dtypes fails with a new type error — NumPy 2.0 is stricter about dtype promotion. Fix by adding explicit `dtype=` argument.
8. Re-run full test suite — all green. Produce diff and updated requirements.

Output:
```
Migration complete: 14 files changed, 37 API calls updated.
- Replaced 22 deprecated type aliases (np.bool → np.bool_, etc.)
- Upgraded scipy 1.10.1 → 1.14.0 for NumPy 2.0 compatibility
- Fixed 3 dtype promotion issues in concatenate/stack calls
- Updated requirements.txt and Dockerfile
- All 148 tests passing in target environment
```

**Example 2: Spring Boot 2 to Spring Boot 3 migration (Java 17)**

User: "Migrate our Spring Boot 2.7 app to Spring Boot 3.2"

Approach:
1. Audit `pom.xml` — current Spring Boot 2.7.x, Java 11. Note: Spring Boot 3 requires Java 17 minimum.
2. Document breaking changes: `javax.*` → `jakarta.*` namespace, removed `WebSecurityConfigurerAdapter`, changed auto-configuration class loading.
3. Create a Docker environment with JDK 17 and Maven 3.9.
4. Update `pom.xml`: Spring Boot parent to 3.2.x, Java source/target to 17, update Spring Cloud and other Spring ecosystem dependencies.
5. Run `mvn dependency:resolve` — capture conflicts. A transient dependency pulls in `javax.servlet` which conflicts with `jakarta.servlet`. Exclude it and add the Jakarta equivalent.
6. Run `find` + `sed` to replace all `javax.persistence` → `jakarta.persistence`, `javax.servlet` → `jakarta.servlet`, etc.
7. Refactor security config: replace `WebSecurityConfigurerAdapter` extends with `SecurityFilterChain` bean pattern.
8. Build with `mvn compile` — 4 errors remain around removed `spring.factories` auto-config. Migrate to `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
9. Run `mvn test` — 2 test failures from changed default behavior in MockMvc content-type handling. Fix assertions.
10. Full build green. Output migration summary with all changes grouped by category.

Output:
```
Migration complete: 89 files changed.
- javax → jakarta namespace: 64 files (automated replacement)
- Security config refactored: 3 files (manual pattern change)
- Auto-configuration migrated to new imports file
- pom.xml: 12 dependency version updates, 3 exclusions added
- Dockerfile updated: JDK 11 → JDK 17
- All 312 tests passing on Spring Boot 3.2.4 / JDK 17
```

**Example 3: Python 3.8 to 3.12 upgrade with environment repair**

User: "Our CI just switched to Python 3.12 and everything is broken"

Approach:
1. Reproduce the CI environment locally — build a Docker image matching the CI runner with Python 3.12.
2. Run `pip install -r requirements.txt` — immediately see `distutils` import errors (removed from stdlib in 3.12). Install `setuptools` as a dependency to restore `distutils`.
3. Build the project — `imp` module import fails (removed in 3.12). Replace `import imp` with `import importlib` and update call sites.
4. Run tests — `asyncio.coroutine` decorator usage fails (removed). Replace with `async def` syntax.
5. Two C extension dependencies fail to compile — `cffi 1.14` and `pycryptodome 3.9` don't support 3.12. Upgrade both to latest compatible versions.
6. Tests pass. Produce updated `requirements.txt` with pinned compatible versions and a Dockerfile that matches the CI environment exactly.

Output:
```
Migration complete: 8 files changed, 4 dependency upgrades.
- Replaced distutils imports with setuptools equivalents
- Replaced imp module usage with importlib (2 files)
- Modernized 5 asyncio.coroutine decorators to async def
- Upgraded cffi 1.14→1.17, pycryptodome 3.9→3.21
- Dockerfile aligned with CI runner (python:3.12-slim)
- All 94 tests passing on Python 3.12.1
```

## Best Practices

- **Do: Always establish a green baseline first.** Run existing tests on the current version before changing anything. You need a reference point to know when migration is complete.
- **Do: Fix environment issues before code issues.** If the build environment itself is broken (wrong runtime version, missing system libraries), no amount of code changes will help. Always get a clean `install` step before trying to `build`.
- **Do: Classify errors before fixing them.** Read the full error output and determine if the failure is a dependency conflict, API change, configuration problem, or behavioral change. The fix strategy differs for each.
- **Do: Make changes incrementally and rebuild after each batch.** One category of fix at a time. This prevents cascading confusion where you can't tell which change caused a new failure.
- **Avoid: Fixing code without verifying in the target environment.** The whole point of EITL is that static reasoning about compatibility is unreliable. Always build and test.
- **Avoid: Upgrading all dependencies simultaneously.** Upgrade the primary target first, then resolve cascade conflicts one at a time. Mass upgrades make it impossible to diagnose which change caused which failure.
- **Avoid: Ignoring deprecation warnings from the previous version.** Most breaking changes in major versions were deprecation warnings in prior minor versions. Check warning output from the baseline build — it previews what will break.

## Error Handling

| Error Type | Diagnosis | Resolution |
|---|---|---|
| **Dependency resolution failure** | Package manager can't find a compatible version set | Relax version constraints, check for alternative packages, or pin transitive dependencies |
| **Build failure after manifest update** | Compilation errors from removed/changed APIs | Consult migration guide, apply API transformations, rebuild |
| **Tests pass locally but fail in container** | Environment mismatch (system libraries, locale, timezone) | Compare `env`, installed packages, and OS-level dependencies between environments |
| **Silent behavioral change** | Tests pass but output differs from baseline | Add assertion-level regression tests comparing old vs. new behavior on edge cases |
| **Circular dependency after upgrade** | Two packages each require incompatible versions of a third | Check if either package has a newer release resolving the conflict, or pin the shared dependency and test both consumers |
| **C extension compilation failure** | Native code incompatible with new runtime | Upgrade the extension package, or if unmaintained, find a pure-Python alternative |

## Limitations

- **Requires a buildable starting point.** If the project doesn't build on the current version, establish a working baseline first before attempting migration.
- **Cannot predict all runtime behavioral changes.** Some behavioral differences only manifest on specific inputs or under concurrency. The feedback loop catches what tests cover — but test coverage limits detection.
- **Large monorepos with many interdependent services** may require coordinated migration across multiple environments simultaneously, which exceeds the single-project scope of this workflow.
- **Proprietary or closed-source dependencies** that don't publish changelogs make it difficult to anticipate breaking changes. The feedback loop still works but each iteration is less informed.
- **Environment construction time** can be significant for projects with heavy native dependencies or large Docker images. Each feedback iteration pays this cost.

## Reference

Li, X., Fei, Z., Ma, Y., Zhang, J., & Sarro, F. (2026). *Environment-in-the-Loop: Rethinking Code Migration with LLM-based Agents.* arXiv:2602.09944v1. [https://arxiv.org/abs/2602.09944v1](https://arxiv.org/abs/2602.09944v1)

Key takeaway: Section 3 describes the three-agent (Migration, Environment, Testsuite) feedback loop architecture. Figure 3 shows the full workflow. The diagnostic classification scheme (dependency conflict vs. API change vs. configuration drift vs. behavioral change) is the most directly actionable element for implementation.