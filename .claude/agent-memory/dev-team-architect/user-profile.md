---
name: user-profile
description: Who the user is for the slm repo — solo learner building a summarization SLM end to end; what that implies for planning
metadata:
  type: user
---

Single user, personal learning project: a ~0.5B summarization SLM, full pipeline (corpus ->
Postgres -> pretrain/SFT/RL with judge rewards -> eval -> serving). "The understanding is the
goal": prefers owning a mechanism over adopting a service when the cost is similar (see D2
recommendation in docs/decisions.md). Develops on Apple Silicon, trains on rented cloud GPUs;
budget-conscious (API < $50/mo). Had a predecessor project (summ-slm, trl GRPO, custom HF
model) that the brief explicitly does not carry forward.

**How to apply:** favour fewer systems and explain-able schemas in recommendations; keep the
Mac smoke path working for every package (no CUDA-only dependencies in core paths).
