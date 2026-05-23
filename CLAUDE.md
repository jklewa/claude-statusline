# Claude Statusline - Development Guide

## Project Overview

Real-time usage tracking statusline for Claude Code using shim architecture.

**Repository**: `~/Projects/cc-statusline`
**Active Branch**: `feature-daily-usage`
**Remotes**:
- Dev fork: `git@github.com:hell0github/claude-statusline-dev.git`
- Production: `https://github.com/hell0github/claude-statusline.git`

## Development Principles

### Open-Closed Principle
**All configuration values MUST be loaded from config files. NEVER hardcode values in scripts or comments.**

**Rules:**
1. **No hardcoded config values in code**
   - ❌ `LAYER1_THRESHOLD=14.29`
   - ✅ `LAYER1_THRESHOLD=$(calculate from config multiplier)`

2. **No hardcoded values in comments**
   - ❌ `# Layer 1: 0-14.29% actual`
   - ✅ `# Layer 1: 0-1.0×base threshold`

3. **Config-driven thresholds**
   - All layer thresholds use `threshold_multiplier` notation
   - Base thresholds calculated at runtime from limits or dynamic values
   - Formula: `layer_threshold = base_threshold × threshold_multiplier`

4. **Examples:**
   ```bash
   # 5-hour window: base = EFFECTIVE_COST_LIMIT (back-computed from $COST / official %)
   LAYER1_THRESHOLD=$(awk "BEGIN {print $EFFECTIVE_COST_LIMIT * $LAYER1_THRESHOLD_MULT}")

   # Daily static: base = weekly_limit / 7
   DAILY_BASE=$(awk "BEGIN {print ($WEEKLY_LIMIT / 7.0) / $WEEKLY_LIMIT * 100}")

   # Daily dynamic: base = recommend value
   DAILY_BASE=$WEEKLY_DISPLAY_VALUE

   # Context: base = CONTEXT_LIMIT
   CTX_LAYER1=$(awk "BEGIN {print $CONTEXT_LIMIT * $CTX_LAYER1_THRESHOLD_MULT}")
   ```

**Benefits:**
- Single source of truth (config files)
- Easy to adjust thresholds without code changes
- Consistent behavior across all sections
- Self-documenting through config structure

## File Structure

```
~/Projects/cc-statusline/
├── src/
│   ├── statusline.sh              # Main implementation
│   ├── statusline-utils.sh        # Daily/weekly/monthly tracking utilities
│   ├── statusline-layers.sh       # Generic 2-/3-layer metric calculations
│   └── statusline-cache.sh        # Unified cache deps + invalidation
├── config/
│   ├── config.json                # User config (gitignored)
│   └── config.example.json        # Template with defaults
├── data/                           # Runtime cache (gitignored)
│   ├── .daily_cache
│   └── .official_weekly_cache
├── install.sh
├── README.md
├── CLAUDE.md
└── .gitignore

~/.claude/
└── statusline.sh                   # 2-line shim → delegates to src/statusline.sh
```

## Features

### Core Features
- **5-hour window tracker** - Current session cost with projection
- **Daily usage tracker** - 24-hour cycle aligned with weekly reset (2-layer: normal/exceeding)
- **Weekly usage tracker** - Full week percentage
- **Context window tracker** - Token usage monitoring
- **Timer** - Countdown to next reset

### Key Implementations
- **Shim architecture** - Stable interface (`~/.claude/statusline.sh`) delegates to implementation
- **Official rate-limit integration** - 5-hour and weekly percentages come from stdin `.rate_limits.{five_hour,seven_day}.used_percentage` (Claude.ai Pro/Max, populated after first API response). Sections are hidden when the field is absent (cold start / API-key users). The 5-hour displayed dollar limit is back-computed from `$COST / (official_pct / 100)`.
- **Multi-layer progress bars** - Auto-scaled visualization (different multipliers per threshold)
- **ccusage_r scheme** - Used by daily tracker and recommend mode to filter cost by Anthropic's reset schedule
- **Daily cost tracking** - `get_daily_cost()` with caching, aligned to weekly reset time
- **Daily projection** - Uses 5-hour window: `daily_cost - window_cost + projected_window_cost`
- **Daily recommendation** - Stable budget recommendation, updates only at daily cycle reset
- **Model-aware context limits** - Auto-detects model from transcript, per-model context window sizes
- **Conditional rendering** - Only computes enabled sections for performance
- **Configurable colors** - Per-layer color customization

### Daily Recommendation Logic

**Formula**: `(weekly_limit - usage_from_weekly_start_to_daily_cycle_start) / cycles_left`

**Behavior**:
- **Stable throughout each daily cycle** - Only updates at daily reset (e.g., 3pm)
- **Cycle-aligned** - Uses usage up to current daily cycle start, not current time
- **Sources cost from ccusage** - The official `rate_limits.seven_day` field gives a current % but not cycle-aware historical cost, so recommend mode uses ccusage block data directly

**Example scenarios**:

