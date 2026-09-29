# Roges Vale

**An execution runtime that lets an AI safely drive real-world actions across any tool, API, or service — without writing arbitrary code and without silently causing irreversible harm.**

> Roges — from "Roger," acknowledgement received. Vale — Latin for "farewell," historically also "go safely." Together: *acknowledged, and sent off safely.* That is the entire job of this system.

---

## What this is, in one paragraph

An AI writes a short, structured program that names which registered tools to call, with what arguments, in what order. Roges Vale validates that program before anything runs, resolves how risky each individual call actually is (not just what kind of tool it is, but what it's about to do), and only lets safe or reversible actions proceed automatically. Anything that can't be undone is held for a human to approve — or rejected outright if no human is available and no safe automated path exists. Every single call is logged.

---

## Who this document set is for

| Document | Read this if you need to... |
|---|---|
| `README.md` (this file) | Understand what the system is and get it running in 5 minutes |
| `docs/ARCHITECTURE.md` | Understand every module, how they connect, and why each decision was made — read this before modifying anything |
| `docs/API_REFERENCE.md` | Look up a specific class, function, or config option while integrating |
| `docs/AI_INTEGRATION.md` | Connect a real AI model (Claude, GPT, or anything else) — the easy way, one function |
| `docs/PLATFORM_AND_DEPLOYMENT.md` | Understand where this can run (web backend, mobile, desktop) and how to deploy it correctly |
| `docs/OPERATIONS.md` | Run this in production — monitoring, what to watch, how to respond to failures |

---

## Quickstart

```bash
pip install -e .
```

```python
from roges_vale.schema import Registry, ToolSchema, ToolType, EffectClass, ArgSpec, CliConfig
from roges_vale.compiler import Call
from roges_vale.gate import EffectGate, GateConfig
from roges_vale.dispatcher import Dispatcher, DispatchConfig
from roges_vale.sandbox import SandboxConfig
from roges_vale.engine import Engine

# 1. Register a real tool
registry = Registry()
registry.register(ToolSchema(
    name="echo",
    type=ToolType.CLI,
    args={"msg": ArgSpec(type="str", required=True)},
    default_effect=EffectClass.PURE,
    cli=CliConfig(binary="/bin/echo", arg_template="{msg}"),
))

# 2. Wire up the pipeline
gate = EffectGate(registry=registry, config=GateConfig())
dispatcher = Dispatcher(config=DispatchConfig())
engine = Engine(registry=registry, gate=gate, dispatcher=dispatcher,
                 sandbox_config=SandboxConfig())

# 3. Run a program (this is what an AI would generate)
program = [Call(id="n1", tool="echo", args={"msg": "hello"})]
results = engine.run(program)

print(results["n1"].output)  # "hello\n"
```

That's the whole loop: register what's real, describe intent as a short program, let the engine handle validation, risk, and execution.

## Connecting a real AI model (the easy way)

The example above shows the manual path — you write the `Call` objects yourself. Normally an AI writes them. Connecting one is one function call:

```python
from roges_vale.ai_integration import run_with_ai

def call_model(system_prompt: str, task: str) -> str:
    # The only model-specific code you write. Example for Claude,
    # OpenAI, and others: docs/AI_INTEGRATION.md
    response = your_model_client.generate(system_prompt, task)
    return response.text

result = run_with_ai(
    task="Resize photo.png to 800px and upload it to the CDN",
    registry=registry, engine=engine, call_model=call_model,
)

if result.success:
    print(result.results)
```

`run_with_ai` builds the prompt from your real registry, calls your model, parses and compiles its output, and automatically retries with a targeted correction prompt if the model's first attempt has a specific, fixable problem (wrong tool name, missing argument). See `docs/AI_INTEGRATION.md` for real wiring to Claude and OpenAI, and how to debug a run afterward with the full trace.

---

## Project layout

```
roges_vale/
├── roges_vale/
│   ├── schema.py            # Tool registry — the typed contract every tool satisfies
│   │                          (auto-picks pydantic or a pure-stdlib fallback — see below)
│   ├── _schema_pydantic.py    # Pydantic-backed schema implementation
│   ├── _schema_fallback.py     # Pure-stdlib schema implementation (used when pydantic unavailable, e.g. WASM)
│   ├── compiler.py          # Validates a program before anything runs
│   ├── sandbox.py           # Restricts what a running program can reach
│   ├── gate.py               # Resolves risk per call, gates irreversible actions
│   ├── dispatcher.py          # Routes a call to its real implementation
│   ├── engine.py               # Wires everything together, runs a program
│   ├── credentials.py           # Per-tenant credential resolution
│   ├── durable_state.py          # Crash-safe storage for pending approvals
│   ├── audit.py                   # Persistent, queryable log of every call
│   ├── parallel.py                 # Concurrent execution + failure policy
│   ├── pause.py                     # Stop mid-program, resume with real results
│   ├── escape_hatch.py               # Sandboxed one-off computation
│   ├── process_isolation.py           # Stronger, separate-process sandboxing
│   ├── rollback.py                     # Undo registry for failed parallel groups
│   ├── platform_capabilities.py         # Detects client vs server capabilities, enforces runs_on
│   ├── http_client.py                    # Platform-agnostic HTTP (requests server-side, browser fetch under WASM)
│   ├── volume_risk.py                     # Catches many small "safe" actions going wrong at scale
│   ├── prompting.py                        # Generates the AI-facing prompt from the live registry
│   ├── parse_program.py                     # Parses the AI's text output into real Call objects
│   ├── tracing.py                            # Reconstructs why/where/when any run failed, per-run and cross-run
│   └── ai_integration.py                      # ONE function connecting any AI model to the full pipeline
├── docs/                                # Full documentation (this directory)
├── test_*.py                             # Test suites — run these after any change
│                                            (test_real_api.py hits a live external
│                                            API and is run separately from the rest)
├── pyproject.toml                         # Package metadata
└── .github/workflows/test.yml              # CI
```

---

## Does this work in a web app, mobile app, or desktop app?

Short answer: **yes, everywhere, but not by being copied into those apps directly.** See `docs/PLATFORM_AND_DEPLOYMENT.md` for the full explanation — it matters enough to get right rather than answered in a sentence here.
# TOOL-Execution
