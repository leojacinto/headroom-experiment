# Headroom Experiment: ServiceNow Fluent SDK Build

A controlled experiment measuring whether the [Headroom](https://github.com/chopratejas/headroom) context compression MCP tool reduces token spend in AI-assisted ServiceNow Fluent SDK builds.

> **Headroom** (Apache 2.0, ~29.5k stars as of June 2026) is a context compression layer by Tejas Chopra (Netflix). It compresses tool outputs, file reads, logs, and structured data before they reach the model, claiming 60–95% token reduction on structured content, 40–80% on real-world mixed workloads. Install: `pip install "headroom-ai[mcp]"`. MCP server: `headroom mcp serve`.

---

## Hypothesis

Compressing large SDK documentation with Headroom before writing files keeps the working context leaner throughout a build session, reducing cache re-reads during debug iterations and lowering total token spend, without degrading output quality.

---

## Methodology

Two conditions built the same application from the same specification, using the same AI model, on the same ServiceNow instance.

| Variable | Condition A (Headroom) | Condition B (Baseline) |
|----------|----------------------|----------------------|
| Spec | Identical | Identical |
| Model | Claude Sonnet 4.6 | Claude Sonnet 4.6 |
| Instance | Shared | Shared |
| SDK | @servicenow/sdk 4.7.2 | @servicenow/sdk 4.7.2 |
| Headroom | Active: SDK explain outputs >1,500 tokens compressed before use | Disabled |
| Reference codebase | None | None |

### Application scope

A multi-artifact ServiceNow application built entirely via Fluent SDK TypeScript files (`.now.ts`), deployed via `npx @servicenow/sdk install`. Artifact count: **28**, spanning:

- 3 tables
- 5 flows (including 3 subflows)
- 2 script includes
- 4 business rules
- 1 ACL set
- 1 workspace + dashboard

### Build order (both conditions)

1. `npm install`
2. Write all `.now.ts` artifact files
3. `npx @servicenow/sdk build` (resolve TypeScript errors)
4. Authenticate explicitly via `npx @servicenow/sdk auth` with env credentials
5. `npx @servicenow/sdk install` (deploy to instance)

### Measurement

Token usage extracted from Claude Code session JSONL files (`~/.claude/projects/`). Pricing applied at Sonnet 4.6 rates:

| Token type | Rate |
|-----------|------|
| Input | $3.00 / MTok |
| Output | $15.00 / MTok |
| Cache write | $3.75 / MTok |
| Cache read | $0.30 / MTok |

---

## Results

| Metric | Condition A (Headroom) | Condition B (Baseline) |
|--------|----------------------|----------------------|
| Total cost | **$13.12** | $14.73 |
| Cost delta | **11% cheaper** | baseline |
| Cache reads (context re-reads) | **14.8M tokens** | 23.2M tokens |
| Context reduction | **36% fewer re-reads** | baseline |
| Artifacts delivered | 28 / 28 | 28 / 28 |
| Flow errors encountered | Yes | Yes (identical) |
| Output quality difference | None | None |
| Headroom compressions called | 4 | 0 |

---

## Key Findings

**Headroom reduced cost by 11% and context re-reads by 36%.**

The mechanism was not improved first-pass code quality; both conditions hit identical flow action errors and required the same debugging cycles. The benefit accrued in the debug phase: a leaner working context meant cheaper context re-reads on every iteration turn.

**Headroom front-loads spend.** Condition B was more file-efficient at the $9 mark (28 files vs 25). The compression overhead is paid upfront; the savings materialise across debug iterations.

**At this build scale, context pressure was low.** Claude Sonnet 4.6 has a 200k context window. For 28 artifacts the full SDK documentation fit without compression. The experiment operated near the lower bound of where Headroom's benefit is meaningful. Larger builds (more artifacts, more SDK topics loaded, longer debug sessions) should see a wider gap.

---

## Alignment with Headroom's benchmarks

Headroom's stated benchmarks: 60–95% token reduction on structured content (code search 92%, SRE log debugging 92%), 40–80% on real-world mixed workloads. Our result of 11% cost reduction is well below this range.

**Why:** This experiment used Headroom narrowly: the agent was instructed to compress SDK explain outputs over 1,500 tokens only. It called `headroom_compress` 4 times across the full build. Headroom is designed to compress *all* tool outputs: file reads, bash outputs, build logs, API responses. Our 11% is a floor result from partial usage, not a representative benchmark of full integration.

Zero quality degradation aligns with Headroom's GSM8K accuracy claims (±0.000 delta). The 36% context re-read reduction aligns with the mechanism: leaner context → cheaper re-reads per iteration turn.

A build using Headroom across all tool outputs (not just documentation) would likely land closer to the 40–80% real-world range.

---

## Limitations

- Single run per condition (no statistical replication)
- Both conditions hit the same stale authentication issue (macOS Keychain overriding env credentials), resolved the same way (symmetric noise)
- Only 4 `headroom_compress` calls made; Headroom was used on SDK documentation only, not all tool outputs; results understate full-integration potential

---

## Replication

### Requirements

- Claude Code with a Claude Sonnet 4.6 session
- `@servicenow/sdk` 4.7.2
- Headroom v0.26.0: `pip install headroom`
- Headroom registered as MCP in `~/.claude.json`:

```json
"headroom": {
  "type": "stdio",
  "command": "/path/to/venv/bin/headroom",
  "args": ["mcp", "serve"],
  "env": {}
}
```

- A ServiceNow instance with Fluent SDK support

### Session identification

Both sessions were opened from the same parent directory as separate IDE chat tabs. Each session prompt included a unique build marker in the first user message (`BUILD_CONDITION_A_HR_2026` / `BUILD_CONDITION_B_NHR_2026`) to enable JSONL session identification for cost extraction.

### Cost extraction

```python
import json, os
from collections import Counter

path = "~/.claude/projects/<project-hash>/<session-id>.jsonl"
input_tok = output_tok = cw = cr = 0
for line in open(os.path.expanduser(path)):
    ev = json.loads(line)
    usage = ev.get("message", {}).get("usage", {})
    if usage:
        input_tok += usage.get("input_tokens", 0)
        output_tok += usage.get("output_tokens", 0)
        cw += usage.get("cache_creation_input_tokens", 0)
        cr += usage.get("cache_read_input_tokens", 0)

cost = input_tok*3/1e6 + output_tok*15/1e6 + cw*3.75/1e6 + cr*0.30/1e6
print(f"${cost:.4f}")
```

---

## Social card

An animated comparison card was generated from `headroom_comparison.html` using `capture_headroom.py` (Playwright + ffmpeg, 60fps, 12s loop).
