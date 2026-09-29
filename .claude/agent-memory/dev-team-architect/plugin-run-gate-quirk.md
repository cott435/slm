---
name: plugin-run-gate-quirk
description: status.py --run-gate <pkg> FAILs with "missing contract" on a first plan-package run — that line is circular, not a blocker
metadata:
  type: project
---

`status.py --run-gate <pkg>` on a package with no contract yet returns FAIL with the single
line "missing docs/packages/<pkg>/contract.md — run /dev-team:plan-package <pkg>". On a
`/dev-team:plan-package <pkg>` run that line describes the run's own purpose.

**Why:** the gate's package check is meant for `run-package`, which needs the contract;
`plan-package` is what creates it. Blocking on it would deadlock every first plan.
Observed 2026-09-28 planning `store`; `--run-gate` with no package passed (branch + baseline).

**How to apply:** on plan-package, treat only the branch and baseline FAIL lines as blockers;
run `--run-gate` without the package argument to confirm those, and say so in the return.
