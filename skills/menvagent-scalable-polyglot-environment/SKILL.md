---
name: "menvagent-scalable-polyglot-environment"
description: "Automated Docker environment construction for polyglot repositories using a Planning-Execution-Verification multi-agent loop with environment reuse. Use when: 'build a Docker environment for this repo', 'set up a reproducible test environment', 'create a verifiable dev container', 'construct an executable environment for this project', 'make this repo's tests runnable in Docker', 'set up CI environments for multiple languages'."
---

# MEnvAgent: Scalable Polyglot Environment Construction

This skill enables Claude to automatically construct working Docker environments for software repositories across 10 programming languages (Python, Java, TypeScript, JavaScript, Rust, Go, C++, Ruby, PHP, C). It applies the MEnvAgent Planning-Execution-Verification architecture: a closed-loop system where specialized reasoning phases analyze a repository, generate Docker build and test scripts, execute them in containers, diagnose failures through error attribution, and iteratively refine until the environment passes Fail-to-Pass verification. The key innovation is treating environment construction as a multi-agent problem with explicit feedback loops rather than a one-shot Dockerfile generation task.

## When to Use

- When the user asks to create a Docker environment for a repository and the language/build system is non-trivial
- When setting up reproducible test environments for open-source projects across any of the 10 supported languages
- When a Dockerfile or docker-compose setup repeatedly fails and needs iterative diagnosis and repair
- When constructing verifiable environments where tests must fail on buggy code and pass on fixed code (Fail-to-Pass criterion)
- When migrating a project's CI from one environment to another and the dependency chain is complex
- When the user has a polyglot repository (multiple languages) that needs a unified container environment
- When reusing or adapting a previous Docker environment for a newer version of the same project

## Key Technique

MEnvAgent decomposes environment construction into three phases operating in a closed loop. The **Planning** phase analyzes the repository structure (file tree, config files like `package.json`, `Cargo.toml`, `pom.xml`, `Gemfile`, `go.mod`, etc.) to produce three outputs: a project summary with language/build-system metadata, a base Docker image selection with a full installation script, and a test configuration script aligned with installed binaries. The **Execution** phase instantiates the container, runs the installation script, and performs dynamic error resolution for immediate failures (missing packages, version conflicts). The **Verification** phase runs the test suite inside the container and performs explicit **error attribution** -- classifying failures as either environment setup failures (wrong base image, missing dependency) or test execution failures (wrong test command, incompatible framework version) -- then feeds structured diagnostic feedback back to Planning for the next iteration.

The second core innovation is the **Environment Reuse Mechanism**. Rather than building every environment from scratch, the system retrieves the most similar historical environment for the same repository (or a close match) using a hierarchical strategy: first exact version matches, then backward-compatible newer versions. An **EnvPatchAgent** then generates incremental patch commands (delta scripts) to adapt the retrieved environment to the target repository state. This avoids full rebuilds and cuts construction time by ~43%. The reuse-first, build-as-fallback approach means the system tries patching before falling back to scratch construction.

The Fail-to-Pass (F2P) verification criterion is what makes environments *verifiable*: tests must fail on the original buggy code AND pass on the fixed code. This differential outcome proves the environment correctly reproduces the issue and validates the fix, making it suitable for SWE benchmark evaluation.

## Step-by-Step Workflow

1. **Analyze repository structure**: Read the file tree, identify the primary language(s), and locate build configuration files (`package.json`, `Cargo.toml`, `pom.xml`, `build.gradle`, `Makefile`, `CMakeLists.txt`, `Gemfile`, `composer.json`, `go.mod`, `pyproject.toml`/`setup.py`). Extract dependency lists, required language versions, and test framework references.

2. **Select base Docker image**: Choose the most appropriate base image based on the primary language and version requirements. Use official language images when possible (e.g., `python:3.10-slim`, `node:18`, `rust:1.75`, `golang:1.21`, `ruby:3.2`, `php:8.2-cli`, `gcc:13`). For multi-language repos, use a broader base like `ubuntu:22.04` with manual language installation.

3. **Generate the installation script**: Write the complete sequence of shell commands to install all dependencies, build tools, and system-level packages. Order matters: system packages first, language runtime configuration second, project dependencies third, build step fourth. Capture this as an executable shell script.

4. **Generate the test configuration script**: Identify the test framework (pytest, jest, mocha, cargo test, go test, maven test, rspec, phpunit, ctest, etc.) and write the exact commands to execute the relevant test subset. Include necessary environment variables, working directory setup, and any test filtering flags.

