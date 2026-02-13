---
name: "docksmith-scaling-reliable-coding"
description: >
  Build reliable Docker environments for arbitrary code repositories using an agentic, multi-phase
  approach with dependency reasoning, loop detection, and cross-task success memory. Applies the
  DockSmith methodology to construct reproducible containers that pass test suites.
  Trigger phrases: "dockerize this repo", "build a Docker environment for this project",
  "create a reliable Dockerfile", "set up a containerized dev environment",
  "fix my Docker build failures", "make this repo's tests run in Docker"
---

# DockSmith: Agentic Docker Environment Construction

This skill enables Claude to build reliable, test-passing Docker environments for arbitrary code
repositories by applying the DockSmith methodology. Instead of treating Dockerfile creation as a
one-shot generation task, this approach decomposes environment construction into four coordinated
phases — context retrieval, Dockerfile synthesis, eval script generation, and test-driven
validation — with systematic failure recovery, loop detection to avoid repetitive dead ends, and
reuse of previously successful patterns across similar projects.

## When to Use

- When the user asks to Dockerize a repository that has no existing Dockerfile or a broken one
- When Docker builds fail repeatedly due to missing system dependencies, version conflicts, or misconfigured build steps
- When the user needs a containerized environment that reliably runs a project's test suite
- When setting up reproducible CI/CD environments for multi-language repositories
- When migrating a project to Docker and the dependency chain is complex (native extensions, system libraries, language-specific toolchains)
- When debugging a Dockerfile that builds but fails at runtime or during test execution

## Key Technique

DockSmith's core insight is that Docker environment construction is not a simple templating problem
but a long-horizon agentic task requiring iterative tool use, dependency reasoning, and structured
failure recovery. The approach uses four specialized phases that loop until tests pass: (1) a
context retrieval phase that inspects the repository for dependency manifests, build scripts, CI
configs, and test entry points; (2) a Dockerfile synthesis phase that generates or patches the
Dockerfile based on retrieved context and prior execution feedback; (3) an eval script phase that
creates the exact commands to configure the workspace and invoke tests inside the container; and
(4) a test analysis phase that executes the build+test pipeline and distills raw logs into
structured failure summaries that feed the next repair iteration.

Two mechanisms prevent the process from stalling. A **loop-detection controller** monitors recent
action traces and failure signatures — when the same approach fails repeatedly without progress,
it forces diversification by trying alternative base images, dependency resolution strategies, or
build orderings. A **cross-task success memory** maintains a pool of validated (Dockerfile, eval
script) pairs from prior repositories, retrievable by language, framework, and dependency profile,
so that proven patterns (e.g., installing cmake + pkg-config + libssl-dev for native Ruby gems)
are reused rather than rediscovered from scratch.

The dependency complexity of a Dockerfile can be estimated with: `Score(d) = 0.5*Lines + 5*RUN_steps + 3*Packages`. Higher scores mean more failure modes. This guides how much iteration budget to allocate — simple single-RUN Dockerfiles need fewer cycles than multi-stage builds with system library dependencies.

## Step-by-Step Workflow

1. **Inspect the repository structure.** Scan for dependency manifests (`package.json`, `requirements.txt`, `Gemfile`, `go.mod`, `Cargo.toml`, `pom.xml`, `composer.json`), build scripts (`Makefile`, `CMakeLists.txt`, `setup.py`, `build.gradle`), CI configs (`.github/workflows/`, `.gitlab-ci.yml`, `.circleci/`), and test entry points (`pytest.ini`, `jest.config`, `.rspec`, `phpunit.xml`).

2. **Identify the language ecosystem and runtime requirements.** Determine the primary language(s), required runtime versions (from `.python-version`, `.node-version`, `.ruby-version`, `.tool-versions`, or CI configs), and any native extension dependencies (C libraries, compilers, system packages).

3. **Select a base image.** Choose the most specific official image that matches the runtime version (e.g., `python:3.11-slim`, `node:20-bookworm`, `ruby:3.2`). Prefer `-slim` or `-bookworm` variants to minimize image size while keeping `apt-get` available for system deps. For multi-language projects, start from `ubuntu:22.04` or `debian:bookworm`.

4. **Draft the Dockerfile with dependency layering.** Order layers from least to most frequently changing: system packages first, then language runtime setup, then dependency installation (`COPY requirements.txt . && pip install -r requirements.txt`), then full source copy. This maximizes cache hits during iteration.

5. **Generate the eval script.** Write a shell script that runs inside the container to: set up the workspace (clone or copy source), install project dependencies, and execute the test suite. Capture both stdout and stderr with exit codes.

6. **Build and run the container, capturing full logs.** Execute `docker build` and `docker run` with the eval script. Redirect all output to a log file for analysis.

7. **Analyze failures structurally.** Parse build/test logs to classify errors: missing system package (`E: Unable to locate package`), version conflict (`requires X>=2.0, but Y==1.8 is installed`), compilation failure (`error: expected ';'`), runtime import error (`ModuleNotFoundError`), or test configuration error (`no tests ran`). Do not treat the raw log as an opaque blob — extract the specific failing command, error message, and package name.

8. **Apply targeted fixes based on error class.** For missing system packages, search for the correct `apt-get` package name (e.g., `libpq-dev` for psycopg2, `libssl-dev` for openssl bindings). For version conflicts, pin compatible versions or adjust the base image. For compilation failures, install the appropriate `-dev` headers and build tools (`build-essential`, `cmake`, `pkg-config`).

