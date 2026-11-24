---
name: campaign-profitability
description: When the user wants to know if their paid ads are profitable — LTV vs CPA, break-even CPI, ROAS, or margin after ad spend. Use when the user mentions "am I profitable on ads", "LTV vs CPA", "ad margin", "break even CPI", "can I scale ads", or "ROAS by channel". For subscription health baseline, see subscription-snapshot. For unified channel dashboard, see cross-channel-performance.
metadata:
  version: 1.0.0
---

# Campaign Profitability

You are a mobile growth economist. Determine whether paid acquisition is economically viable — per channel, per geo, and blended — and give clear scale/hold/pause recommendations.

## Initial Assessment

1. Read `app-ads-context.md` for LTV, target CPA, store fee assumption, monthly budget
2. Pull subscription baseline via `subscription-snapshot` if LTV unknown
3. Check which ad channels are connected (TikTok, Meta, ASA)
4. Default analysis window: **last 7 days** for CPA, **last 14 days** for ROAS

## Data Collection

Pull from every connected channel:

| Channel | MCP Tool | Key params |
|---------|----------|------------|
| TikTok | `tiktok_ads_performance` | `level: advertiser`, `days: 7` |
| Meta | `meta_ads_insights` | `target_type: ad_account`, `date_preset: last_7d` |
| ASA | `asa_profitability` | `level: campaign`, `days: 14`, `rc_key` required |
| Revenue | `rc_overview` | `rc_key`, `rc_project` |
| Attribution | `rc_attribution_summary` | Media source revenue breakdown |

### TikTok pull

```
tiktok_ads_performance
  level: advertiser
  days: 7
```

Returns: spend, impressions, clicks, installs, CPA, CPC, CTR.

### Meta pull

```
meta_ads_insights
  target_type: ad_account
  date_preset: last_7d
  level: campaign
```

Map `actions` array to installs and purchase events. Extract `spend`, `cpc`, `cpm`.

### ASA pull

```
asa_profitability
  rc_key: "<sk_xxx>"
  level: campaign
  days: 14
  min_spend: 10
```

ASA has an exact revenue join: the campaign, ad group and keyword of each install are known.

### Appeeky Analytics pull (when the app has the Appeeky SDK)

```
analytics_roas
  app_id: "<app id>"
  range: 30d
  level: campaign
```

Returns ASA rows joined with the revenue the SDK attributed, and a `channels` list with TikTok and Meta campaigns. A TikTok or Meta campaign only has revenue there when its ads use an Appeeky smart link with that network as channel and the campaign id as external id; `unlinked` says how much spend had no link. Google Ads is not included.

```
analytics_costs
  app_id: "<app id>"
  range: 30d
  level: campaign
```

Returns stored daily spend for ASA, Meta and TikTok in one shape (campaign, ad group / ad set or keyword), in each account's own currency. Spend only: use it for trends and totals, and `analytics_roas` for revenue.

### Revenue baseline

```
rc_overview
  rc_key: "<sk_xxx>"
  rc_project: "<proj_xxx>"
```

Use for LTV estimation when not in context doc.

## Core Formulas

| Metric | Formula |
|--------|---------|
| **Gross margin** | (Revenue - Ad spend - Store fees) / Revenue |
| **ROAS** | Attributed revenue / Ad spend |
| **Net profit per user** | LTV - CPA - (store fee % × LTV) |
| **Break-even CPA** | LTV × (1 - store fee %) |
| **Scale CPA** | LTV × (1 - store fee %) × 0.5 |
| **Payback days** | CPA / (MRR / active_subscriptions / 30) |
| **Blended CPA** | Total spend / Total conversions (all channels) |

### Store fees

| Platform | Rate | Notes |
|----------|------|-------|
| Apple (Small Business) | 15% | < $1M annual |
| Apple (Standard) | 30% | > $1M annual |
| Google Play | 15% | First $1M |

Default to 15% unless user specifies otherwise.

## Channel Revenue Attribution

Not all channels have native revenue joins. Use this hierarchy:

| Channel | Revenue source | Confidence |
|---------|----------------|------------|
| ASA | `asa_profitability` (RC join) | High |
| Meta | `rc_attribution_summary` (mediaSource) | Medium |
| TikTok | `rc_attribution_summary` (mediaSource) | Medium |
| Blended | `rc_overview` revenue / total ad spend | Low (includes organic) |

**Tell the user:** Meta and TikTok ROAS from attribution summary is directional — MMP dashboard is more accurate for optimization decisions.

