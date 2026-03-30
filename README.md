# Skill Cost Analyzer for OpenClaw

An OpenClaw skill that estimates token consumption of any skill before running it. **This skill only performs estimation and never executes the target skill.**

Adapts to each user's runtime environment: detects your LLM model, available tools, and platform for personalized estimation.

## Installation

Copy the `Skill-Cost-Analyzer/` folder to your OpenClaw skills directory:

```bash
# Workspace-level (this project only)
cp -r Skill-Cost-Analyzer/ <your-project>/skills/

# Managed-level (all projects)
cp -r Skill-Cost-Analyzer/ ~/.openclaw/skills/
```

Or paste the GitHub repository link directly into an OpenClaw chat for automatic setup.

## Usage

### Basic usage

```
/skill-cost-analyzer <skill_name>
```

Analyzes the target skill with the default 50,000 token limit.

### With custom limit

```
/skill-cost-analyzer coding-agent --limit 100000
```

### Show detailed analysis

```
/skill-cost-analyzer browser --detail
```

## Options

| Flag | Default | Description |
|------|---------|-------------|
| `--limit <number>` | 50000 | Custom token consumption threshold |
| `--detail` | false | Show detailed analysis breakdown including model detection and tool cross-referencing |

## How It Works

### Personalized Runtime Detection (Step 0)

Before any analysis, the skill reads your OpenClaw session context to detect:

- **Your LLM model** -- selects model-specific tokenization rates
- **Your available tools** -- cross-checks against the target skill's tool requirements
- **Your registered skills** -- enables zero-cost skill lookup
- **Your platform** -- OS and runtime information

### Model-Aware Tokenization

Different models tokenize text differently. The skill adapts:

| Model Family | English chars/tok | CJK chars/tok | Context Window |
|-------------|------------------|--------------|---------------|
| Claude | 3.5 | 1.5 | 200k |
| GPT-4o / o-series | 4.0 | 1.2 | 128k |
| DeepSeek | 3.8 | 1.8 | 128k |
| Gemini | 4.0 | 1.5 | 1M+ |
| Qwen | 3.8 | 1.2 | 128k |

Unknown models use conservative defaults.

### Universal Skill Analysis

Works with any skill source:

- **File-based skills** (SKILL.md or legacy .md): reads full prompt, High confidence (+-30%)
- **Known built-in skills** (14 profiled): uses behavior profiles, Medium confidence (+-50%)
- **Unknown skills**: description-based analysis, Low confidence (+-70%)

### 5-Dimension Analysis

1. **Prompt Tokens** -- model-aware character-to-token conversion
2. **Tool Call Density** -- 25+ OpenClaw tools covered (file, exec, browser, web, agent/session, messaging, device, memory, image)
3. **Iteration Depth** -- loop and multi-step detection
4. **Sub-Agent Usage** -- sessions_spawn / subagents cost multiplier
5. **Output Scale** -- expected output size classification

### Token Hot Spots & Optimization Suggestions (only when over budget)

When estimated tokens **exceed your limit** (BLOCKED or WARNING), the report additionally includes:
- **Token Hot Spots**: the top 3 most token-hungry parts of the skill, with plain-language explanations of why they're expensive
- **Optimization Suggestions**: 2-4 specific, actionable tips you could use to rewrite the skill, each with estimated savings

These are **suggestions for your reference only** -- you decide whether to act on them. When the estimate is within budget (PASS), these sections are omitted to keep the report concise.

### Tool Availability Cross-Check

If the target skill references tools not available in your environment, the report warns you -- so you know the skill may fail before you run it.

## Complexity Levels

| Level | Token Range | Typical Characteristics |
|-------|-------------|------------------------|
| Lightweight | 1k - 5k | Single read/query, no agent calls |
| Medium | 5k - 30k | Multi-file read/write, few tool calls |
| Heavy | 30k - 100k | Agent/sub-process, multi-round iteration |
| Ultra-heavy | 100k+ | Multi-agent parallel, large-scale codegen |

## Example Output

### When within budget (PASS) -- clean and concise:

```
+====================================================+
|           Token Efficiency Report                  |
+====================================================+
| Target Skill:     commit                           |
| Skill Type:       Built-in                         |
| Skill Location:   System                           |
| Confidence:       Medium                           |
+----------------------------------------------------+
| Model:            gpt-4o                           |
| Tokenizer:        GPT Profile                      |
| Platform:         linux                            |
+----------------------------------------------------+
| Prompt Tokens:          ~800                       |
| Est. Tool Calls:        4-6 calls                  |
| Est. Tool Tokens:       ~5,500                     |
| Est. Output Tokens:     ~500                       |
| Sys Prompt Overhead:    ~1,200                     |
| Sub-Agent:              No                         |
| Iteration Multiplier:   x1                         |
| Sub-Agent Multiplier:   x1                         |
+----------------------------------------------------+
| Complexity:       Medium                           |
| Est. Total:       ~8,000 tokens                    |
| Range:            4,000 - 12,000 tokens            |
| User Limit:       50,000 tokens                    |
| Context Window:   128,000 tokens                   |
| Est. vs Window:   6%                               |
| Status:           PASS                             |
+====================================================+

You can safely run /commit -- estimated cost is
within your 50,000 token limit.
```

