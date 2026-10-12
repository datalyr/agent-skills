---
name: datalyr-ads
description: Create and launch Meta ads through DATALYR's MCP server (api.datalyr.com/mcp) — upload creative, build paused campaigns, ad sets and ads, preview every placement, then turn them on within the workspace's spend limits. Use when the user wants to make, launch, pause or change ads, budgets or campaigns, or asks what their ad accounts can take.
---

# DATALYR ads

DATALYR connects to the user's own Meta ad account. You create everything **paused**; money starts only through `activate_ad_objects` (or a budget raise), and only within limits the workspace owner set. Every write is two calls: a preview, then the same call with `confirm_token` after the person says yes.

## Before anything

1. `list_workspaces` (pass `workspace_id` everywhere if there's more than one).
2. `list_ad_accounts` — pick the account with `can_publish: true`. If it says `reconnect` or `connect`, tell the person to fix it in DATALYR (Sources) and stop.
3. `get_ad_publish_settings` — shows the access level of this connection (`read_only`, `propose`, `create_paused`, `publish_live`) and the enhancement defaults.

## The two-step rule (every write tool)

- Call **without** `confirm_token` → a preview with `status`:
  - `incomplete`: ask the person **every** `missing[].question` in one message (offer `options`, mention `default`), then preview again.
  - `invalid`: apply each `errors[].fix` once and preview again. If it's still blocked, tell the person why in plain words.
  - `ready`: show `summary[]` **exactly as given** (and `all_placements_url` for ads), then ask for an explicit yes.
- Only after a clear yes, call again with the **same input** plus `confirm_token` (valid 30 minutes, once) and a stable `operation_id`. Retries of the same request reuse the same `operation_id`.
- `approval_required`: give the person `approval_url`; an owner or admin approves in DATALYR. Check later with `get_ad_status`.
- `unknown`: the platform didn't answer. Don't retry blind — check `get_ad_status` / Ads Manager (label `datalyr-agent`) first.

Never change the input between the preview and the confirmed call. Never invent a URL, price, offer, claim or discount.

## Add ads to an existing ad set (most common)

1. `list_ad_targets(account_id)` — campaigns and their ad sets; use ones with `can_add_ads: true`. Ask which one if it's ambiguous, using names, not ids.
2. `get_ad_requirements(account_id, container_id)` — allowed formats, text limits, file shapes, identities, what you must ask.
3. Files:
   - Attached in this chat (ChatGPT): `import_ad_assets` with `files`.
   - Online: `import_ad_assets` with `urls` (Google Drive and Dropbox share links work; other links must be the file itself, not a page).
   - Local, in a shell/sandbox: `create_asset_upload` → `curl -X PUT --data-binary @file` with the returned headers → `finish_asset_upload`. In Claude.ai the person may need to allow `api.datalyr.com` (Settings → Capabilities).
   - Poll `get_ad_asset` until `ready`. Reuse what's there with `list_ad_assets`.
4. Write the copy yourself within the limits from step 2.
5. `create_ad` preview → show summary + all-placements link → yes → `create_ad` with `confirm_token`. Ads are created **paused**.

## Launch from scratch (campaign → ad set → ads → on)

1. `create_campaign` — objective `sales` or `traffic`. Either a campaign budget (`budget.daily`, ad account currency, e.g. `50` = 50.00/day) shared by its ad sets, or none (then each ad set has its own).
2. Planning reads: `list_ad_pixels` (sales), `search_targeting` (interests, behaviors), `list_ad_audiences`.
3. `create_ad_set` — `goal.optimize_for`: `conversions` (sales; pixel + event, default PURCHASE) or `landing_page_views` / `link_clicks` (traffic). Targeting by country codes, ages, genders, interests, behaviors, audiences. Advantage+ audience is on by default and needs `age_min ≤ 25` and `age_max 65`; set it `false` for a strict range. Placements `automatic` unless the person wants specific ones.
4. `create_ad` into the new ad set (step list above).
5. Turn on: `activate_ad_objects` with the ads, ad set and campaign in **one** call (DATALYR turns on the campaign last). The preview states the new daily spend; show it, then get the yes.

Everything stays paused until step 5, so the person can also turn it on themselves in Ads Manager.

## Money: turning on and budgets

- Only `activate_ad_objects` and `set_ad_budget` (raising) start or add spend.
- They apply directly only for the **workspace owner** connected over MCP with the Publish live level, with live ads switched on for that ad account, inside the spend limits. Otherwise the yes becomes an owner approval link — that's expected, not an error.
- Meta may spend up to 1.75× a daily budget on a single day (never more than 7× it in a week). Say so when you show the new spend.
- `get_ad_objects` shows status, budget and today's spend for up to 50 objects.
- `pause_ad_objects` pauses; with Create paused access you can only pause what agents made.
- **Stop:** if the person says stop, or spend looks wrong, call `stop_agent_ads` immediately (no preview). It switches live ads off and pauses everything agents turned on.

## Good ads (what to draft)

- Lead with the outcome for the customer in the first 125 characters of primary text; Meta cuts the rest behind "See more" in feeds.
- Keep headlines short; `get_ad_requirements` gives the exact limits, and the all-placements page shows where each placement cuts them.
- Give 2–5 primary texts and headlines when the format allows; Meta rotates them.
- Vertical 9:16 for Stories and Reels, 4:5 or 1:1 for feeds. When the person has both, send both (placement variants) so nothing is cropped.
- Use the person's own landing page and domain. DATALYR adds tracking (UTMs + ad ids) to every link; don't add your own.
- Keep enhancements off unless the person asks (the workspace defaults are shown in the preview).

## After creating

- `get_ad_status(operation_id)` — each ad's review status (`in_review`, `approved`, `rejected` with Meta's reason) and Ads Manager links.
- `list_ad_activity` — the log of everything agents did.
- For results, use DATALYR's analytics tools (`get_ads`, `get_attribution`): revenue is verified, not Meta's estimate.

## What not to do

- Don't create housing, credit, employment or political ads (not supported).
- Don't turn anything on, raise a budget, or retry an `unknown` create without the person's explicit yes in this conversation.
- Don't paste or log `confirm_token` or upload URLs for the person; they're single-use plumbing.
- If something you need doesn't exist as a tool, say so and call `report_missing_capability`.
