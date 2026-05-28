<div align="center">

# AI Code Self-Improver 🔄

**Generator → Critic → Fixer → Verifier Loop — One Command Code Improvement via Local LLMs**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![GitHub Stars](https://img.shields.io/github/stars/nerudek/ai-code-self-improver?style=flat-square)](https://github.com/nerudek/ai-code-self-improver/stargazers)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/nerudek/ai-code-self-improver/pulls)

An iterative AI critic loop — write, review, fix, verify in one command. Powered entirely by local LLMs via LM Studio. No cloud API keys, no subscription, no data leaving your machine.

</div>

---

## Table of Contents

- [Problem Statement](#1-problem-statement)
- [Solution Overview](#2-solution-overview)
- [Architecture](#3-architecture)
- [Quick Start](#4-quick-start)
- [CLI Usage](#5-cli-usage)
- [Python API](#6-python-api)
- [Use Cases](#7-use-cases)
- [Persistence Mode (Classic Ralph)](#8-persistence-mode-classic-ralph)
- [Best Practices & Pitfalls](#9-best-practices--pitfalls)
- [FAQ](#10-faq)
- [Contributing & Support](#11-contributing--support)

---

## 1. Problem Statement

Writing high-quality code is hard. Getting feedback from an AI reviewer requires multiple manual steps: paste code, review, suggest, fix, re-review, verify. Each cycle costs tokens, time, and attention — and if you're using a cloud API, each cycle also costs money and sends your code to a third party.

The common approaches each have trade-offs:

| Approach | Problem |
|----------|---------|
| **Manual self-review** | Humans miss bugs, especially in unfamiliar code |
| **Cloud AI review** | Costs money, sends code off-device, requires internet |
| **Rule-based linters** | Catch style issues but miss logic bugs, security holes, optimization opportunities |
| **Manual multi-step AI workflow** | Tedious copy-paste between agent and editor |

You write code, you get a review, you fix, you re-review — all as separate manual steps. **This should be one command.**

## 2. Solution Overview

AI Code Self-Improver (codename: *Ralph Wiggum Loop*) automates the entire improvement cycle in a single command:

| Phase | Component | What It Does |
|-------|-----------|-------------|
| **1. Generator** | `scripts/generator.py` | Creates initial output (code, text, analysis) using a local LLM |
| **2. Critic** | `scripts/critic.py` | Analyzes for bugs, security flaws, performance issues, style problems, documentation gaps |
| **3. Fixer** | `scripts/ralph-loop.sh` | Sends code + issue list to LLM and applies fixes |
| **4. Verifier** | `scripts/ralph-loop.sh` | Checks fixes compile and no new issues were introduced |

All powered by **local LLMs via LM Studio** (`http://127.0.0.1:1234/v1`). Zero cloud dependency. Zero data egress. Zero API costs.

## 3. Architecture

```
┌─────────────┐
│  Generator  │ → Creates initial output (code/text/analysis)
└──────┬──────┘
       ▼
┌─────────────┐
│   Critic    │ → Finds bugs, optimization, security, style, docs issues
└──────┬──────┘
       ▼
┌─────────────┐
│  Fixer      │ → Applies fixes for all reported issues
└──────┬──────┘
       ▼
┌─────────────┐
│ Verifier    │ → Checks compilation + re-reviews for new issues
└──────┬──────┘
       │
       ├─ Issues remain? → Back to Critic (max N iterations)
       ▼
   [IMPROVED OUTPUT]
```

### Data Flow

```
User input (file or inline code)
        │
        ▼
Generator creates initial output via LLM
        │
        ▼
Critic analyzes output → JSON issue list
        │
        ▼
Fixer applies fixes based on issue list
        │
        ▼
Verifier checks:
  ├── Python syntax valid? (py_compile)
  └── Re-run Critic → new issues found?
        │
        ├── If issues remain → loop back to Critic
        └── If clean → emit final output
```

### Role Prompts

Each role has a specialized system prompt:

- **Generator**: Expert programmer creating high-quality code with best practices, type hints, docstrings, error handling
- **Critic**: Merciless code reviewer checking errors, optimization, security, style, documentation, testability — returns structured JSON with severity/category/line/description/suggestion
- **Fixer**: Receives code + issue list, fixes every listed problem without introducing new functionality

## 4. Quick Start

```bash
# 1. Clone this repository
git clone https://github.com/nerudek/ai-code-self-improver.git
cd ai-code-self-improver

# 2. Ensure LM Studio is running with API enabled
curl http://127.0.0.1:1234/v1/models

# 3. Improve code from a file
./scripts/ralph-loop.sh -f examples/bad_code.py -o fixed.py -v

# 4. Improve code inline
./scripts/ralph-loop.sh -c "def hello(): print('world')"

# 5. Improve with context prompt
./scripts/ralph-loop.sh -f input.py -p "Optimize for performance"
```

## 5. CLI Usage

### `ralph-loop.sh` Options

| Flag | Description |
|------|-------------|
| `-f FILE` | Input file with code to improve |
| `-c CODE` | Inline code (instead of file) |
| `-o FILE` | Output file (default: stdout) |
| `-p PROMPT` | Additional context/prompt |
| `-i N` | Max iterations (default: 3) |
| `-m MODEL` | Model name in LM Studio |
| `-v` | Verbose — show progress |
| `--json` | Output as JSON |
| `-h` | Show help |

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `LMSTUDIO_URL` | `http://127.0.0.1:1234/v1` | LM Studio API endpoint |
| `RALPH_MODEL` | Auto-selects best available | Model name to use |
| `RALPH_MAX_ITER` | 3 | Maximum improvement iterations |

### Examples

**Refactor code from a file:**
```bash
./scripts/ralph-loop.sh -f examples/bad_code.py -o fixed.py -v
```

**Generate from scratch:**
```bash
./scripts/ralph-loop.sh -c "Write a Python quicksort function" \
  -o quicksort.py -p "Add type hints and docstrings"
```

**JSON output for programmatic use:**
```bash
./scripts/ralph-loop.sh -f input.py --json -o result.json
```

## 6. Python API

Use the modules directly in your own Python projects:

### Generator

```python
from scripts.generator import Generator

gen = Generator(model="qwen2.5-coder-32b")
code = gen.generate(
    task="Write a FizzBuzz function",
    context="Use list comprehension, add type hints"
)
```

### Critic

```python
from scripts.critic import Critic

critic = Critic()
result = critic.analyze(code)
print(f"Found {result['summary']['total_issues']} issues")

# Quick check without LLM (pattern-based only)
quick_result = critic.quick_check(code)
```

The critic returns structured JSON:

```json
{
  "issues": [
    {
      "severity": "high",
      "category": "security",
      "line": 42,
      "description": "Potential SQL injection vulnerability",
      "suggestion": "Use parameterized queries"
    }
  ],
  "summary": {
    "total_issues": 1,
    "high": 1,
    "medium": 0,
    "low": 0,
    "assessment": "Code has critical issues that must be fixed"
  }
}
```

### Full Loop Programmatically

```python
from scripts.generator import Generator
from scripts.critic import Critic

gen = Generator()
critic = Critic()

code = gen.generate("Write a secure password hasher")
issues = critic.analyze(code)

# Pipe issues back through the shell script for fixing
# or implement your own fixer loop
```

## 7. Use Cases

| Scenario | What Happens | Benefit |
|----------|-------------|---------|
| Refactor legacy code | Paste → one command → improved code | Saves hours of manual cleanup |
| Generate from scratch | Describe task → loop produces production-ready code | No back-and-forth prompting |
| Security audit | Critic finds SQL injection, hardcoded secrets, XSS | Identifies issues before deployment |
| Learning/mentoring | See what the critic flags and how fixes are applied | Learn best practices by example |
| CI pipeline | Run loop as a pre-commit hook | Catch issues before code review |
| Multi-language support | Critic supports any language; just adjust prompts | Works with Python, JS, Go, Rust, etc. |

## 8. Persistence Mode (Classic Ralph)

An alternative pattern — instead of improving quality, **keep working until the task is done**.

```
Loop structure:
1. SNAPSHOT task state
2. REVIEW what's been done
3. CONTINUE from where left off
4. DELEGATE to sub-agents
5. LONG OPERATIONS in background (build, tests)
6. VERIFY — evidence required
7. ARCHITECT REVIEW
8. DESLOP — clean up AI slop
9. REGRESSION CHECK
10. DECIDE: done → exit / not done → next iteration
```

### Completion Criteria (all must be true)

- [ ] All requirements satisfied
- [ ] Zero TODO/FIXME remaining
- [ ] Tests pass (fresh run)
- [ ] Build succeeds
- [ ] No new bugs introduced
- [ ] Architect review OK
- [ ] Regression tests OK

### When to Use Each Mode

| Situation | Mode |
|-----------|------|
| Improve quality of existing code | **Generator→Critic→Fixer→Verifier** |
| Finish a task, don't give up | **Persistence** |
| Complex multi-step task | Both: persistence wrapper + improvement loop inside |

## 9. Best Practices & Pitfalls

1. **LM Studio must be running** — the loop requires `http://127.0.0.1:1234/v1` to be available
2. **Use a capable model** — minimum 7B parameters recommended; 32B+ for best results
3. **Uncensored models work best** — models like dolphin, wizard-vicuna, or qwen3-35b-uncensored produce better critiques
4. **Low temperature for fixing** — Fixer uses temperature 0.2 for consistent, targeted repairs
5. **Set max iterations** — default 3 is good; increase for complex code, decrease for simple fixes
6. **Verbose mode for debugging** — `-v` shows each phase: what the critic found, what the fixer changed
7. **Critic supports `--quick` mode** — pattern-based checks without LLM for fast pre-screening
8. **Save examples** — `examples/` directory contains test files to verify the loop works
9. **JSON output for pipelines** — use `--json` to integrate with CI/CD or other tooling
10. **Critic checks 6 categories** — errors, optimization, security, style, documentation, testability

## 10. FAQ

**Q: Does this send my code anywhere?**
A: No. Everything runs locally through LM Studio. Zero data leaves your machine.

**Q: Do I need an API key?**
A: No. LM Studio runs local models on your hardware. No cloud subscriptions required.

**Q: What models work best?**
A: Uncensored/liberated models (dolphin, wizard-vicuna, qwen3-35b-uncensored) produce more thorough critiques. For code generation, qwen2.5-coder, deepseek-coder, or codellama are excellent.

**Q: Can I use this with non-Python code?**
A: Yes. The Critic accepts a `--language` flag and adjusts its analysis accordingly. The Fixer is language-agnostic.

**Q: What if the loop never converges?**
A: The max iteration limit (`-i`) prevents infinite loops. Default is 3. If output still has issues after max iterations, increase the limit or use a better model.

**Q: How is this different from just asking an AI to review my code?**
A: This is an automated loop that doesn't require manual copy-paste between steps. One command: review, fix, verify, repeat — all autonomous.

**Q: Can I use the Critic without the full loop?**
A: Yes. `python3 scripts/critic.py -f myfile.py` gives you a standalone review. Use `--quick` for pattern-based checks without LLM.

**Q: What if LM Studio is on a different port?**
A: Set `LMSTUDIO_URL` environment variable, or use `--api-url` flag on the Python scripts.

**Q: Can this run as a pre-commit hook?**
A: Yes. Pipe input through `ralph-loop.sh` with `--json` for automated pre-commit quality gates.

## 11. Contributing & Support

Contributions are welcome! Here's how to help:

1. **Fork** the repository
2. **Create a feature branch:** `git checkout -b feat/my-feature`
3. **Commit your changes:** `git commit -am 'feat: add support for JavaScript critic'`
4. **Push:** `git push origin feat/my-feature`
5. **Open a Pull Request**

Please ensure your changes maintain backward compatibility with the existing loop architecture (Generator → Critic → Fixer → Verifier).

---

**License:** MIT — see [LICENSE](LICENSE) for details.

**Issues:** [GitHub Issues](https://github.com/nerudek/ai-code-self-improver/issues)

**Author:** [@nerudek](https://github.com/nerudek) on GitHub

---

<div align="center">

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)

⭐ Star the repo if you find this useful!

</div>

---

See [SKILL.md](./SKILL.md) for the skill reference card (Claude Code / Hermes Agent skill manifest).
