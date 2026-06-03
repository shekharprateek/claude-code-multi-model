# Claude Code Multi-Model

[![License: MIT-0](https://img.shields.io/badge/License-MIT--0-yellow.svg)](LICENSE)
[![Bedrock](https://img.shields.io/badge/Amazon%20Bedrock-Mantle-blue)](https://docs.aws.amazon.com/bedrock/latest/userguide/models-endpoint-availability.html)
[![Models: 43+](https://img.shields.io/badge/Models-43%2B%20from%2012%20providers-orange)](./)

> **This is sample code intended for demonstration and learning purposes only.**
> It is not meant for production use. Review and harden all scripts, configurations,
> and IAM permissions before using in any production or sensitive environment.

## The Problem: AI Coding Agents Are Expensive at Scale

Enterprise spending on generative AI hit **$13.8 billion in 2024** — a 6x increase from $2.3B the year before ([Menlo Ventures](https://menlovc.com/2024-the-state-of-generative-ai-in-the-enterprise/)). A significant portion goes to LLM inference costs powering coding assistants, chat agents, and autonomous workflows.

The economics are stark:

- **Frontier models cost 10-100x more** than budget alternatives ($3-15/M tokens vs $0.15-0.60/M tokens)
- **AI coding agents are token-hungry** — a single complex task session can consume 100K-500K+ tokens with tool use, multi-file edits, and iterative reasoning
- **Not every task needs a frontier model** — bug fixes, test generation, and boilerplate don't require the same reasoning power as architecture decisions
- **44% of enterprises cite price as a motivation for switching LLMs** ([Menlo Ventures](https://menlovc.com/2024-the-state-of-generative-ai-in-the-enterprise/))

Research confirms that intelligent model routing dramatically reduces costs without sacrificing quality:

- [FrugalGPT](https://arxiv.org/abs/2305.05176) (Stanford) — matches GPT-4 performance with up to **98% cost reduction** through LLM cascades
- [RouteLLM](https://arxiv.org/abs/2406.18665) (UC Berkeley) — reduces costs by **over 2x** without compromising response quality
- [Hybrid LLM](https://arxiv.org/abs/2404.14618) (ICLR 2024) — **40% fewer calls** to the expensive model with no quality drop

## This Solution: Claude Code with Any Model

Run [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with **any foundation model** — not just Anthropic models. Route routine tasks to models that cost 5-20x less, reserve frontier models for complex reasoning. Choose your deployment path:

| Path | Models | Cost Model | Best For |
|------|--------|------------|----------|
| [**Bedrock (Mantle)**](bedrock/) | 43 models from 12 providers | Pay-per-token | Teams wanting model variety + zero infrastructure |
| [**Self-Hosted (EC2)**](self-hosted/) | Any Ollama/vLLM model | Fixed hourly GPU cost | Data sovereignty, air-gapped environments, unlimited tokens |

```
Task Complexity        Recommended Model         Cost vs Sonnet
────────────────       ─────────────────         ──────────────
Simple bug fixes       Qwen Coder 30B            20x cheaper
Test generation        Kimi K2.5                  5x cheaper
Feature additions      Qwen Coder Next           10x cheaper
Complex refactoring    Claude Sonnet             baseline
Architecture decisions Claude Opus               frontier
```

## Benchmark Results

We evaluated 5 models across 5 real-world coding tasks (bug fix, test writing, feature addition, refactoring, circular import resolution). Each model runs Claude Code with full tool access and is scored by both deterministic verifiers (pytest) and an LLM-as-judge (Claude Opus evaluating the actual generated code).

| Model | Input $/M | Output $/M | Pass Rate | Quality (1-5) | Avg Latency | Cost Efficiency |
|-------|-----------|------------|-----------|---------------|-------------|-----------------|
| **claude-sonnet** | $3.00 | $15.00 | **100%** | **4.5** | 35s | baseline |
| **qwen-coder-30b** | $0.15 | $0.62 | 80% | **4.2** | 129s | 20x cheaper, 93% quality |
| **kimi-k2.5** | $0.60 | $2.50 | 80% | **4.1** | 94s | 5x cheaper, 91% quality |
| **qwen-coder-next** | $0.30 | $1.20 | 80% | **4.0** | 140s | 10x cheaper, 89% quality |
| **deepseek-v3** | $0.50 | $2.00 | 60% | **3.2** | 155s | 6x cheaper, 71% quality |

> Full methodology, per-task breakdown, and how to run the benchmark yourself: [bedrock/README.md](bedrock/README.md#benchmark-results)

## Architecture

```text
                         ┌─────────────────────────────┐
                         │      Claude Code CLI        │
                         │  (Anthropic Messages API)   │
                         └──────────┬──────────────────┘
                                    │
                 ┌──────────────────┼──────────────────┐
                 │                  │                   │
         ┌───────▼──────┐  ┌───────▼──────┐  ┌────────▼─────────┐
         │ Native Path  │  │ LiteLLM      │  │ LiteLLM          │
         │ (no proxy)   │  │ Proxy        │  │ Proxy            │
         │              │  │ → Bedrock    │  │ → Self-Hosted    │
         │ Claude Opus  │  │   Mantle     │  │   (Ollama/vLLM)  │
         │ Claude Sonnet│  │              │  │                  │
         │ Claude Haiku │  │ 38 models    │  │ Any GGUF/HF      │
         └──────┬───────┘  │ 12 providers │  │ model on GPU     │
                │          └──────┬───────┘  └────────┬─────────┘
                │                 │                    │
         ┌──────▼───────┐  ┌─────▼────────┐  ┌───────▼─────────┐
         │ Amazon       │  │ Bedrock      │  │ EC2 GPU         │
         │ Bedrock      │  │ Mantle       │  │ Instance        │
         │ (Anthropic)  │  │ (us-east-1)  │  │ (your VPC)      │
         └──────────────┘  └──────────────┘  └─────────────────┘
```

## Benchmark

We measured model quality on the public [HumanEval](https://github.com/openai/human-eval)
benchmark (164 tasks), driving each task through Claude Code backed by each model
and scoring with standard `pass@1`:

| Model | pass@1 |
| --- | --- |
| Claude Sonnet 4.6 | 97.6% |
| Kimi K2.5 | 96.3% |
| DeepSeek V3 | 94.5% |
| Qwen Coder Next | 91.5% |
| Qwen Coder 30B | 90.9% |

Budget models reach 93–99% of the frontier model's pass rate. Full method,
caveats, and reproduce steps in [bedrock/README.md](bedrock/README.md#benchmark-humaneval).

## Quick Start

### Option A: Bedrock (43 models, pay-per-token)

```bash
cd bedrock

# Anthropic models — no proxy needed
./scripts/claude-model.sh --model claude-sonnet

# Third-party models — start proxy first
./scripts/setup-proxy.sh
./scripts/claude-model.sh --model qwen-coder-next
./scripts/claude-model.sh --model kimi-k2.5
./scripts/claude-model.sh --model deepseek-v3
```

See [bedrock/README.md](bedrock/README.md) for full setup, all 43 models, and proxy management.

### Option B: Self-Hosted on EC2 (fixed cost, data stays in VPC)

```bash
cd self-hosted

# Launch GPU instance + install Ollama + pull model
./scripts/setup.sh

# Run Claude Code with self-hosted model
./scripts/run.sh --model qwen3.5:35b
```

See [self-hosted/README.md](self-hosted/README.md) for instance types, GPU selection, and SSH tunnel setup.

## Comparison

| | Bedrock (Mantle) | Self-Hosted (EC2) |
|---|---|---|
| **Models** | 43 from 12 providers | Any GGUF/HF model |
| **Pricing** | Per-token ($0.15-$15/M) | Per-hour ($0.84-$4.60/hr GPU) |
| **Setup time** | 5 minutes | 15-20 minutes |
| **Latency** | 16-180s per task | Depends on GPU + model size |
| **Data location** | AWS Bedrock service | Your VPC, your instance |
| **Best when** | Variable workload, model variety | Fixed workload, data sovereignty |
| **Break-even** | < ~2M tokens/hour | > ~2M tokens/hour |

## Repository Structure

```
claude-code-multi-model/
├── README.md                  ← You are here
├── LICENSE
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── SECURITY.md
├── SUPPORT.md
├── bedrock/                   ← Bedrock Mantle path (38 third-party + 5 Anthropic)
│   ├── README.md              Full Bedrock setup guide + benchmark results
│   ├── scripts/               setup-proxy.sh, claude-model.sh, mantle-token.sh
│   ├── config/                litellm-config.yaml, claude-proxy-settings.json
│   └── benchmark/             5-task evaluation suite + LLM-as-judge
└── self-hosted/               ← EC2 self-hosted path (Ollama/vLLM)
    ├── README.md              Full EC2 setup guide
    ├── SETUP-GUIDE.md         Step-by-step GPU instance provisioning
    ├── scripts/               setup.sh, run.sh, tunnel.sh
    └── config/                litellm-config.yaml, model configs
```

## See Also

- **Anthropic's "Serving a Trillion Tokens a Month"** — our multi-model routing approach implements recommendations from references [14] and [35] of this whitepaper
- [FrugalGPT](https://arxiv.org/abs/2305.05176) — Academic foundation for LLM cascade cost optimization
- [RouteLLM](https://arxiv.org/abs/2406.18665) — Dynamic model selection framework
- [Claude Code docs](https://docs.anthropic.com/en/docs/claude-code) — Official Claude Code documentation

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
