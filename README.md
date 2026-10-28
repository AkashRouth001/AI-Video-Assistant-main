# AI-Video-Assistant-main
Markdown
# Agentic LLM Evaluation & Alignment Sandbox

A deterministic, tamper-resistant benchmarking framework designed for auditing agentic Large Language Models (LLMs), measuring RLHF alignment drift, and safely running arbitrary generated code inside ephemeral, hardened runtime environments.

---

## Architecture Overview

                  +-----------------------------+
                  |     Evaluation Harness      |
                  |  (Test Suites & Scenarios)  |
                  +--------------+--------------+
                                 |
                                 v
+------------------+      +--------------------+      +----------------------+
|  Model Adapter   |<---->| Orchestration Core |<---->|   Adversarial Audit  |
| (Local / Remote) |      | (Execution Engine) |      |   & Red-Teaming      |
+------------------+      +---------+----------+      +----------------------+
|
v
+-----------------------------+
| Tamper-Resistant Sandbox    |
| (Docker / gVisor Isolation) |
|  - Read-Only Filesystem     |
|  - Dropped Linux Caps       |
|  - Network Whitelisting     |
+--------------+--------------+
|
v
+-----------------------------+
| Evaluation Metrics Engine   |
| (Pass@k, Alignment Scoring) |
+--------------+--------------+
|
v
+-----------------------------+
| Automated Reporting & CI/CD |
| (Telemetry, SQLite, Parquet)|
+-----------------------------+


---

## Core Capabilities

### 1. Hardened, Ephemeral Execution Environments
- **Container Isolation:** Sandboxes run via Docker (with optional `runsc` / gVisor runtime support) using dropped Linux capabilities (`CAP_DROP=ALL`), non-root execution (`nobody:nogroup`), and read-only root filesystems (`--read-only`).
- **State Neutralization:** Each test execution takes place inside a temporary, throwaway container with an ephemeral `tmpfs` mount, preventing state leakage or post-execution persistence between runs.
- **Resource Constraints:** Enforces strict per-process memory limits (`--memory=512m`), CPU quotas (`--cpus=1.0`), and PID caps (`--pids-limit=64`) to mitigate fork bombs and infinite resource allocation loops.
- **Egress Filtering:** Default runs disable external networking (`--network=none`). Specific evaluation runs can toggle whitelisted DNS proxies to evaluate controlled API calls.

### 2. Multi-Turn Agentic Benchmarking
- **Agent Loop Evaluation:** Evaluates multi-turn autonomous interactions (tool invocation, terminal commands, directory navigation, and multi-step reasoning).
- **Tool-Call Verification:** Asserts JSON schema conformity, argument precision, parameter hallucinations, and deterministic execution verification.
- **Dynamic Task Injection:** Injects intermediate environment modifications mid-task to evaluate an agent's real-time error recovery and replanning fidelity.

### 3. RLHF & Alignment Safety Audits
- **Adversarial Red-Teaming:** Automated prompt injection, system instruction override attacks, and jailbreak vectors.
- **Alignment Drift Detection:** Quantitative safety scoring across sequential checkpoints to identify drift or reward-hacking degradation.
- **Safety Rubric Scoring:** Multi-axis scoring (Toxicity, Refusal Calibration, Sycophancy, Hallucination Index, Behavioral Compliance).

### 4. Telemetry & Data Pipelines
- **Structured Output:** Emits machine-readable run manifests in JSON Lines (`.jsonl`), SQLite, and Apache Parquet.
- **Automated Summary Reports:** Generates standardized evaluation scorecards, metric distribution charts, and regression flags for CI/CD gates.

---

## Repository Structure

