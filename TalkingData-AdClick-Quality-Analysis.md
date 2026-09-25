# Mobile Ad Click Quality Analysis: Separating Real Signal from Noise in Click-Fraud Data

**Tools:** SQL (BigQuery), Python (statsmodels)
**Dataset:** TalkingData AdTracking Fraud Detection (Kaggle) — real mobile ad click data, ~100K clicks over Nov 6-9, 2017, with click-to-install attribution

## Overview

This project analyzes anonymized mobile ad click data to answer a core growth marketing question for app-based businesses: which clicks represent real, valuable traffic, and which represent noise or fraud that's quietly wasting acquisition spend? The process deliberately included catching and correcting real data-quality issues along the way — a date-parsing bug and a reused filter that silently corrupted an earlier conclusion — because that diagnostic process is as much a part of the deliverable as the final numbers.

## 1. Baseline Funnel & Channel Performance

Overall click-to-install conversion across the full dataset: **0.227%** — roughly 2-3 installs per 1,000 clicks, consistent with industry norms for ad click data, where the large majority of traffic never converts.

Breaking this down by channel (filtered to channels with 500+ clicks, to avoid trusting rates built on tiny samples): most channels converted near 0%, while a small number stood out — **channel 347 (2.17%, 507 clicks)** and **channel 101 (1.10%, 1,180 clicks)** — both with real volume behind the rate, unlike smaller channels showing misleadingly high percentages from just a handful of clicks.

## 2. Time-Gap Fraud Signal Detection

Using `LAG` and `TIMESTAMP_DIFF`, each click was compared to the previous click from the same IP address. **502 clicks (out of ~100K) occurred with a 0-second gap from the prior click on the same IP** — inhumanly fast, repeated clicking, a classic bot/click-farm signal.

Counterintuitively, these "suspicious" zero-gap clicks converted at a **higher** rate (0.40%) than normal clicks (0.08%) — the opposite of what a simple "bots never convert" assumption would predict. Given the small sample (2 installs out of 502 clicks), this was treated as an ambiguous signal worth flagging, not proof either way — it could reflect small-sample noise, deliberate fraud designed to occasionally "convert" to evade detection, or an unrelated technical artifact (like duplicate tracking pixels).

*(Note: this phase also surfaced and fixed a real data-quality bug — BigQuery's schema auto-detect misread the source CSV's `DD-MM-YYYY` dates as `MM-DD-YYYY`, silently swapping day and month. Caught by noticing the "fixed" date range spanned three months instead of the documented three days, and corrected before any time-based analysis proceeded.)*

## 3. Channel-Level Fraud Distribution

Extending the suspicious-click flag to a per-channel breakdown found a **narrow, evenly-distributed spread (0.15%–1.36%) across all 39 channels** — no sharp cluster of "bad" channels separate from "clean" ones. This pattern is more consistent with a shared technical cause affecting all channels similarly than with fraud deliberately concentrated on specific channels, which would typically show a sharper, more uneven distribution.

One channel — **377** — stood out at this stage with both an above-average suspicious rate (1.14%) and the highest raw conversion rate (0.38%), flagged for individual follow-up.

## 4. Time-of-Day Pattern & Significance Testing

Grouping conversion by hour-of-day (UTC) revealed a sharp drop in both volume and conversion during hours 16-20 UTC. Using outside context about the dataset's origin — TalkingData is a Chinese company, and the dataset's traffic is predominantly Chinese — converting to China Standard Time (UTC+8) maps this window to **midnight–4am local time**, aligning precisely with natural overnight human activity drop-off rather than anything suspicious.

A two-proportion z-test confirmed the nighttime-vs-daytime conversion gap (0.07% vs. 0.24%) is statistically significant (p = 0.00113) — a real, non-random pattern. Importantly, the test proves the *difference* is real; it does not by itself prove the *cause* — the timezone explanation is a well-reasoned inference from the dataset's documented origin, not something provable from the click data alone.

## 5. Re-testing Channel 377 — A Self-Corrected Finding

Revisiting channel 377 with a proper two-proportion z-test (377 vs. all other channels) surfaced a real error: the comparison numbers initially used were built from a query that had inherited a `WHERE seconds_since_last_click IS NOT NULL` filter from earlier fraud-gap analysis — a filter that silently excluded every IP's *first* click from the count, undercounting total clicks and skewing the conversion rate.

With corrected, unfiltered numbers, channel 377's conversion rate (0.24%) was statistically indistinguishable from all other channels combined (0.23%; p = 0.9553) — the earlier "standout" did not hold up. This correction is reported transparently rather than omitted, since catching and fixing this kind of silent, filter-inherited error is a core part of trustworthy analysis.

## Strategic Recommendations

1. **Treat the zero-gap click signal as a monitoring flag, not a confirmed fraud filter.** With only 502 affected clicks and an ambiguous (higher, not lower) conversion rate, blocking this traffic outright risks losing real converting users; it's better suited to ongoing monitoring alongside other signals.
2. **Deprioritize channel-specific fraud investigation in favor of a platform-wide technical review.** The even, non-concentrated spread of suspicious activity across all 39 channels suggests the root cause is more likely a shared tracking/logging artifact than channel-specific bad actors.
3. **Schedule campaign spend awareness around the nighttime conversion dip.** If ad delivery isn't already time-targeted, the statistically confirmed 0.07% vs 0.24% gap during local overnight hours represents a real, low-return spend window worth reviewing.

## Limitations & Assumptions

- Channel, app, device, and OS values are anonymized integer IDs, not real names — findings describe patterns, not specific named platforms.
- The China-timezone explanation for the nighttime pattern is a well-supported inference from the dataset's documented origin, not something provable from the click data columns alone.
- The zero-gap "suspicious" click flag is a simple heuristic (same-IP, 0-second gap) — a production fraud system would likely combine multiple signals, not rely on this alone.
- Channel 377's initial "standout" status was a data-quality error, corrected in Day 5 — reported here transparently as part of the analysis process.
