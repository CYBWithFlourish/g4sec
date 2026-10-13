# g4sec - Gemma 4 Security

```text
✦  g4sec  ·  Gemma 4 Security
```

> Terminal-native offline AI security and compliance auditing.

`g4sec` is a local-first CLI for auditing source code and configuration files
with Gemma models - Dockerfiles, infrastructure config, logs, and security
advisories included. It combines deterministic security rules with local model
reasoning and emits structured reports suitable for both humans and automation.

## Why g4sec?

Sensitive infrastructure files and logs often cannot be pasted into hosted
AI services. `g4sec` is designed to keep audit input on the user's machine
by default:

- **Offline by default:** local audits do not call a remote API.
- **Gemma-powered:** use a locally installed Gemma model through a supported
  provider.
- **Automation-friendly:** JSON output is validated against a versioned
  report schema.
- **Defense in depth:** deterministic rules complement, constrain, and
  validate model output.
- **Explicit network boundaries:** network-backed advisory lookups require
  an explicit opt-in.

## Status

The repository is currently in the planning and foundation phase. Model
quality, benchmark values, and fine-tuning results will be published only after
they are measured on a documented, held-out evaluation set.

## Planned command examples

```bash
# Audit a local file and print a script-safe report
g4sec audit ./Dockerfile --format json

# Use a readable terminal presentation
g4sec audit ./docker-compose.yml --format table

# Look up an advisory from a local index or, with explicit opt-in, a network source
g4sec lookup CVE-2026-1234 --format json

# Inspect installed local model providers and artifacts
g4sec models list

# Print CLI and report-schema versions
g4sec version
```

The final CLI will return non-zero status when analysis fails or when findings
meet a configured severity threshold. Diagnostics will be written to stderr so
JSON stdout remains safe for pipelines.

## Architecture

```text
CLI (Typer/Rich)
      |
      v
Application orchestration
      |
      +--> Safe input loader --> redaction --> normalized evidence
      |
      +--> deterministic rules -----------+
                                          v
      +--> local Gemma provider ----> findings normalizer
                                          |
                                          v
                              Pydantic + JSON Schema validation
                                          |
                                          v
                              JSON / table / text renderers
```

The model provider is replaceable. The initial local backend is planned to
use `llama-cpp-python` with GGUF artifacts for CPU-friendly execution and
optional GPU acceleration. An Ollama provider may be added as a convenience
adapter for users who already run Ollama locally; it is not required for the
core offline runtime.

## Planned technology

- Python 3.11+
- Typer and Rich for the terminal interface
- Pydantic v2 and JSON Schema for report contracts
- `llama-cpp-python` with GGUF for local Gemma inference
- Optional Ollama provider
- pytest, Ruff, mypy, and pre-commit
- Optional local fine-tuning tooling (PEFT/TRL, bitsandbytes) under `train/`
- GitHub Actions for tests and quality checks without model weights

## Installation

Installation will be provided once the first vertical CLI slice is complete.
The intended developer setup is:

```bash
git clone https://github.com/CYBWithFlourish/g4sec.git
cd g4sec
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
```

Model weights are not committed to this repository. A future model setup
command will document how to install a compatible Gemma artifact and verify
its checksum.

## Local model providers

The runtime will expose a small provider interface so the audit pipeline can
work with:

1. A local GGUF Gemma provider.
2. An optional Ollama provider.
3. A deterministic fake provider for tests.
4. A rules-only fallback when no model is available.

The CLI has no cloud dependency and makes no network calls during an audit.
The optional training workflow runs locally on user-controlled hardware and
must never receive user audit data without an explicit, deliberate export
step.

## Report format

Reports will include a versioned contract with fields such as:

- Target and report metadata
- Risk score from 0 to 100
- Executive summary
- Finding identifier, severity, and confidence
- Evidence and source locations
- Impact and remediation
- Optional CWE/CVE references
- Model, adapter, schema, and provenance metadata

Example shape:

```json
{
  "schema_version": "1.0",
  "target": "Dockerfile",
  "risk_score": 72,
  "summary": "The image uses a mutable tag and runs as root.",
  "findings": [
    {
      "id": "container.mutable-base-tag",
      "severity": "MEDIUM",
      "confidence": 0.98,
      "title": "Mutable base image tag",
      "description": "The base image is not pinned to a digest.",
      "evidence": [{"line": 1, "text": "FROM python:3.11"}],
      "remediation": "Pin the image to a reviewed digest."
    }
  ]
}
```

The example is illustrative; the published schema is authoritative once
Milestone 0 is implemented.

## Fine-tuning and evaluation

The optional fine-tuning workflow trains a LoRA/QLoRA adapter on a Gemma
checkpoint using curated security examples, entirely on local hardware:

```text
dataset revision
      |
      v
train/sft.py --> local LoRA/QLoRA adapter --> GGUF for llama.cpp
      |                              |
      +--> train/evaluate.py <-------+
                    |
                    v
       base vs fine-tuned benchmark
```

Adapters can be merged and converted to GGUF with `llama.cpp`, so the
fine-tuned model runs through the same offline provider.

The evaluation harness will measure schema adherence, finding precision and
recall, severity agreement, evidence quality, remediation completeness,
latency, and output length. Claims such as “99% schema adherence” will not be
made until they are reproduced on a stated held-out test set.

Training data must be synthetic, redacted, or appropriately licensed. Do not
include private logs, credentials, customer configurations, or unreviewed
security data.

## Development

The planned project layout is:

```text
g4sec/
├── src/g4sec/              # CLI, engine, parsers, schemas, prompts
├── tests/                  # unit, integration, and golden fixtures
├── dataset/                # documented JSONL examples and generators
├── train/                  # optional local fine-tuning and evaluation scripts
├── schemas/                # versioned JSON report schemas
├── models/                 # ignored local model artifacts
├── docs/                   # setup and benchmark documentation
├── pyproject.toml
└── README.md
```

Run the planned checks with:

```bash
pytest
ruff check .
ruff format --check .
mypy .
```

These commands become active as each corresponding project component is
implemented.

## Privacy and security boundaries

- Audit input is local and offline by default.
- Paths, file size, recursion depth, and binary input are validated.
- Likely secrets are redacted before model prompting and display.
- Model output is treated as untrusted data and schema-validated.
- Advisory network access is explicit and provenance is recorded.
- Model weights, API keys, credentials, and private datasets are excluded
  from version control.

`g4sec` is an audit assistant, not a replacement for professional security
review, vendor advisories, or incident response.

## Roadmap

- [ ] Establish Python package, CLI entry point, and CI
- [ ] Define report schema and exit-code contract
- [ ] Ship a Dockerfile/Compose audit vertical slice
- [ ] Add local Gemma inference and rules-only fallback
- [ ] Add dataset generation and reproducible evaluation
- [ ] Run an optional local Gemma fine-tuning experiment
- [ ] Publish measured benchmark results and an offline demo

## License

The project license will be added before the first public release. Third-party
models, datasets, and runtime components remain subject to their own licenses.
