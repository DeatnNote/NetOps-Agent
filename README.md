# NetOps Agent: an evaluated agentic system for network troubleshooting

A LangGraph ReACT agent that diagnoses network incidents (Wi-Fi connectivity, slow
networks, web access failures) by calling tools served over a real **MCP (Model Context
Protocol)** server. It routes each complaint to a **SKILL.md playbook** (procedural
memory), reorders its checks using **similar past incidents** (episodic memory),
**self-corrects** on tool errors and inconclusive evidence, **verifies** findings against
independent client-side evidence, and returns a **structured diagnosis**. An evaluation
harness A/B-tests agent configurations on held-out worlds with paired significance tests.
The agent is served over **FastAPI** with liveness and readiness probes, packaged for
**Kubernetes**, and gated in **CI**.

## Architecture

```
complaint ──► plan ──► agent (model step) ◄──► tools ──► MCP server (stdio) ──► simulated network
               │            │                    │
   SKILL.md playbooks       │          errors come back as error results:
   + episodic memory        │          the model retries / broadens (self-correction)
   (hybrid retrieval)       ▼
                    submit_diagnosis (pydantic schema)  ──► diagnosis + trace (JSONL spans)
```

| Component | File |
|---|---|
| LangGraph graph: plan → agent ⇄ tools → finalize | `netops/agent.py` |
| MCP server (FastMCP, stdio), flaky-call injection | `netops/mcp_server.py` |
| Domain tools, robust-z anomaly detection | `netops/tools.py` |
| SKILL.md loader, safe rule evaluator, routing | `netops/skills.py`, `skills/*/SKILL.md` |
| Episodic memory (metadata filter + TF-IDF) | `netops/memory.py` |
| Deterministic policy + real-LLM factory | `netops/policy.py` |
| Tracing (latency, status, tokens, cost) | `netops/tracing.py` |
| Evaluation harness, A/B, significance tests | `netops/evals.py`, `netops/report.py` |
| HTTP service | `api/app.py` |
| Docker, Kubernetes, CI | `deploy/`, `.github/workflows/ci.yml` |

## What is real and what is not

Read this before citing any number from this project.

- **The network is simulated.** `netops/sim.py` generates sites with one injected fault
  each and known ground truth. On the *standard* world, faults are separable by design, so
  the 1.000 accuracy there says the harness works, not that the agent is good. The *hard*
  world, where noise crosses the diagnostic thresholds, is the meaningful test.
- **Two decision-makers were evaluated.** Most numbers use `PlaybookChatModel`, a
  deterministic policy implemented as a LangChain chat model. It reads the same system
  prompt an LLM receives and emits real tool calls through the same graph, MCP server,
  memory, tracing, and evaluation. A smaller evaluation used a real LLM, Anthropic's Claude
  (`<MODEL_NAME>`), through `--backend anthropic`: 30 incidents on the hard world, 2
  configurations, 3 trials. The OpenAI backend is implemented but has not been run.
- **The Docker image was not built** (no Docker in the build environment). The Kubernetes
  manifests were schema-validated with `kubeconform -strict` (4/4 valid) but not deployed.
- **Tracing is a custom JSONL tracer**, not LangFuse or Prometheus.
- **No thresholds were tuned on evaluation outcomes.** Diagnostic and corroboration
  thresholds come from the simulator's specified ranges. One design change was made during
  development and is documented in `netops/memory.py`: pure text retrieval was replaced
  with hybrid retrieval after a unit test showed wording drowning out site type.

## Results

Full tables, generated from the raw JSON: [`RESULTS.md`](RESULTS.md). 150 incidents per world.

| Configuration | Standard | Hard (seed 7) | Hard (seed 8) | Tool calls, hard 7 |
|---|---|---|---|---|
| Playbook only | 0.773 | 0.727 | 0.740 | 1.77 |
| + self-correction | 1.000 | 0.907 | 0.953 | 2.01 |
| + episodic memory | 1.000 | 0.960 | 0.947 | 1.65 |
| + verification | 1.000 | **0.987** | **0.980** | 2.39 |
| Full sweep (no playbook routing) + self-correction | 1.000 | 0.827 | 0.853 | 3.22 |

Findings (paired tests on the same incidents):

1. **Self-correction** (retry failed tools; broaden when the playbook finds nothing) fixed
   27–34 incidents per world with zero regressions (McNemar p ≤ 1.5e-8).
2. **Episodic memory** cut 0.30–0.35 tool calls per incident on every world
   (Wilcoxon p ≤ 2.4e-5). Its accuracy effect on hard worlds is mixed: +5.3 points on seed 7,
   −0.7 points (one incident) on seed 8.
3. **A first-match agent is brittle when noise crosses thresholds.** Checking *more*
   made it worse: the full sweep was less accurate than playbook routing on both hard worlds.
4. **Verification** (corroborate a finding with client-side failure events before
   committing) was the most accurate configuration on both hard worlds, at +0.74 tool calls.
   Per seed it is not significant (p = 0.125, p = 0.0625); pooled over both independent
   seeds it fixed 9 incidents and broke 0 (McNemar p = 0.0039).