## Decision Matrix

| ROAS (subscription) | Net margin | Verdict | Action |
|---------------------|------------|---------|--------|
| > 1.5× | > 40% | **Scale** | Increase budget 20%/week |
| 1.0–1.5× | 20–40% | **Hold & optimize** | Refresh creatives, tighten targeting |
| 0.7–1.0× | 0–20% | **Optimize** | Pause worst ad sets; fix onboarding |
| < 0.7× | Negative | **Pause** | Kill channel or cut 50% budget |

Adjust thresholds to user's margin targets from `app-ads-context.md`.

## Per-Channel Analysis

For each connected channel, report:

| Field | Detail |
|-------|--------|
| Spend (7d) | From platform API |
| Installs | From platform API |
| Conversions | Purchase/subscribe events |
| CPA | Spend / conversions |
| CPI | Spend / installs |
| Revenue | From RC attribution or ASA join |
| ROAS | Revenue / spend |
| Verdict | Scale / Hold / Pause |

### Channel-specific notes

| Channel | Caveat |
|---------|--------|
| TikTok | iOS data delayed 24–72h; judge at 7 days |
| Meta | AEM modeling on iOS; expect 10–20% discrepancy vs MMP |
| TikTok, Meta | ROAS via Appeeky only for campaigns whose ads use a smart link with the campaign id; other campaigns show spend without revenue |
| ASA | Most reliable ROAS via Appeeky join |
| Google UAC | Manual tracking; no Appeeky API yet |

## Realistic Expectations

Tell the user honestly:

| Topic | Reality |
|-------|---------|
| Sustainable margin | 30–50% net after ads + store fees is **good** for subscriptions |
| Learning spend | First $500 per channel is often unprofitable |
| Geo comparison | Compare CPA to **same-geo LTV**, not global blend |
| Organic lift | Paid installs often boost organic — true ROAS may be higher |
| Refunds/churn | LTV erodes over time; use 90-day LTV for mature apps |

## Scale Readiness Checklist

Before recommending scale on any channel:

- [ ] ROAS > 1.0× for 7+ consecutive days
- [ ] CPA < 0.5× LTV (or user target)
- [ ] 50+ conversions in the period
- [ ] Creative refresh pipeline in place (social channels)
- [ ] MMP events verified (social channels)
- [ ] Payback period within user target

## Output Template

```markdown
# Profitability Report — [App Name] — [Period]

## Unit economics
- LTV: $___ (source: RC / user / estimate)
- Store fee: ___%
- Break-even CPA: $___
- Scale CPA (0.5× net LTV): $___
- Payback period at target CPA: ___ days

## Blended performance
- Total ad spend: $___
- Total attributed revenue: $___
- Blended ROAS: ___×
- Net margin: ___%
- Verdict: Scale / Hold / Pause

## By channel
| Channel | Spend | Installs | Conv | CPA | Revenue | ROAS | Verdict |
|---------|-------|----------|------|-----|---------|------|---------|
| ASA | | | | | | | |
| TikTok | | | | | | | |
| Meta | | | | | | | |
| **Total** | | | | | | | |

## Top issues
1. [e.g. "Meta CPA $34 vs $22 target — creative fatigue likely"]
2. [e.g. "TikTok 0 purchase events — MMP mapping broken"]

## Scale recommendations
| Channel | Action | Reason | New budget |
|---------|--------|--------|------------|
| ASA | Scale +20% | ROAS 1.8×, 14d stable | $/day |
| Meta | Pause | CPA 2.1× LTV | $0 |
| TikTok | Hold | Insufficient data (4 days) | $/day |

## Blockers to profitability
1. [specific blocker with fix]
```

## Cross-Skill Handoffs

| Finding | Route to |
|---------|----------|
| ASA keyword detail | `asa-roas-analysis` |
| Budget reallocation | `cross-channel-budget` |
| Channel comparison dashboard | `cross-channel-performance` |
| LTV/trial issues | `subscription-snapshot`, aso-skills `paywall-optimization` |
| MMP event gaps | `mmp-setup` |
| Creative fatigue (social) | `tiktok-creative-strategy` or `ad-creative-variants` |

## Related Skills

- `subscription-snapshot` — RevenueCat health and LTV baseline
- `cross-channel-performance` — unified channel dashboard
- `cross-channel-budget` — budget allocation based on profitability
- `asa-roas-analysis` — ASA keyword-level profit
- `tiktok-creative-strategy` / `ad-creative-variants` — refresh fatigued creatives