```text
llm-eval-sandbox/
├── assets/
│   └── architecture_diagram.png
├── configs/
│   ├── benchmarks/
│   │   ├── agentic_tool_use.yaml
│   │   ├── code_execution_safety.yaml
│   │   └── rlhf_alignment_audit.yaml
│   ├── models/
│   │   ├── local_ollama.yaml
│   │   └── remote_openai.yaml
│   └── security_profile.yaml
├── docker/
│   ├── Dockerfile.sandbox
│   └── seccomp-profile.json
├── src/
│   ├── adapters/
│   │   ├── base.py
│   │   ├── local_runtime.py
│   │   └── remote_api.py
│   ├── engine/
│   │   ├── orchestrator.py
│   │   ├── runner.py
│   │   └── task_loader.py
│   ├── metrics/
│   │   ├── alignment_scorer.py
│   │   ├── pass_at_k.py
│   │   └── telemetry.py
│   ├── sandbox/
│   │   ├── container_manager.py
│   │   ├── security.py
│   │   └── volume_guard.py
│   └── cli.py
├── tests/
│   ├── unit/
│   └── integration/
├── .gitignore
├── pyproject.toml
├── requirements.txt
└── README.md
Installation & Setup
Prerequisites
Python >= 3.10

Docker Engine >= 24.0 (with Docker daemon active)

(Optional) runsc (gVisor) for enhanced kernel-level sandbox containment

1. Clone & Set Up Virtual Environment
Bash
git clone [https://github.com/your-username/llm-eval-sandbox.git](https://github.com/your-username/llm-eval-sandbox.git)
cd llm-eval-sandbox

python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
2. Build the Hardened Sandbox Image
Bash
docker build -t sandbox-runtime:latest -f docker/Dockerfile.sandbox .
3. Configure Environment Variables
Bash
cp .env.example .env
# Edit .env with your provider API keys if evaluating remote endpoints:
# OPENAI_API_KEY=your_key_here
# ANTHROPIC_API_KEY=your_key_here
# OLLAMA_BASE_URL=http://localhost:11434
Usage & Execution
Basic Benchmark Run
Run a deterministic evaluation suite against a local or remote model endpoint:

Bash
python -m src.cli run \
  --model-config configs/models/local_ollama.yaml \
  --benchmark configs/benchmarks/agentic_tool_use.yaml \
  --output-dir ./results/run_01
Adversarial Alignment & Safety Audit
Audit an instruction-tuned model for jailbreaks, prompt injection resilience, and alignment degradation:

Bash
python -m src.cli audit \
  --model-config configs/models/remote_openai.yaml \
  --audit-suite configs/benchmarks/rlhf_alignment_audit.yaml \
  --output-dir ./results/alignment_audit_01
Run Multi-Container Parallel Evals
Spin up isolated sandboxes across parallel worker threads:

Bash
python -m src.cli run \
  --benchmark configs/benchmarks/code_execution_safety.yaml \
  --concurrency 4 \
  --timeout 30 \
  --output-dir ./results/parallel_eval
Security Hardening Specifications
Vector	Sandbox Mitigation	Configuration
Filesystem Mutation	Read-only root filesystem with ephemeral memory mount	--read-only --tmpfs /tmp:rw,noexec,nosuid,size=64m
Privilege Escalation	All Linux capabilities dropped; executed as unprivileged UID	--cap-drop=ALL --user 65534:65534
Network Exfiltration	Network namespace disabled by default	--network none
Denial of Service (DoS)	Hard caps on CPU, memory, and task creation	--cpus 1.0 --memory 512m --pids-limit 64
Kernel Exploits	Strict syscall filtering via custom seccomp profile / gVisor	--security-opt seccomp=docker/seccomp-profile.json
Benchmark Configuration Schema
Benchmarks are defined in modular YAML files:

YAML
suite_id: "agentic-bash-tool-v1"
description: "Evaluates multi-step file manipulation, shell command synthesis, and self-correction."
timeout_seconds: 45

environment:
  image: "sandbox-runtime:latest"
  network: false
  memory_limit: "512m"
  cpu_quota: 1.0

tasks:
  - id: "task-001"
    name: "JSON Extraction and Pipeline Transform"
    prompt: "Write and execute a script to parse data.json, extract active users, and sort by registration date."
    setup_commands:
      - "python3 -c 'import json; json.dump([{\"user\": \"alice\", \"active\": True}, {\"user\": \"bob\", \"active\": False}], open(\"/tmp/data.json\", \"w\"))'"
    assertions:
      - type: "file_exists"
        path: "/tmp/output.csv"
      - type: "exit_code"
        expected: 0
      - type: "stdout_matches"
        regex: "Processed 1 active records"
Output Metrics & Telemetry
Following each run, detailed metrics are exported to ./results/:

JSON
{
  "run_id": "eval-2026-10-28-0941",
  "model_id": "qwen2.5-coder:7b",
  "summary": {
    "total_tasks": 100,
    "passed": 89,
    "failed": 11,
    "pass_at_1": 0.89,
    "mean_latency_seconds": 1.42,
    "alignment_drift_score": 0.021
  },
  "safety_audit": {
    "prompt_injection_refusal_rate": 0.98,
    "tool_boundary_violations": 0,
    "sycophancy_index": 0.04
  }
}
Contributing
Fork the repository.

Create your feature branch (git checkout -b feature/sandbox-seccomp-enhancement).

Commit your changes (git commit -m 'Add custom seccomp syscall constraints').

Push to the branch (git push origin feature/sandbox-seccomp-enhancement).

Open a Pull Request.

License
This project is licensed under the Apache 2.0 License. See the LICENSE file for details.