5. **Check for reusable environments**: If a prior Docker image or Dockerfile exists for this repository (or a similar version), attempt to reuse it. Run the test script inside the existing environment. If tests pass, reuse directly. If they fail, generate an incremental patch script (install missing packages, update versions) rather than rebuilding from scratch.

6. **Build and execute in Docker**: Write the Dockerfile combining the base image, installation script, and repository checkout. Build the image and run the installation commands. Capture all stdout/stderr output for diagnostic analysis.

7. **Verify with Fail-to-Pass criterion**: Run the test script on the *unfixed* code -- tests should fail. Apply the fix patch and run again -- tests should pass. Both conditions must hold. If either fails, proceed to error attribution.

8. **Attribute errors and generate diagnostic feedback**: Classify the failure. If the container build itself failed, identify the broken command (missing system package, incompatible version, network issue). If tests failed unexpectedly, determine whether the test command is wrong, the framework version is incompatible, or a runtime dependency is missing. Produce a structured diagnosis.

9. **Iterate with feedback**: Feed the diagnostic report back into step 2 or 3 (depending on error type). Adjust the base image, installation script, or test configuration accordingly. Retry up to a bounded number of iterations (typically 3-5 rounds).

10. **Export the verified environment**: Once F2P verification passes, output the final Dockerfile, installation script, and test script as reproducible artifacts. Tag the Docker image for future reuse by the Environment Reuse Mechanism.

## Concrete Examples

**Example 1: Python repository with complex dependencies**

User: "Set up a Docker test environment for this Python ML repo that uses PyTorch, has a Makefile, and tests with pytest."

Approach:
1. Analyze repo: find `pyproject.toml` listing torch, numpy, scipy; `Makefile` with `test` target; `.python-version` specifying 3.10
2. Select base image: `python:3.10-slim` (matches version constraint)
3. Generate install script:
```bash
apt-get update && apt-get install -y build-essential git
pip install --upgrade pip setuptools wheel
pip install -e ".[dev,test]"
# If torch install fails due to CUDA, fall back to CPU-only:
pip install torch --index-url https://download.pytorch.org/whl/cpu
```
4. Generate test script:
```bash
cd /workspace && python -m pytest tests/ -x --tb=short
```
5. Build Docker image, run tests on buggy commit (expect failure), apply patch, run tests again (expect pass)
6. First iteration fails: `ModuleNotFoundError: No module named 'scipy.special'` -- error attribution identifies scipy was not installed because `pyproject.toml` lists it under `[project.optional-dependencies]`
7. Patch install script: add `pip install -e ".[dev,test,scientific]"` -- retry succeeds

Output:
```dockerfile
FROM python:3.10-slim
RUN apt-get update && apt-get install -y build-essential git
WORKDIR /workspace
COPY . .
RUN pip install --upgrade pip setuptools wheel
RUN pip install torch --index-url https://download.pytorch.org/whl/cpu
RUN pip install -e ".[dev,test,scientific]"
CMD ["python", "-m", "pytest", "tests/", "-x", "--tb=short"]
```

**Example 2: Rust project with environment reuse**

User: "Create a test environment for this Rust CLI tool. We already had one working for v0.8, now we need v0.9."

Approach:
1. Retrieve the v0.8 Docker image from prior builds
2. Run `cargo test` inside v0.8 environment against v0.9 code
3. Tests fail with: `error[E0432]: unresolved import 'clap::Parser'` -- clap was upgraded from v3 to v4 in v0.9
4. EnvPatch strategy: generate incremental patch rather than full rebuild:
```bash
# Incremental patch for v0.8 -> v0.9
cargo update
cargo build --release 2>&1 || cargo clean && cargo build --release
```
5. Re-run tests -- pass. Environment reused successfully with minimal patching.

Output:
```dockerfile
# Reused from v0.8, patched for v0.9
FROM rust:1.75-slim
RUN apt-get update && apt-get install -y pkg-config libssl-dev
WORKDIR /workspace
COPY . .
RUN cargo build --release
CMD ["cargo", "test"]
```

**Example 3: Polyglot TypeScript + Go monorepo**

User: "Build a test environment for our monorepo -- TypeScript frontend with Jest tests and Go backend with go test."