5. **Tool-configuration A/B:** a static 50% utilization threshold produced 45–50 false
   RF-interference alarms per world (precision 0.30–0.35). The robust-z anomaly scan
   (recent window vs. the AP's own baseline) produced none, with equal recall.
6. **Known limitation:** every remaining error is a false "WAN packet loss." Lingering WAN
   loss has no client-side failure event to corroborate against, so verification cannot
   catch it.

### With a real LLM as the decision-maker

Claude (`<MODEL_NAME>`) via `--backend anthropic`, on the first 30 incidents of the hard
world (seed 7), 3 trials per incident. The deterministic policy was run on the identical
30 incidents for comparison. Raw output: `results/eval_anthropic_hard_seed7.json`.

| Configuration | Claude: mean accuracy, 3 trials | Claude: consistency across trials | Claude: tool calls (trial 1) | Deterministic policy, same 30 incidents |
|---|---|---|---|---|
| Playbook + self-correction | 0.967 (29/30 every trial, std 0) | 1.000 | 3.93 | 0.933 (1.83 tool calls) |
| + memory + verification | **0.989** (89/90 diagnoses, std 0.016) | 0.967 | 3.83 | 0.967 (2.30 tool calls) |

Consistency is the share of incidents that got the same diagnosis in all 3 trials.

What this shows, and what it doesn't:

- **The LLM path works end to end.** The model chooses tools served over MCP, recovers from
  injected tool failures, verifies, and submits a schema-validated diagnosis.
- **Claude matched or beat the hand-built policy, by about one incident per configuration.**
  At 30 incidents that difference is not statistically meaningful.
- **Claude checked more before committing**, using roughly 1.7–2x the tool calls. In a
  3-incident smoke test it made 8 model calls for 13 tool calls, so it ran several checks
  per turn, where the deterministic policy runs one at a time. That's an observation from a
  small sample, not a measured result.
- **Its one consistent error under self-correction** was an offline access point diagnosed
  as DHCP exhaustion, the same mistake the deterministic policy made on that incident.
- **Not done:** the LLM was not run on the standard world, on seed 8, or on all five
  configurations, so the paired significance tests above apply to the deterministic policy only.

## Sample: a flaky-tool incident over HTTP

`POST /diagnose {"site_id": "site-0000", "complaint": "New devices won't join the network at the office"}`

```json
{
  "diagnosis": {
    "root_cause": "ap_offline",
    "confidence": 0.9,
    "evidence": ["get_ap_status: offline_count=1 (offline_count >= 1)",
                 "get_client_events: ASSOC_FAIL=37 (corroborates, >= 10)"],
    "recommended_fix": "Check the offline AP's PoE switch port and uplink; power-cycle or replace the AP."
  },
  "skill": "wifi-connectivity",
  "trace": {"model_calls": 5, "tool_calls": 4, "tool_errors": 1, "tokens_in": 0, "tokens_out": 0, "cost_usd": 0.0}
}
```

Trace spans (`NETOPS_TRACE_DIR`): `check_radius ok → get_ap_status error (timeout) →
get_ap_status ok (retry) → get_client_events ok (corroboration) → submit_diagnosis`.

## How to run

```bash
python -m venv venv && . venv/bin/activate
pip install -r requirements.txt pytest

pytest -q                                              # 25 tests, including live MCP + HTTP
python -m netops.evals --difficulty hard --seed 7      # A/B evaluation -> results/
python -m netops.report                                # regenerate RESULTS.md
uvicorn api.app:app --port 8080                        # HTTP service
python -m netops.mcp_server                            # standalone MCP server (stdio)
```

### Running the real-LLM arm (requires your API key; costs money)

```bash
pip install -r requirements-llm.txt
export ANTHROPIC_API_KEY=...  NETOPS_MODEL=<model name>
export NETOPS_PRICE_IN_PER_MTOK=<price>  NETOPS_PRICE_OUT_PER_MTOK=<price>   # optional cost tracking
# smoke test first: 3 incidents, one config
python -m netops.evals --backend anthropic --difficulty hard --n-sites 3 --configs playbook+self_correct+memory+verify
# the run reported above: 30 incidents, two configs, 3 trials
python -m netops.evals --backend anthropic --difficulty hard --n-sites 30 --configs playbook+self_correct,playbook+self_correct+memory+verify --trials 3
```

The console prints trial 1 only. The mean, spread, and consistency across trials are in
`results/eval_anthropic_hard_seed7.json` under each configuration's `trials` key.

**On Windows PowerShell:** activate with `.\venv\Scripts\Activate.ps1`, set variables with
`$env:ANTHROPIC_API_KEY = "..."` and `$env:NETOPS_MODEL = "..."`, and keep each command on
one line (a trailing `\` is not a line continuation in PowerShell). Variables and
activation last only for the current terminal window.

### Kubernetes

```bash
docker build -f deploy/Dockerfile -t netops-agent:0.1.0 .
kubectl create secret generic netops-agent-secrets --from-literal=anthropic-api-key=...   # optional
kubectl apply -f deploy/k8s/configmap.yaml -f deploy/k8s/deployment.yaml -f deploy/k8s/service.yaml
```

Readiness (`/readyz`) only passes once the MCP session is open and tools are loaded;
liveness (`/healthz`) checks the process. CI runs the tests, an evaluation with a
quality gate (`--min-accuracy 0.97`), kubeconform validation, and the image build.
