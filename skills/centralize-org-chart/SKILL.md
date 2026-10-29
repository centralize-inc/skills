---
name: "centralize-org-chart"
description: "Build or refresh an account's org chart in Centralize: leaders on top, teams grouped by business unit, leavers removed, new hires added. Say \"build the org chart for [Account]\" or \"refresh [Account]\". Also use it to map an account, clean up its contacts, or add newly imported contacts to a chart."
---

# Centralize Org Chart Builder

Build a clean, deal-ready org chart for a prospect account in Centralize, structured the way our team reads account maps: exec spine on top, business-unit group nodes under the revenue leader, region/segment sub-groups inside Sales, narrow stacked columns instead of wide flat rows. The chart exists to answer MEDDPICC questions at a glance — who's the economic buyer, where does power sit, which region owns the evaluation — so favor clarity over org-chart-perfect accuracy, and always tell the AE which lines are inferred.

Two modes, same workflow:

- **Build** — the chart is empty or thin.
- **Refresh** — the chart exists but is months stale. Detect leavers, add new hires, fix misfiled people, and preserve the AE's hand-built structure.


## Connected tools and evidence

Use the connected Centralize MCP tool list and current schemas. Tool names may be singular or batch forms (for example setOrgChartManagers and removeOrgChartNodes). Use returned IDs, honor workspace access, and inspect each batch result for partial failures. Contact search and suggestions can trigger background enrichment and require write scope. A requested build or refresh covers chart curation; it does not authorize outreach or unrelated record changes.

The GTM/post-sale template below suits sales-team account maps. Adapt the scope when the user is mapping another buying organization. Titles, geography, and job history support inference; verified reporting evidence and the AE’s confirmed structure take precedence. A leader with no reports is a reason to investigate, not proof that a team exists.

## Target structure (the template)

```
Company root
└── CEO
    ├── C-suite peers person-to-person under the CEO (CTO, CFO, CPO...)
    │   ├── co-founders/advisors stack under the founder-CTO if there are several
    │   └── CMO keeps marketing directs nested under them (VP Mktg Ops → managers)
    └── CRO (or senior-most revenue leader) — the hub; EVERY unit below is a
        │                                     group node nested under this person
        ├── [Sales]
        │   ├── [NAMER]     → NA leaders as siblings (e.g. Head of Enterprise,
        │   │                 Head of Commercial) → managers → AE stacks by geo
        │   │                 └── [Public] → public-sector leader → their AEs
        │   │                     (sub-groups can nest under a person when a
        │   │                      distinct team exists, e.g. federal)
        │   ├── [EMEA]      → EMEA VP → segment/country managers → AE stacks
        │   ├── [APAC]      → APAC VP → regional directors → their teams
        │   └── [Strategic] → strategic-sales leader (a group even for one person)
        ├── [Business Development] → BD leader → regional BD leads → SDR/BDR
        │       stacks; if NO leader exists, chain the members into one column
        │       inside the group, senior-most on top (layout convenience — flag it)
        ├── [RevOps]       → RevOps leader → ops managers, analytics, sales ops
        ├── [Customer Success] (or [Success] / [Account Management] — match the
        │       company's titles or the existing group) → CS/AM leaders as
        │       siblings → CSM/AM stacks by region/segment; a support/services
        │       VP sits here too if worth mapping
        ├── [Enablement]   → anyone with "enablement" in their title
        └── [Partnerships] → channel/alliances leader → partner managers,
                channel directors (federal SI/channel included — function wins)
```

Rules that make this shape work:

- **Every functional unit is a group node, and regions/segments inside Sales are group nodes too.** Groups carry the org story; people carry the reporting lines inside each group. A unit or region gets its box even when it holds a single person.
- **Group names are unique chart-wide.** `parentGroupName` reuses any existing group with that name wherever it sits, so you cannot have `[EMEA]` in both Sales and CS. Inside non-Sales units, use person chains instead of region sub-groups.
- **Region names:** `NAMER`, `EMEA`, `APAC`, `LATAM`. Segment names: `Strategic`, `Public`, `Enterprise`, `Mid-Market`, `Commercial` — short, matching how the company's own titles carve territories. Reuse the exact existing name from `getOrgChart` or you'll spawn a duplicate group.
- **Width control — stack, don't sprawl.** No parent fans out wider than ~3-4 boxes (real managers/directors don't count toward the cap). Same-rank ICs cluster into vertical chains by shared trait — segment in title → city/geography → function flavor — earliest start date (senior-most) on top, the rest one-per-level beneath. Keep chains ~3 deep; split longer ones by sub-geography. These chains are layout conveniences, NOT reporting lines — name each stack and its anchor in the report.
- **Solutions/presales people get no unit of their own** — nest them under the sales leader whose team they support. Marketing stays under the CMO on the CEO side.
- **An AE's hand-built arrangement outranks the template** (e.g. BD under the CMO). Keep their structural choices but fix misfiled people inside them; flag a structural choice that looks wrong rather than silently changing it.

## Workflow

### 1. Find the account

`searchCustomers` with the company name. If duplicates come back (same name/domain), call `getOrgChart` on each and use the one with a company root and contacts; one that errors "Customer not found" is junk. Tell the AE about the duplicate so they can delete it.

### 2. Pull the chart, the full roster, and suggested contacts

- `getOrgChart` first. Note existing groups (`groupName` set, `contact` null), who sits where, and red flags: leaders with zero reports, managers whose reports are in the wrong region or segment.
- `searchContacts` with the customerId and `limit: 100`. The server returns ~27-28 per page, so paginate with `cursor` until `nextCursor` is null (the response `message` states the total). Pages usually overflow and get saved to files — process them together with `jq -s`.
- Useful jq pass: for each contact print id, name, location, function/seniority, latest target-company job (title, start, isCurrent), any newer current job elsewhere, and the first prior non-target job. When checking for a newer job elsewhere, check ALL jobs flagged current and ignore membership-type titles (advisor, board, investor, member, adjunct, fellow, mentor, angel) — they aren't employment moves. Tag each contact OK / LEFT / MOVED? / NO-JOB.
- `suggestedContacts` for the account. Add current GTM/post-sale people who are missing; mention out-of-scope suggestions instead of silently dropping them.

### 3. Clean the roster before placing anyone

- **LEFT** (the target-company job has an end date, or a newer current job elsewhere started after it). Remove them from the chart. If a departed person was a leader, their team re-parents to the group automatically — call out the leadership vacancy as deal intel. If their newest current job is at the target company, they're in, even if an old employer is still wrongly flagged current.
- **NO-JOB** (target-company email but no target-company job in the data — enrichment lag). If they're already on the chart and a prior job ended around a plausible join date, keep them and flag. If they're not on the chart, don't add them; flag them for the AE. No company email and current elsewhere → junk.
- **Dedupe.** Same LinkedIn URL twice: keep the record with the company email and flag the other for merge.
- **Scope to GTM + post-sale:** C-suite, sales, BD/SDR, RevOps, solutions/presales, enablement, partnerships, marketing leadership, and CS/AM (CSMs, AMs, expansion AMs, TAMs, onboarding) — post-sale headcount is where seat-count expansion lives. Exclude talent/HR, engineers, delivery/implementation, product support, founder's associates and back office. Mention notable borderline people.
- **Flag stale data:** emails from a previous employer and missing titles. Warn the AE not to sequence to old-employer addresses.

### 4. Classify roles — titles lie

Startups often title everyone "GTM", "Growth", "Growth & BD" or "Enterprise GTM", which mixes AEs and BDRs together.

- Classify by **prior job history**, not the current title. Previously an AE, AM or sales director → carries a bag (AE). From SDR/BDR/BD-associate or early-career roles → BDR.
- Sanity-check manager titles too. A "Demand Generation Manager", or a "Director, Growth & BD" with Non-Manager seniority, is not necessarily a people manager — don't let them anchor a team without evidence.
- A leader with zero reports ("Head of Commercial", "VP EMEA") may mean their team is misfiled. Investigate before assigning reports.
- Region beats manager for AEs. London AEs under a US enterprise manager are almost always wrong; place them under the EMEA leader.

