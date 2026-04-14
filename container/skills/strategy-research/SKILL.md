---
name: strategy-research
description: Run one iteration of the strategy research loop — assess current performance, identify a weakness, propose and test one improvement, validate across multiple time windows, and report results. Use when the user says "research", "improve strategy", or on scheduled research tasks.
---

# /strategy-research — Strategy Improvement Loop

Run one iteration of the autonomous strategy research loop.

## Pre-flight check

Verify the quanthub codebase is mounted:

```bash
test -d /workspace/extra/quanthub/src/quanthub/strategies && echo "READY" || echo "NOT_READY"
```

If `NOT_READY`, respond:
> The quanthub codebase is not mounted. This skill requires the quanthub-research group with proper mounts configured.

Then stop.

## Step 1: Baseline Assessment

```bash
cd /workspace/extra/quanthub
git status
```

Run the current strategy on multiple windows and record baseline metrics:

```bash
cd /workspace/extra/quanthub
python -c "
import sys; sys.path.insert(0, 'src')
from quanthub.backtest import run_backtest
from quanthub.strategies import momentum

windows = [
    ('2018-01-01', '2026-01-01'),
    ('2020-01-01', '2026-01-01'),
    ('2022-01-01', '2026-01-01'),
    ('2023-01-01', '2026-01-01'),
]
print('=== BASELINE ===')
for start, end in windows:
    try:
        r = run_backtest(momentum, bt_start=start, bt_end=end)
        m = r.metrics
        print(f'{start}-{end[:4]}: Sharpe={m[\"sharpe\"]:.2f}  Ret={m[\"annual_return\"]:.1%}  DD={m[\"max_drawdown\"]:.1%}  Vol={m[\"annual_volatility\"]:.1%}  Calmar={m[\"calmar_ratio\"]:.2f}')
    except Exception as e:
        print(f'{start}-{end[:4]}: ERROR - {e}')
"
```

Record these numbers — they are your comparison target.

## Step 2: Identify Weakness

Analyze the baseline results. Look for:
- Which window has the worst Sharpe? Why?
- Is max DD too high in certain periods?
- Is volatility disproportionate to returns?
- Are there periods where the strategy is essentially flat?

Read the research log for past experiments:
```bash
cat /workspace/group/research-log.md 2>/dev/null || echo "No prior research"
```

Choose ONE weakness to address. Do not attempt to fix everything at once.

## Step 3: Hypothesize and Implement

1. Create a research branch:
```bash
cd /workspace/extra/quanthub
git checkout -b research/$(date +%Y-%m-%d)-description
```

2. Make ONE targeted change to the strategy code.

3. The change must have a clear statistical justification (not "let's try this and see").

## Step 4: Multi-Window Validation

Run the same multi-window backtest as Step 1 with the modified code. Compare results.

Optionally run alphalens if you changed the factor:
```bash
cd /workspace/extra/quanthub
python scripts/factor_analysis.py 2>&1 | tail -30
```

## Step 5: Accept or Reject

**Accept** if:
- Sharpe improves on ≥3 of 4 windows
- Max DD does not worsen by >5pp on ANY window
- Improvement is not concentrated in one period

**Reject** otherwise. Revert the branch:
```bash
cd /workspace/extra/quanthub
git checkout zipline
git branch -D research/$(date +%Y-%m-%d)-description
```

## Step 6: Log and Report

Append to the research log:
```bash
cat >> /workspace/group/research-log.md << 'EOF'

## [DATE]: [Description]
**Hypothesis:** [why this should help]
**Change:** [what was modified]
**Results:**
| Window | Sharpe (before → after) | DD (before → after) |
| ... | ... | ... |
**Decision:** Accepted/Rejected
**Reason:** [explain]
EOF
```

Send the result summary to the user via `send_message`.