### When over budget (BLOCKED) -- includes hot spots & suggestions:

```
+====================================================+
|           Token Efficiency Report                  |
+====================================================+
| Target Skill:     coding-agent                     |
| Skill Type:       Built-in                         |
| Skill Location:   System                           |
| Confidence:       Medium                           |
+----------------------------------------------------+
| Model:            claude-sonnet-4-5-20250929       |
| Tokenizer:        Claude Profile                   |
| Platform:         darwin (macOS)                   |
+----------------------------------------------------+
| Prompt Tokens:          ~2,500                     |
| Est. Tool Calls:        15-25 calls                |
| Est. Tool Tokens:       ~35,000                    |
| Est. Output Tokens:     ~10,000                    |
| Sys Prompt Overhead:    ~1,500                     |
| Sub-Agent:              Yes (1-2)                  |
| Iteration Multiplier:   x3                         |
| Sub-Agent Multiplier:   x4                         |
+----------------------------------------------------+
| Complexity:       Ultra-heavy                      |
| Est. Total:       ~588,000 tokens                  |
| Range:            294,000 - 882,000 tokens         |
| User Limit:       100,000 tokens                   |
| Context Window:   200,000 tokens                   |
| Est. vs Window:   294%                             |
| Status:           BLOCKED (8.8x over limit)        |
+----------------------------------------------------+
| Tool Warnings:                                     |
|   (none - all referenced tools available)          |
+----------------------------------------------------+
| Token Hot Spots (where your tokens go)             |
+----------------------------------------------------+
| #1  Sub-agent spawning (sessions_spawn x2)         |
|     ~220,000 tokens (37% of total)                 |
|     Why: Each sub-agent starts a whole new          |
|     conversation, duplicating all the context.      |
|     Two sub-agents = double the duplication.        |
+----------------------------------------------------+
| #2  File reading inside a loop (read x15)          |
|     ~150,000 tokens (26% of total)                 |
|     Why: Every file read dumps the full file        |
|     content into the conversation. Reading 15       |
|     files in a loop adds up fast.                   |
+----------------------------------------------------+
| #3  Iteration multiplier (x3 loop)                 |
|     ~120,000 tokens (20% of total)                 |
|     Why: The skill repeats its core logic 3         |
|     times. Everything inside the loop costs 3x.     |
+====================================================+
| Optimization Suggestions (for your reference)      |
+----------------------------------------------------+
| 1. Replace sub-agents with direct tool calls       |
|    What to change: If the sub-task is just          |
|    "read files and edit code", do it directly       |
|    instead of spawning a new agent for it.          |
|    Why it helps: Avoids duplicating the entire      |
|    conversation context for each sub-agent.         |
|    Est. savings: ~150,000 tokens (25%)              |
+----------------------------------------------------+
| 2. Read files once before the loop                 |
|    What to change: Move all file reads outside      |
|    the loop. Read once, store results, then         |
|    process them in the loop.                        |
|    Why it helps: You only pay the token cost        |
|    of reading each file once instead of every       |
|    iteration.                                       |
|    Est. savings: ~100,000 tokens (17%)              |
+----------------------------------------------------+
| 3. Add early-exit to reduce loop iterations        |
|    What to change: Add a condition to stop the      |
|    loop early when the task is done, instead        |
|    of always running all 3 iterations.              |
|    Why it helps: If the task finishes in 1          |
|    iteration, you save 2x worth of tokens.          |
|    Est. savings: ~80,000 tokens (14%)               |
+----------------------------------------------------+
| Note: These are suggestions only. You decide       |
| whether to modify your skill. After changes,       |
| run /skill-cost-analyzer again to verify savings.  |
+====================================================+
```

> Note: This skill only reports estimates. It never executes the target skill. Hot spots and suggestions only appear when the estimate exceeds your limit -- you decide whether to act on them.

## Limitations

- Estimates are approximate -- accuracy varies by skill type (see confidence levels)
- Built-in skills are estimated from behavior profiles, not full prompt analysis
- Model detection depends on OpenClaw's Runtime section being present
- Tool output sizes depend on actual content; estimates use historical averages
- Sub-agent and browser behaviors have the highest variance
- Browser screenshot/vision tools may consume vision tokens not captured in text-token estimates

## License

MIT