**Day 1 (right after weekly reset at 3pm):**
- Weekly start: 3pm Day 1
- Daily cycle start: 3pm Day 1 (same!)
- Usage from weekly→daily start: **$0**
- Available: $850 - $0 = **$850**
- Recommend: $850 / 7 = **$121/day**

**Day 3 (at 4pm):**
- Weekly start: 3pm Day 1
- Daily cycle start: 3pm Day 3
- Usage from weekly→daily start: Day 1 + Day 2 costs
- Available: $850 - (Day 1 + Day 2)
- Recommend: available / 5 remaining days
- **Stays at this value from 3pm Day 3 until 3pm Day 4**

**Rendering precision**:
- Dollar amount calculated from precise division: `$850 / 7 = $121.43 → $121`
- Percentage shown as rounded value: `14.29% → 14%`
- Avoids rounding error: `14% × $850 = $119` ❌ vs `$850 / 7 = $121` ✓

## Configuration

**Path**: `config/config.json`

**Key settings**:
- `user.plan` - pro/max5x/max20x
- `limits.weekly` - Per-plan weekly cost, used by daily tracker and recommend mode (the displayed weekly % comes from `rate_limits.seven_day`)
- `limits.context` - Per-model context limits (auto-detected, e.g., `default: 200`, `claude-opus-4-6: 1000`)
- `multi_layer` - 3-layer thresholds + colors for the 5-hour window
- `daily_layer` - 2-layer thresholds + colors (normal/exceeding)
- `sections.show_*` - Toggle individual sections
- `tracking.weekly_scheme` - "ccusage" (ISO week) or "ccusage_r" (official reset) — only affects recommend mode's cost slicing
- `tracking.official_reset_date` - Required for ccusage_r, daily tracking, and recommend mode

## Development

### Testing
```bash
# Test manually
echo '{"workspace":{"current_dir":"~"},"transcript_path":""}' | src/statusline.sh

# Test with official rate_limits (Claude.ai Pro/Max stdin shape)
echo '{"workspace":{"current_dir":"~"},"transcript_path":"","rate_limits":{"five_hour":{"used_percentage":42.5,"resets_at":'$(($(date +%s)+7200))'},"seven_day":{"used_percentage":28,"resets_at":'$(($(date +%s)+86400))'}}}' | src/statusline.sh

# Test daily functions
source src/statusline-utils.sh
get_daily_cost "2025-10-08T15:00:00-07:00"
```

### Path Conventions
- Always use relative paths from `$SCRIPT_DIR`
- Config: `$SCRIPT_DIR/../config/config.json`
- Data: `$SCRIPT_DIR/../data/filename`

### Git Workflow

**IMPORTANT: Commit Authorship Policy**
- **DO NOT** include Claude Code co-authorship in commit messages
- **DO NOT** add `🤖 Generated with [Claude Code](...)` footer
- **DO NOT** add `Co-Authored-By: Claude <noreply@anthropic.com>` trailer
- Keep commits clean and professional

**Commit conventions**:
- `feat:` New features
- `fix:` Bug fixes
- `refactor:` Code reorganization
- `docs:` Documentation
- `chore:` Maintenance

### Recent Updates

**v3.0** (2026-05-23) - Official `rate_limits` integration
- 5-hour and weekly percentages now come from stdin `.rate_limits.{five_hour,seven_day}.used_percentage` (Claude.ai Pro/Max, populated after first API response)
- 5-hour displayed dollar limit is back-computed from `$COST / (official_pct / 100)` so `$X/$Y` always matches the percentage
- Timer prefers official `resets_at` epoch over ccusage's `endTime`
- Removed `limits.cost` config (no longer a hardcoded 5-hour estimate)
- Removed `tracking.weekly_baseline_percent` and the calibrator tool (`tools/calibrate_weekly_usage.sh`) — calibration is now done by Anthropic
- Fixed pre-existing unit bug in `calculate_three_layer_metrics` (compare $COST to dollar thresholds, not pct to dollar thresholds)
- 5-hour and weekly sections hidden on cold start until rate_limits arrives

**v2.3** (2025-10-08) - Daily Recommendation Fix
- Fixed recommend calculation to use correct cycle-aligned logic
- Formula: `(weekly_limit - usage_from_weekly_start_to_daily_cycle_start) / cycles_left`
- Stable recommendations that update only at daily reset (3pm)
- Fixed rounding precision: calculate dollar amount from exact division

**v2.2** (2025-10-06) - Weekly Usage Calibration Tool [retired in v3.0]
- `tools/calibrate_weekly_usage.sh` - Aligns tracking with official usage
- Compensates for untracked costs (deleted transcripts, extended context)
- Interactive baseline adjustment with safety validations

**v2.1** (2025-10-05) - Daily Usage Tracking
- Two-layer daily system (14.29% normal, 21.44% exceeding)
- 5-hour window projection integration
- `get_daily_cost()` with caching

**v2.0** (2025-10-05) - Daily Foundation
- `get_daily_period()` function
- Conditional section rendering
- Configurable layer colors

**v1.5** (2025-10-02) - Project Reorganization
- src/, config/, data/ structure
- 2-line shim architecture

---

**Last Updated**: 2026-05-23
