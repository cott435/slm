---
name: external-facts-ml-stack
description: Dated external facts checked for the slm repo contract — trl PPOTrainer removal (GRPO is the RL path), vLLM Python support, local venv version
metadata:
  type: reference
---

Checked 2026-09-28 by web search while planning the repo:

- `trl` deleted `PPOTrainer` (and `PPOConfig`, `modeling_value_head.py`) in v1.13.0; it had
  earlier been moved to `trl.experimental.ppo`. `GRPOTrainer` is the maintained path, so the
  brief's "PPO section" (RL from judge scores) is planned on GRPO. Sources:
  https://github.com/huggingface/trl/releases, https://pypi.org/project/trl/
- vLLM supports Python 3.10–3.14 (3.14 from vLLM 0.20.0); ROCm wheels 3.12 only (irrelevant,
  rented NVIDIA). https://docs.vllm.ai/en/stable/getting_started/installation/gpu/
- The repo's `.venv` is CPython 3.13.1, uv 0.11.27. The user's auto-memory pins 3.11 for a
  *retired* predecessor project (summ-slm); the brief says start clean, so 3.11 is not binding.

**How to apply:** do not re-search these on a re-run within a few months; re-check trl before
freezing a training-package signature.
