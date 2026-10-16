---
name: datalyr-ads
description: Plan and propose Meta, TikTok and Google Ads ads through DATALYR's MCP server (api.datalyr.com/mcp) — upload creative, propose paused campaigns, ad sets and ads onto one review page the person uploads from, then turn things on within the workspace's spend limits. Use when the user wants to make, launch, pause, edit, archive or delete ads, change budgets or campaigns, or asks what their ad accounts can take.
---

# DATALYR ads

DATALYR connects to the user's own Meta, TikTok and Google Ads accounts (`list_ad_accounts` shows which ones agents can use).

**You propose; the person uploads.** No tool creates or changes anything from chat. Every ad write runs DATALYR's checks, the platform's validation and the placement previews, and a ready proposal is added to the person's **review page** (`review_url`). An owner or admin opens it, looks everything over and presses **Upload**. DATALYR then creates it all **paused**, in order, paced per ad account. Turning things on, raising budgets and deleting wait for the workspace owner, inside the spend limits they set.

## Before anything

1. `list_workspaces` (pass `workspace_id` everywhere if there's more than one).
2. `list_ad_accounts` — pick the account with `can_publish: true`. If it says `reconnect` or `connect`, tell the person to fix it in DATALYR (Sources) and stop.
3. `get_ad_publish_settings` — the access level of this connection and the enhancement defaults.
4. `get_ad_playbook(platform, account_id)` — how this ad account's ads are made. If `playbook` is null, or `gaps` lists sections, do the **first-run setup** below before planning anything. Otherwise follow it and only ask about what's new.

## First run: the ad account's playbook

The first time an agent works on an ad account, learn how its ads are made and save it once, so every later run (yours, a teammate's, another app's) follows it without asking again. DATALYR keeps it per ad account.

1. **Read the account.**
   - `list_ad_targets` — campaign and ad set names (the naming pattern), objectives, optimization goals, placements.
   - `list_ad_examples` (Meta) — the newest running ads with their names, every primary text, headline and description, CTA, links, Page / Instagram account, format, placement rules and attribution. On TikTok and Google Ads, ask the person for two or three of their best ads instead.
   - `get_ads` (last 30 days, ad level) — which ads perform, so their voice and naming count most.
   - `get_ad_publish_settings` — the AI enhancements and their defaults.
2. **Draft the playbook** from what you read, never from guesses:
   - **Naming:** campaign, ad set and ad patterns (e.g. `CBO - {product} - {geo}`, `Broad - {angle} - {age}+`, `{n}{variant} - {hook}`), each with real names that follow it.
   - **Structure:** budget on the campaign or each ad set; angles per ad set; variants per angle.
   - **Voice:** a few bullets on how they sound, what to do and avoid, and two or three lines from their best ads.
   - **Claims:** offers, prices and claims you saw (the person confirms which are current).
   - **Landing pages, CTA, Page / Instagram account, sizes** (e.g. 1:1 + 9:16 in one ad), **placements** (automatic, or what to exclude) and **attribution**.
3. **Show it to the person in one message** and ask them to correct anything, one short question per uncertain item, with options where you can. Never pre-fill a budget. Don't ask about URL parameters: DATALYR adds tracking itself.
4. **Ask which AI enhancements they want on**, and save those with `update_ad_publish_settings` (they live there, not in the playbook). If that's refused (a propose-only connection), ask the person to set them in **Settings → Ad publishing**.
5. **Save** with `save_ad_playbook(platform, account_id, playbook, confirmed: true)`. From a propose-only connection it saves as a draft; the person confirms it in Settings.

Later runs:
- Read `get_ad_playbook` first and follow it: names, structure, voice, claims, links, sizes, placements, attribution.
- Only ask about what's new (a new product, offer, landing page or angle).
- When the person corrects something that belongs to the setup ("always use LP2 for this angle", "stop saying 'game-changer'"), offer to save it, and after a yes call `save_ad_playbook` with just that field (it's a patch).
- If the account's own naming has drifted from the playbook, mention it once.

## How every write works

Call the tool with what you want. The answer is one of:

- `added_to_review` — it passed every check and is on the review page. You get `review_url`, `item_id`, the `summary[]` and any `warnings`. Keep going; don't ask for a yes per item.
- `already_in_review` — the same thing was proposed before; nothing doubled.
- `incomplete` — ask the person **every** `missing[].question` in one message (offer `options`, mention `default`), then call again. Nothing was added.
- `invalid` — apply each `errors[].fix` once and call again. If it's still blocked, tell the person why in plain words.

When everything for this request is proposed, **send `review_url` once** with a one-line summary ("3 ad sets, 9 ads, $75/day new budget"). The page updates live; don't paste every summary into chat.

Changes in chat:

- "Change the headline on 2B" → call the same tool again with the new content and `review: { replaces: "<item_id>" }`. The page marks it Updated.
- "Drop the third angle" → `remove_review_item(item_id)` (anything proposed under it goes too).
- "Is it live yet?" / "what failed?" → `get_review_session`: each item's status, created ids, ad links and reasons. `stale` means something changed since review (propose it again); `blocked` means Stop, the platform switch or the person's role stopped it.

`review.new_session: true` starts a separate page (e.g. a different client or launch). A page stays open 24 hours after its last change.

Never invent a URL, price, offer, claim or discount: use the playbook's landing pages and approved claims, or ask.

## Add ads to an existing ad set (most common)

1. `list_ad_targets(account_id)` — campaigns and their ad sets; use ones with `can_add_ads: true`. Ask which one if it's ambiguous, using names, not ids.
2. `get_ad_requirements(account_id, container_id)` — allowed formats, text limits, file shapes, identities, what you must ask.
3. Files:
   - Attached in this chat (ChatGPT): `import_ad_assets` with `files`.
   - Online: `import_ad_assets` with `urls` (Google Drive and Dropbox share links work; other links must be the file itself, not a page).
   - Local, in a shell/sandbox: `create_asset_upload` → `curl -X PUT --data-binary @file` with the returned headers → `finish_asset_upload`. In Claude.ai the person may need to allow `api.datalyr.com` (Settings → Capabilities).
   - Poll `get_ad_asset` until `ready`. Reuse what's there with `list_ad_assets`.
4. Write the copy yourself within the limits from step 2.
5. `create_ad` with `target.container_id`, **one angle per call** (up to 6 ads). Call again for the next angle.

## Launch from scratch (Meta: campaign → ad sets → ads, one review)

Propose the whole tree before anything exists; each child points at its parent's `item_id`:

1. `create_campaign` — `objective`: `sales`, `traffic`, `leads`, `awareness`, `engagement` or `app_promotion`. Budget on the campaign (shared by its ad sets) or on each ad set, never both:
   - `budget.daily` (ad account currency, `50` = 50.00/day), or `budget.lifetime` with `end_time`.
   - `bid_strategy`: `lowest_cost` (default), `cost_cap` / `bid_cap` with `amount`, or `roas_floor` with `roas_floor` (needs `value` optimization).
   → returns `campaign_ref` (its `item_id`).
2. Planning reads: `list_ad_pixels` (sales, leads), `search_targeting` (`interest`, `behavior`, `location`, `zip`, `language`), `list_ad_audiences`, `list_ad_apps` (app promotion), `list_ad_lead_forms` (instant forms).
3. `create_ad_set` with `target.campaign_ref` — `goal.optimize_for` must fit the objective:
   - sales: `conversions` or `value` (pixel + event, default PURCHASE)
   - traffic: `landing_page_views` or `link_clicks`
   - leads: `conversions` (website) or `instant_form` (needs `goal.page_id`; the Page must have accepted Meta's Lead Ads terms)
   - awareness: `reach` or `impressions`
   - engagement: `post_engagement` (`goal.page_id`) or `video_views`
   - app_promotion: `app_installs` (`goal.app_id` + `goal.store_url`)

   Targeting: `locations` (countries, regions, cities with a radius, zips, map pins; places must not overlap), `exclude_locations`, ages, genders, `languages`, interests, behaviors, audiences. Advantage+ audience is on by default and needs `age_min ≤ 25` and `age_max 65`; set it `false` for a strict range. Placements `automatic` unless the person wants specific ones. Website conversion ad sets default to 7-day click, 1-day view, 1-day engaged view attribution (Meta can't change it after creation, so ask if the person uses another). `start_time` / `end_time` optional (`end_time` is required with a lifetime budget).
   → returns `ad_set_ref`.
4. `create_ad` with `target.ad_set_ref`, one angle per call. For instant-form ad sets, pass `platform_options.lead_form_id`.
5. Send `review_url`. On Upload DATALYR creates the campaign, then its ad sets, then their ads, and checks each child against what was actually created.

Proposed parents are Meta only: on TikTok and Google Ads, the campaign / ad group must exist first (propose it, have the person upload it, then propose the ads with its real id).

Everything uploads paused. Turning on is a separate proposal (next section), or the person does it in Ads Manager.

## Money: turning on and budgets

- Only `activate_ad_objects` and `set_ad_budget` (raising) start or add spend. They need real ids: objects proposed in this review must be uploaded first.
- They go to the review page like everything else, and **only the owner** can upload them, with live ads switched on for that ad account and inside the spend limits. Say the new daily spend from the proposal.
- `set_ad_budget`: give `daily` for an object with a daily budget, `lifetime` for one with a lifetime budget (the type can't be switched). A lifetime raise counts as its daily share of the days left.
- Meta may spend up to 1.75× a daily budget on a single day (never more than 7× it in a week). Say so when you give the new spend.
- `get_ad_objects` shows status, budget and today's spend for up to 50 objects.
- `pause_ad_objects` proposes pauses; with Create paused access you can only pause what agents made.
- **Stop:** if the person says stop, or spend looks wrong, call `stop_agent_ads` immediately. It is the one write that applies at once: it switches live ads off, pauses everything agents turned on and cancels everything still waiting on review pages.

## Changing what's running

- **New content for an ad** (text, image or video, link, CTA): `update_ad_content` with the ad's id and the full new ad, as for `create_ad`. Same checks and placement previews; budget, targeting and status stay as they are. Meta reviews the new content before it delivers. A running ad's new content is the owner's to upload.
- **Archive** (`archive_ad_objects`): stops delivery and keeps reporting; a person can unarchive in Ads Manager. Prefer it to deleting.
- **Delete** (`delete_ad_objects`): can't be undone. Only when the person asks for it by name; only the owner can upload it.
- Archive and delete work only on what agents made, including everything underneath (a campaign takes its ad sets and ads along).

## TikTok and Google Ads

Same tools and the same review page; pass `platform: "tiktok"` or `"google"` in `target` (and to the planning reads). What differs:

- **Names:** ad sets are **ad groups**; `container_id` is the ad group id.
- **TikTok:**
  - Ads run as a TikTok account (`list_ad_identities`): `single_video` and `existing_post` are Spark Ads; `carousel` needs a music track (`platform_options.music_id`).
  - Ad text is 1–100 characters, no emoji.
  - Ad groups target `location_ids` from `search_targeting` (type `location`), not country codes; interests and languages come from `search_targeting` too.
  - No archive (pause or delete instead).
  - TikTok may spend up to 125% of a campaign's daily budget on a day.
- **Google Ads:**
  - Search campaigns with their own daily budget and bidding (`maximize_conversions`, `target_cpa`, `target_roas`, `maximize_clicks`, `manual_cpc`); locations and languages from `search_targeting` with `platform: "google"`.
  - Ad groups take keywords (exact, phrase, broad) and negatives.
  - Ads: responsive search (`search_text`: up to 15 headlines and 4 descriptions), responsive display and Demand Gen image (`multi_asset`), Demand Gen video (`video_responsive` with `platform_options.youtube_video_ids`).
  - No archive; shared budgets aren't changed.
  - Google may spend up to 2× a daily budget on a day, and at most 30.4× it in a month.
- **Previews:** the review page draws every Meta placement. TikTok returns its own preview link; Google has no rendered preview, so the page shows the headlines, descriptions and link.

## Good ads (what to draft)

- **Follow the playbook.** Names, structure, voice, claims and links come from `get_ad_playbook`. Without one, read the account first (`list_ad_targets`, `list_ad_examples`) and copy its naming exactly (e.g. `CBO - Shopify - US`, `Broad - Rebills - 18+`, ads `2A`, `2B`, `2C`). Don't invent a new scheme.
- **Angles and variants.** One angle = one idea for one audience; give each angle its own ad set when the person tests angles, with 2–3 creative variants per angle as separate ads.
- **Sizes in one ad.** Send the 1:1 (or 4:5) and the 9:16 of the same creative in **one** ad as placement variants, so feeds get the square and Stories / Reels get the vertical. Never make two ads for two sizes.
- **Copy.** Lead with the outcome for the customer in the first 125 characters of primary text; Meta cuts the rest behind "See more" in feeds. Keep headlines short; `get_ad_requirements` gives the exact limits. Give 2–5 primary texts and headlines when the person wants Meta to rotate them; otherwise one of each.
- **Links.** Use the person's own landing page, one per angle when they have them. DATALYR adds tracking (UTMs + ad ids) to every link; don't add your own.
- **Enhancements and extras.** Keep AI enhancements, site links and catalog add-ons off unless the person asks (the workspace defaults show on the page).
- **Placements.** Automatic unless the person names some. If they exclude a network (e.g. no WhatsApp or no Audience Network), list the rest explicitly.

## Example requests

- "Upload these 9 creatives as 3 angles into the Shopify campaign, one ad set per angle, same settings as Broad - Rebills." → read the existing ad set's settings, propose 3 `create_ad_set` (into the real campaign) and 3 `create_ad` (3 variants each, 1:1 + 9:16 in one ad), send `review_url`.
- "Launch a new leads campaign for the SaaS page, $25/day, US, two angles." → `create_campaign` → 2 × `create_ad_set` with `campaign_ref` → 2 × `create_ad` with `ad_set_ref`, send `review_url`.
- "Make the headline on 2B shorter." → `create_ad` again for that angle with `review.replaces` = its `item_id`.
- "Turn the winners on." → `get_ads` / `get_attribution` to pick them with the person, then `activate_ad_objects` with their ids; tell them the owner uploads it.
- "Stop everything." → `stop_agent_ads`, now.

## After Upload

- `get_review_session` — what uploaded, what's still going, what failed and why.
- `get_ad_status("rv-<item_id>")` — one uploaded item's ads with review status (`in_review`, `approved`, `rejected` with Meta's reason) and Ads Manager links.
- `list_ad_activity` — the log of everything agents did.
- For results, use DATALYR's analytics tools (`get_ads`, `get_attribution`): revenue is verified, not Meta's estimate.

## What not to do

- Don't create housing, credit, employment or political ads (not supported).
- Don't tell the person something is live or uploaded until `get_review_session` says `done`.
- Don't re-propose an `unknown` item: ask the person to check Ads Manager (label `datalyr-agent`) first.
- Don't paste upload URLs for the person; they're single-use plumbing.
- If something you need doesn't exist as a tool, say so and call `report_missing_capability`.
