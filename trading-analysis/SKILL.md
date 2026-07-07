---
name: trading-analysis
description: "Use whenever the user asks to analyze price action, read a chart, or plan/size a trade on a crypto symbol (e.g. 'what's your read on BTC right now', 'should I long ETH here', 'plan a trade for SOL', 'update the BTC bias'). Requires the live TradingView MCP chart plus a project-local `wiki/` directory (`wiki/index.md`, `wiki/log.md`, `wiki/trading/`) for prior-analysis context and persistence — not applicable in a typical codebase that lacks that wiki/. Drives a structured H4-bias/H1-execution/Daily-swing workflow — market structure, volume confirmation, SMC order-block/CHoCH-BOS indicators — and produces a full trade plan (entry, stop, targets, R:R, invalidation) persisted to the trading wiki. Trigger even on casual or partial asks ('thoughts on BTC?', 'is this a good entry?') — don't just eyeball the chart ad hoc, always run this workflow so bias, volume, and prior analysis are checked consistently."
---

# Trading Analysis Workflow (TradingView MCP)

Read this when the user asks to **analyze price action / plan a trade** on a crypto symbol. Use the live TradingView MCP chart. Tag all output pages `trading`.

## 0. Requirements

This workflow needs two things the current project may not have:

- **TradingView MCP tools** (`data_get_pine_lines`, `data_get_study_values`, `draw_shape`, etc.) — for reading the live chart.
- **A `wiki/` directory** at the project root (`wiki/index.md`, `wiki/log.md`, `wiki/trading/`) — for loading prior analysis (§1) and persisting the new one (§5).

Check for both before starting. If either is missing, don't invent a substitute structure or silently skip the step — tell the user this skill's prerequisites aren't present in this project and ask where analysis should be pulled from / saved instead (or whether to proceed chart-only, without wiki context/persistence).

## 1. Context: load prior analysis first

Before pulling chart data, find and read the most recent `<SYMBOL>-H4-Analysis-*.md` page in `wiki/trading/` (check `wiki/index.md` / directory listing for the latest date). Carry forward: prior bias/thesis, active trade plan and whether it played out, invalidation levels already set, and any open targets. Note what's changed vs. confirmed since that page in the new analysis, and link back to it with `[[wikilinks]]`.

## 2. Timeframes

- **Execution / price-action read: H1.** Primary timeframe for structure, entries, stops, triggers.
- **Higher timeframe (bias): H4.** Establish trend direction here first — H4 controls the bias. Only take H1 setups that align with it (or are explicit counter-trend scalps, labelled as such).
- **Swing / deep-target validation: Daily.** Pull D for major swing highs/lows and to confirm real structure (or an air pocket) behind a multi-day target. Don't project a swing target that isn't grounded in HTF structure.
- Default to the chart's current symbol/timeframe; restore the original timeframe when done.

## 3. What to read on each timeframe (in this order)

1. **Price action / market structure** — swing highs/lows, HH/HL vs LH/LL, BOS/CHoCH, range vs trend, liquidity sweeps (wicks beyond equal highs/lows). Anchor the week to its actual open (Mon 00:00 UTC); note week open / high / low.
2. **Volume — always.** Confirmation, never optional. Compare each leg's volume to the recent baseline. Climax volume (≫ baseline) at a low/high = exhaustion/sweep (don't chase). Rising price on **falling** volume = weak demand → fadeable; breakouts/breakdowns need **expanding** volume to be trusted. State a volume-based invalidation when the thesis depends on a move being weak.
3. **SMC / Order Block indicators** (LuxAlgo Smart Money Concepts, Order Block Detector) — read via `data_get_pine_lines` / `data_get_pine_boxes` / `data_get_pine_labels` (pass `study_filter`; indicator must be visible). Use OBs/zones as entries & targets; CHoCH/BOS/EQH/EQL labels to confirm structure shifts.
4. **Other visible indicators** — RSI (divergence/OB-OS), funding rate (crowded positioning / squeeze risk), EMAs, etc. via `data_get_study_values`. Confluence, not the lead signal.

## 4. Output: the trade plan

- Clear directional lean (long/short) **aligned to the H4 bias**, with a one-line rationale.
- **Entry zone, stop (with invalidation logic), laddered targets (R multiples), R:R.**
- **Timing** — session context (London/NY opens drive moves) and the concrete trigger/confirmation to wait for (e.g. H1 rejection candle, D-close beyond a level). Don't enter "in the air."
- State the **invalidation level** explicitly. Note conditions that downgrade conviction.
- Swing conversions: widen the stop to HTF invalidation, ladder targets to Daily structure, gate the deep target on a confirming HTF close + expanding volume.

## 5. Persist

- Write/update a dated page `BTC-H4-Analysis-YYYY-MM-DD.md` (or `<SYMBOL>-...`), thread it to the prior analysis with `[[wikilinks]]`, update `wiki/index.md`, and append to `wiki/log.md`: `## [YYYY-MM-DD] analysis | <symbol> <timeframes> — <summary>`.
- Optionally draw levels/zones and a position tool on the chart (`draw_shape`), screenshot to verify. Note: the native `short_position`/`long_position` shape may not render — fall back to red (risk) + green (reward) rectangles. Restore the user's timeframe afterward.