Approach:
1. Analyze repo: `frontend/package.json` (jest, react), `backend/go.mod` (go 1.21)
2. Select base: `ubuntu:22.04` (polyglot requires both runtimes)
3. Generate install script:
```bash
# System deps
apt-get update && apt-get install -y curl git build-essential

# Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
apt-get install -y nodejs

# Go 1.21
curl -fsSL https://go.dev/dl/go1.21.6.linux-amd64.tar.gz | tar -C /usr/local -xz
export PATH=$PATH:/usr/local/go/bin

# Frontend deps
cd /workspace/frontend && npm ci

# Backend deps
cd /workspace/backend && go mod download
```
4. Generate test scripts:
```bash
# Frontend
cd /workspace/frontend && npx jest --ci --forceExit
# Backend
cd /workspace/backend && go test ./... -v -count=1
```
5. Build, verify F2P for both test suites independently
6. First attempt: Go tests fail with `cgo: C compiler not found` -- attribute to missing `gcc` in Go build step
7. Patch: add `apt-get install -y gcc` before Go installation -- retry succeeds

## Best Practices

- **Do**: Always read and parse the actual build config files (`package.json`, `Cargo.toml`, `pom.xml`, etc.) before generating installation scripts. Never guess dependencies from file extensions alone.
- **Do**: Use the lightest base image that satisfies requirements. Start with language-specific slim images; only escalate to `ubuntu` for polyglot or complex system-dependency cases.
- **Do**: Separate the installation script from the test script. They serve different purposes and fail for different reasons -- keeping them separate enables precise error attribution.
- **Do**: When a build fails, classify the error before retrying. Distinguish between "wrong base image" (needs replanning), "missing apt package" (needs script patch), and "wrong test command" (needs test reconfiguration). Each requires a different fix.
- **Avoid**: Blindly retrying the same commands after failure. Each iteration must incorporate diagnostic feedback from the previous failure -- otherwise you waste retry budget on the same error.
- **Avoid**: Installing unnecessary packages "just in case." Every extra package increases image size and attack surface. Add packages only when a specific error demands them.
- **Avoid**: Hardcoding language versions without checking the repository's version constraints. Always read `.python-version`, `rust-toolchain.toml`, `.nvmrc`, `.go-version`, or equivalent files first.

## Error Handling

| Error Type | Diagnosis | Resolution |
|---|---|---|
| Base image incompatibility | Build fails immediately with architecture or OS-level errors | Switch base image (e.g., from alpine to debian if musl causes issues) |
| Missing system package | `command not found` or linker errors during build | Add the specific `apt-get install` for the missing tool/library |
| Dependency version conflict | Package manager reports incompatible version constraints | Pin versions explicitly, or update the dependency resolution strategy |
| Test framework not found | `command not found` for pytest/jest/cargo etc. | Ensure dev dependencies are installed (e.g., `pip install -e ".[test]"`, `npm ci` including devDependencies) |
| Network timeout during build | Package download fails | Add retry logic to download commands, or pre-cache packages in the base image |
| Flaky test pass on buggy code | F2P criterion violated -- tests pass when they should fail | Verify the correct commit is checked out; ensure the test subset actually covers the bug |
| Tests fail on fixed code | F2P criterion violated -- fix doesn't resolve the issue | Re-examine the test configuration; the test command may be targeting the wrong test files |

When the maximum retry count is reached without success, output the last diagnostic report with the specific failure classification so the user can make an informed manual intervention.

## Limitations

- This approach requires Docker access. It cannot construct environments in sandboxed settings where container creation is restricted.
- Environment reuse depends on having prior builds for the same or similar repository. For completely novel projects with no history, every build starts from scratch.
- The Fail-to-Pass criterion requires both a buggy version and a fix patch. For general "make this repo testable" tasks without a specific bug/fix pair, skip F2P verification and just verify that tests pass.
- Extremely large repositories (>1GB) or those with proprietary dependencies not available in public registries may require manual intervention for authentication or artifact staging.
- Languages with non-standard build systems (custom Makefiles with undocumented targets, Bazel with complex workspace rules) are harder to auto-configure and may need more iterations.
- The technique optimizes for test-based verification. Projects without test suites cannot be verified and benefit less from this approach.

## Reference

**Paper**: [MEnvAgent: Scalable Polyglot Environment Construction for Verifiable Software Engineering](https://arxiv.org/abs/2601.22859v2) (Guo et al., 2026)
**Key insight**: Treat Docker environment construction as a multi-agent closed-loop problem with explicit error attribution and environment reuse, not a one-shot Dockerfile generation task. Look for Algorithm 1 (the full PEV loop), the Environment Reuse Mechanism (Section 3.2), and the error attribution taxonomy in the Verification stage.
**Code & Data**: [github.com/ernie-research/MEnvAgent](https://github.com/ernie-research/MEnvAgent) | Dataset: `ernie-research/MEnvData-SWE` on Hugging Face