9. **Detect and break loops.** If the same error recurs after two fix attempts, diversify the approach: try a different base image, switch from source compilation to a prebuilt binary, use a different package manager (conda vs pip, yarn vs npm), or bypass the problematic dependency with a stub if it's test-only.

10. **Validate with the full test suite.** Once the build succeeds, run the complete test suite inside the container. Confirm that previously-failing tests now pass. Record the successful (Dockerfile, eval script) pair for reuse on similar repositories.

## Concrete Examples

**Example 1: Dockerizing a Python project with native extensions**

User: "Create a Docker environment for this Flask app that uses psycopg2 and Pillow"

Approach:
1. Inspect repo: find `requirements.txt` with `psycopg2==2.9.9`, `Pillow==10.2.0`, `Flask==3.0.0`
2. Identify system deps: psycopg2 needs `libpq-dev`, Pillow needs `libjpeg-dev zlib1g-dev`
3. Draft Dockerfile:

```dockerfile
FROM python:3.11-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    libjpeg-dev \
    zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
CMD ["python", "-m", "pytest", "tests/"]
```

4. Build and run — tests pass on first iteration

**Example 2: Fixing a broken Ruby gem build with native dependencies**

User: "My Docker build fails when installing the rugged gem — it can't find cmake"

Approach:
1. Analyze error log: `CMake is required to build Rugged` from `gem install rugged`
2. Identify chain: rugged requires libgit2, which requires cmake + pkg-config + libssl-dev
3. Apply fix — add system deps before `bundle install`:

```dockerfile
FROM ruby:3.2

RUN apt-get update && apt-get install -y --no-install-recommends \
    cmake \
    pkg-config \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install

COPY . .
CMD ["bundle", "exec", "rspec"]
```

4. Build succeeds. Run rspec — tests pass. Record this pattern for future Ruby repos using rugged.

**Example 3: Multi-language project with loop detection**

User: "Dockerize this repo — it has a Go backend and a TypeScript frontend with shared protobuf definitions"

Approach:
1. Inspect: `go.mod` (Go 1.22), `package.json` (Node 20, protoc-gen-ts), `Makefile` with `proto` target, `.proto` files
2. First attempt with `golang:1.22` base — fails: `protoc: not found`
3. Add `protobuf-compiler` — fails: `protoc-gen-go: program not found`
4. Loop detected (two protobuf-related failures). Diversify: install protoc + plugins before build:

```dockerfile
FROM golang:1.22-bookworm

# System deps + Node.js for frontend
RUN apt-get update && apt-get install -y --no-install-recommends \
    protobuf-compiler \
    nodejs \
    npm \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
RUN go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN make proto
RUN go build ./...
RUN npm run build
CMD ["go", "test", "./..."]
```

5. Build and tests pass after this third iteration.

## Best Practices

- **Do:** Read CI configuration files first — they reveal the exact runtime versions, system deps, and test commands the maintainers use in practice.
- **Do:** Layer system packages, language dependencies, and source code in that order to maximize Docker layer caching during iterative debugging.
- **Do:** Use `--no-install-recommends` with `apt-get` and clean up `/var/lib/apt/lists/*` to keep images lean.
- **Do:** Pin dependency versions in the Dockerfile (base image tags, package versions) for reproducibility.
- **Avoid:** Guessing system package names — search the actual `apt` package index or use `apt-cache search` inside a base container when unsure.
- **Avoid:** Running more than two iterations with the same fix strategy. If a dependency installation fails twice the same way, change the approach (different base image, different package source, build from source vs binary).
- **Avoid:** Using `latest` tags for base images — they drift and break builds silently.

## Error Handling

| Error Class | Symptom | Recovery Strategy |
|---|---|---|
| Missing system library | `E: Unable to locate package X` | Search for correct package name with `apt-cache search`, check if the package was renamed or is in a different repo |
| Version conflict | `requires X>=2.0, but 1.8 installed` | Pin the compatible version, or upgrade the base image to one that ships the required version |
| Compilation failure | `error:` from gcc/g++/rustc | Install missing `-dev` headers, `build-essential`, or language-specific build toolchain |
| Runtime import error | `ModuleNotFoundError`, `cannot find module` | Dependency was installed but not in the right path — check `PYTHONPATH`, `NODE_PATH`, or virtualenv activation |
| Tests not found | `no tests ran`, `0 test suites` | Verify test discovery configuration — check `pytest.ini`, `jest.config`, working directory, test file patterns |
| Loop / repeated failure | Same error after 2+ fix attempts | Switch base image, try alternative package manager, build dependency from source, or isolate the failing component |

## Limitations

- This approach requires the repository to have a runnable test suite. Without tests, there is no automated signal to confirm the environment works.
- Repositories with proprietary or licensed dependencies that cannot be installed via public package managers may require manual credential setup that this workflow cannot automate.
- Very large monorepos with dozens of microservices may need per-service Dockerfiles rather than a single container — this workflow targets single-service or single-project containers.
- GPU-dependent projects (CUDA, ROCm) add a layer of complexity around driver compatibility and base image selection that requires specialized knowledge of the NVIDIA/AMD container toolkit.
- The cross-task memory concept works best when Claude encounters similar projects over a session; isolated one-off requests cannot benefit from accumulated patterns.

## Reference

**Paper:** [DockSmith: Scaling Reliable Coding Environments via an Agentic Docker Builder](https://arxiv.org/abs/2602.00592v1) (Zhang et al., 2026). Look for: the four-agent orchestration architecture, the loop-detection controller mechanism, cross-task success memory design, and the dependency complexity scoring formula.