### 5. Infer reporting lines

Infer from title + prior roles + seniority + geography, and be explicit with the AE afterward about which calls were guesses:

- Sales: regional VPs → their region group; segment leaders → siblings inside the region; managers → their segment leader; AEs → the manager whose region/segment matches, stacked per the width rule.
- A distinct public-sector/federal team gets a `[Public]` sub-group.
- BD: SDRs → the BD leader. A newer permanent Head of BD sits above an older "Interim Head", who usually becomes the regional BD lead (flag it). Stack BDRs by region/segment.
- CS/AM: leaders parent to the CS group as siblings. CSMs → their segment/region manager, or a chain inside the group if that manager left.
- When a title's function points to one leader and its region to another, function usually wins — flag it.

### 6. Write the chart

`setOrgChartManager(customerId, contactId, parentContactId | parentGroupName, [groupParentContactId | groupParentGroupName])`:

- Pass **at most one** of `parentContactId` / `parentGroupName`. `parentContactId: null` puts a contact directly under the company root — use for the CEO.
- `parentGroupName` reuses an existing group anywhere in the chart, or creates it. `groupParentGroupName` nests or moves that group inside another group — e.g. create `NAMER` inside `Sales` with its first leader: `parentGroupName: "NAMER", groupParentGroupName: "Sales"`. `groupParentContactId` hangs the group off a person (e.g. `[Partnerships]` under the CRO). Groups can be nested and moved by the API, so there is no drag list.
- Create a new group with ONE call first (the leader's placement), then place its other members in parallel. Parallel creates of the same new group can race and duplicate it.
- **Moving a contact brings their whole subtree, so place people top-down, level by level:** group leaders (L1), then their direct reports (L2), then the second element of each stack (L3), and so on. Batch within a level in parallel and wait for each level before the next. The target parent is then always already in its final spot, so there are no accidental subtree drags or cycle rotations. Ignore `wasReparented`; verify at the end.
- `removeOrgChartNode(customerId, nodeId | groupName)` works on contacts AND groups. Children re-parent to the removed node's parent, the contact record is untouched, and influencer links on a removed contact are cleared. Use it for confirmed leavers, junk records, and duplicate or empty groups — run removals BEFORE restructuring. Flag ambiguous cases instead of removing them.
- Do NOT edit the chart via Claude-in-Chrome. Canvas drops have insert-and-swallow semantics and coordinate drags are unreliable — a miss quietly rewires the chart.

### 7. Verify and report

Re-pull `getOrgChart` and confirm each group contains exactly the intended branch, there are no duplicate groups, and every stack is in the intended order. Then report to the AE, concisely:

- **Removals** — who left and where they went, plus any leadership vacancy created
- **Structure as built** — compact tree, groups as headers, new additions marked
- **What was misfiled** — the biggest structural fixes (regional AEs under the wrong leader, a leader with no team, a non-manager running a team)
- **Judgment calls to sanity-check** — every inferred line worth doubting, every layout stack and its anchor
- **Cleanup for the AE** — duplicate records/accounts to merge, NO-JOB contacts, stale emails
- **Deal intel** — leadership gaps, hiring waves (new teams haven't mapped their relationships yet), execs from existing customers or competitors (warm paths), PE/ownership signals, likely EB and evaluators
- Offer the next layer: influencer flags (`setOrgChartInfluencers`) and sentiment (`updateContactSentiment`) once the AE names the champion and EB — that's the layer that actually feeds MEDDPICC.

## API limitations (tell the AE, don't fight them)

- **No group renames** — to fix a name, remove the group, re-create it with the right name, and re-attach its members.
- **No merging duplicate contact records** — flag dupes for the AE to merge in the UI.
- **Group names are unique chart-wide** (see structure rules).

## Tone of the final report

The team uses MEDDPICC. Keep it direct, no fluff. Lead with what changed, then the judgment calls, then deal-strategy observations (single-threaded risk, warm paths, where seat count lives). Never use the word "quick."
