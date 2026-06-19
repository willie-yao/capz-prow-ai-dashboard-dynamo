# CAPZ Prow Dashboard (Dynamo Demo)

This repo is the **demo variant** of the [Cluster API Provider Azure
prow dashboard](https://github.com/willie-yao/capz-prow-ai-dashboard).
Same engine ([willie-yao/prow-ai-dashboard](https://github.com/willie-yao/prow-ai-dashboard)),
same prow jobs, same prompt, same evidence. Only the model and endpoint
differ: analyses here are produced by an open-weight model served
in-cluster via NVIDIA Dynamo, instead of Claude on GitHub Models.

| | Prod | Demo (this repo) |
|---|---|---|
| Dashboard | https://willie-yao.github.io/capz-prow-ai-dashboard/ | https://willie-yao.github.io/capz-prow-ai-dashboard-dynamo/ |
| Model | Claude (GitHub Models) | Kimi-K2.7-Code (in-cluster Dynamo) |
| Endpoint | `api.githubcopilot.com` | In-cluster Dynamo frontend (cluster DNS) |
| Runner | `ubuntu-latest` (GitHub-hosted) | `arc-qwen-runner` (self-hosted, in cluster) |
| Tool calling | Native | Native (vLLM backend) |

The served model is whatever Dynamo deployment the `AI_ENDPOINT` /
`AI_MODEL` repo variables point at. It currently targets the
`kimi-k27-code-agg` deployment (Kimi-K2.7-Code), a reasoning model, but
the repo is intentionally generic so the team can swap in any
OpenAI-compatible in-cluster endpoint without touching `project.yaml`.

## Why this repo exists

The engine's universal AI path lets any OpenAI-compatible chat-completions
endpoint serve the same agentic evidence-gathering loop, so this
deployment validates two things at once:

1. **Engine portability**: same code, same prompts, swap the endpoint
   and model, get useful summaries on real CAPZ failures.
2. **Cluster-internal AI plumbing**: proves end-to-end that a private
   model served from your own cluster can drive the dashboard without any
   data leaving the cluster.

The prod dashboard stays on Claude (high signal) so this experimental
variant doesn't risk the trustworthy production signal.

## Engine pinning

Unlike the other consumers, this repo pins the engine to `@main` rather
than a release tag. The slow Kimi reasoning endpoint needs the
per-request HTTP timeout fix (no fixed 60s cap) that landed on main after
`v1.0.0-beta.1`. Re-pin to the next release once it ships with that fix.

## Setup

One-time, before the first deploy succeeds:

1. **Install actions-runner-controller (ARC)** in the demo AKS cluster
   with a runner scale set named `arc-qwen-runner`. Runner pods need
   cluster DNS access to the Dynamo frontend service in the `default`
   namespace. See
   [prow-ai-dashboard/docs/self-hosted-runner-in-cluster.md](https://github.com/willie-yao/prow-ai-dashboard/blob/main/docs/self-hosted-runner-in-cluster.md).

2. **Set repo variables** to the in-cluster Dynamo endpoint and model:
   ```bash
   gh variable set AI_ENDPOINT \
     -b "http://kimi-k27-code-agg-frontend.default.svc:8000/v1/chat/completions"
   gh variable set AI_MODEL -b "moonshotai/Kimi-K2.7-Code"
   ```

3. **Set `AI_TOKEN` secret** to any non-empty string. Dynamo doesn't
   require an API key, but the engine asserts the secret is set.
   ```bash
   gh secret set AI_TOKEN -b "dynamo"
   ```

## What stays in lockstep with the prod CAPZ dashboard

The whole point of this repo is to be an **A/B variant**, not a fork.
Drift kills the comparison.

Keep identical to https://github.com/willie-yao/capz-prow-ai-dashboard :

- `prompts/system.md` (project-specific knowledge)
- `project.yaml`: everything except `id`, `branding`, the AI
  endpoint/model (which come from repo variables anyway), and the
  agentic tuning that compensates for a slower self-hosted model
  (`concurrency`, `timeout`, `max_iters`).

If the prod dashboard changes its prompt or evidence config, mirror
the change here within a few days so the comparison stays meaningful.
