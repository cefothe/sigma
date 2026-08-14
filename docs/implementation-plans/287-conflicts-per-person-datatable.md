# Issue #287 — `/conflicts` per-person `DataTable` + detail on the person/company pages

**Status:** analysis / not-yet-ready-to-build (4 product decisions open — see Part IV).
**Issue:** [#287](https://github.com/midt-bg/sigma/issues/287) — „свързани лица — таблица по лице, а подробностите на страницата на лицето".
**Provenance:** produced by a multi-agent analysis — five read-only code analysts (DataTable reuse, ConflictCards/routes, queries/grouping, ADR/design principles, test surface) + a synthesis pass, then a product review by the PM and PO agents. **All file paths and line numbers below refer to the `feat/registry-evidence-links` branch** (the #279 work), which is where the analysis was run.

> **Key correction to the issue's premise (verified in source).** The issue asks, as an immediate separate fix, that three reader-facing texts on `/conflicts/official/:id` stop claiming the page shows only the person's *own* stake ("собствен дял"). On this branch that is **already fixed**: `conflict.official.tsx` interpolates `declaredStakeNoun(links)` (family-aware), and the render guard `conflict.pages.render.test.tsx:112` (`not.toContain('собствен дял')`) already passes. The **only** remaining artefact is a stale, factually-wrong **code comment** at `conflict.official.tsx:13` ("Reads private-ownership interest_links only"). That is comment-only — no query change, no output change.

---

## Part I — Implementation plan

### 1. Summary & how it fits this branch (#279)

`/conflicts` today renders one `ConflictCards` card per `ConflictLink` (one official×company pair), and both `/conflicts/official/:id` and `/conflicts/company/:eik` reuse the *same* component with an `omit` prop, so list and detail are visually identical. #287 turns the **leaderboard** into SIGMA's established `DataTable` with **one row per длъжностно лице** (a person with stakes in three companies is one row, not three cards), and moves the per-link detail onto the person and company pages so those pages become worth opening.

This sits on `feat/registry-evidence-links` (the #279 work): #279 owns the Trade-Register chain, evidence tiers/seals, dataset expansion (~100→~330 links across 300+ people), and the „собственик/управител и днес" labels. #287 must ship the DataTable refactor and the detail-page split **without** rendering any #279 evidence surface it doesn't already show — it only leaves clearly-commented hooks. The DB already returns links NEXUS-ordered and family-inclusive (ADR-0032); grouping is a **presentation** concern, not a query change.

### 2. Component & pattern reuse map

| Asset | Path | Verdict | Notes |
|---|---|---|---|
| `DataTable<Row>` | `apps/web/app/components/DataTable.tsx` | **Reuse verbatim** | `Column<Row>` already supports `isRank` (corner badge), `isTitle` (card heading), `secondary` (`.col-secondary`, drops on tablet), `align='money'` (right mono cell + numeric header). Emits `<caption className="sr-only">` (line 35), `scope="col"` on every `<th>` (line 41), `data-label` for phone reflow (line 66). No client JS. **No DataTable change needed.** |
| `entity-tables.tsx` | `apps/web/app/lib/entity-tables.tsx` | **Extend** | Already the home for shared `Column<Row>[]` sets (`trendYearColumns`, `networkColumns`) plus a row-builder (`networkRows`). Add `conflictLeaderboardColumns: Column<PersonRow>[]` here — matches the established pattern; keeps the six column defs out of the route body. |
| `ConflictCards` (+ `CaseDetail`, `Timeline`, `AuthorityShares`, `ContractList`, `ContractItem`) | `apps/web/app/components/ConflictCards.tsx` (~431–453 lines) | **Reuse verbatim on detail pages; remove from leaderboard** | Keeps `omit="official"` / `omit="company"`, lazy `useFetcher` contract expansion, all sub-sections. Do **not** delete — both detail pages still render it. The person page already exposes the full `CaseDetail`, so it is already "worth opening." |
| `PageHeader`, `Breadcrumbs`, `Callout`, `Section`, `FactsList`, `ShareBar` | `apps/web/app/components/` | **Reuse verbatim** | `conflicts.tsx` keeps the `PageHeader` lede, the explanatory `Callout` block, the `FactsList` summary and the headline `ShareBar` unchanged — only the list between them swaps `ConflictCards`→`DataTable`. |
| `Chip` (`tone='strong'` / `tone='window'`) | `apps/web/app/components/ui.tsx` (≈10–12) | **Reuse verbatim** | The two conflict-signal tones already exist: `strong`=red („от собствената институция"), `window`=turquoise („към момента на договор"). Monochrome-with-accent, not colored blobs. |
| `conflictHeadline` | `apps/web/app/lib/conflicts.ts` (≈352–389) | **Reuse verbatim + copy its dedup** | Stays as-is over the **flat** links for the FactsList. Its per-ЕИК `Math.max` dedup (documented in-code, the #226 double-count guard) is the exact pattern the new grouping helper must mirror for money. |
| `fundsCellLabel`, `contractsCountLabel`, `hasContemporaneousContracts`, `contractYearsLabel`, `relationLabel`, `declaredStakeNoun` | `apps/web/app/lib/conflicts.ts` (≈45–115) | **Reuse; extend money for aggregate** | `fundsCellLabel` takes a single `ConflictLink`; extract its `{primary, total}` shape into a helper that accepts `{contemporaneousValueEur, contractValueEur}` for the grouped cell (or inline the same branch). `declaredStakeNoun` is already family-aware and correct — no change. |
| `leaderboardRankOffset`, `withParams`, `PageNav` | `apps/web/app/lib/filters.ts` | **Reuse verbatim** | Pagination math + rank offset already used by `conflicts.tsx`, `companies.tsx:130`, `authorities.tsx:107`. |
| `officialHref`, `companyConflictsHref`, `companyProfileHref` | `apps/web/app/lib/conflicts.ts` (71–82) | **Reuse verbatim** | Row title links to `officialHref`. **Company references link to `companyProfileHref(eik)` → `/companies/:eik`** (the winner's spending profile), matching `ConflictCards.tsx:130` exactly. `companyConflictsHref` (`/conflicts/company/:eik`) is the mirror *route*, not a company-name link target. |
| `NEXUS_ORDER`, `LINK_SELECT`, `SURFACED_OWNERSHIP` | `packages/db/src/queries/related-persons.ts` (≈113–120, 162–203, 262–277) | **Reuse verbatim, no SQL change** | Array arrives strongest-first; grouping happens in TS. |
| `StackedBar` / `RankedBars` | `apps/web/app/components/` | **Not in scope** | The person-page year chart / „Дял при възложителите" already come from `AuthorityShares`/`Timeline` inside `ConflictCards`. No new bar component for #287. |

**DataTable capability gap:** one real gap — the `Признаци` header must be a **plain string** (not a `ReactNode`), because `DataTable.labelOf()` returns `undefined` for non-string headers (line 31), so a `ReactNode` header yields no `data-label` on phone. Render the chips **only in the cell**; keep the header a string so the a11y contract holds. **Do not extend DataTable; do not add a wrapper.**

### 3. Data & grouping strategy

**Row shape** — new `PersonRow` in `apps/web/app/lib/conflicts.ts`:

```ts
interface PersonRow {
  officialSlug: string;
  official: string;
  institution: string;
  links: ConflictLink[];          // the person's full group (single-company name + navigation)
  companyCount: number;           // distinct ЕИК
  companyName: string | null;     // set when companyCount === 1, else null
  contractCount: number;          // Σ link.contractCount across the group
  contemporaneousContractCount: number; // Σ link.contemporaneousContractCount
  contemporaneousValueEur: number;// per-ЕИК MAX then Σ (mirror conflictHeadline)
  contractValueEur: number;       // per-ЕИК MAX then Σ (mirror conflictHeadline)
  ownInstitution: boolean;        // OR across links  (see Decision D1)
  contemporaneous: boolean;       // OR across links  (see Decision D1)
}
```

**Aggregates:**
- `companyCount` = distinct ЕИК. `NOT_REDUNDANT_FAMILY` guarantees ≤1 link per `(official, ЕИК)` reaches the DTO (`related-persons.ts` ≈121–125), so each link is a distinct ЕИК for the person.
- **Money:** copy `conflictHeadline`'s per-ЕИК `Math.max` then sum. Within one person, distinct ЕИК means the MAX is a no-op today, but **code it defensively** — a future ETL bug producing two links for one `(official, ЕИК)` must not double-count. `contractValueEur` is the winner's constant total; `contemporaneousValueEur` is a per-link window subset (MAX, deterministic, never overstated). **See Decision D2 — whether the column aggregates at all is a product call.**
- **Counts** (`contractCount`, `contemporaneousContractCount`) are **summed** across the person's links (each link's count is scoped to its own winner, so summing across distinct ЕИК is correct). Do **not** MAX these.
- **Flags** (`ownInstitution`, `contemporaneous`): the analysis defaulted to **boolean OR across all links**; the PM prefers the **strongest link's** flags for discriminability. **This is Decision D1 — must be settled before the column is testable.**

**Rank-by-strongest under NEXUS_ORDER:** the DB applies `NEXUS_ORDER` (`own_institution='exact' DESC → contemporaneous_contract_count>0 DESC → contemporaneous_value_eur DESC → link_key`). The helper **must not re-sort**. It groups by `officialSlug` into a JS `Map` (insertion order preserved): the **first occurrence** of each slug is that person's strongest link, and its position fixes the person's rank. Later links fold into aggregates only. This guarantees „a weak second link must not sink them." Encode this dependency in the JSDoc and in a test.

**SQL GROUP BY vs TS-in-loader — recommend TS grouping in the component** (not SQL, not loader):
- The loader already fetches the full NEXUS-ordered `ConflictLink[]` and the summary `conflictHeadline` needs the **flat** array; grouping in SQL would force two projections or lose the flat array the FactsList depends on.
- Keeping the loader return shape (`{ links }`) unchanged means `conflicts.loaders.test.ts` stays **entirely green** — no loader edits.
- Money/flag aggregation must exactly mirror `conflictHeadline`; keeping it in TS next to `conflictHeadline` in `conflicts.ts` keeps the two dedup implementations side-by-side and unit-testable.
- Grouping ~330 links in the component is trivially cheap; SSR only.

Call `groupByPerson(links)` **before** the pagination slice, so pagination counts **persons** not links. `pageCount`/`PageNav` derive from `personRows.length`, and the pagination unit changes `връзки → лица`.

### 4. Per-file change list

| Path | Change |
|---|---|
| `apps/web/app/lib/conflicts.ts` | **Add** `PersonRow` + `groupByPerson(links): PersonRow[]` (group by `officialSlug`, preserve insertion order = NEXUS rank, per-ЕИК MAX money mirroring `conflictHeadline`, summed counts, flags per D1). Optionally extract an aggregate `fundsCellLabel` variant. No existing export changes. JSDoc must state „input MUST be NEXUS-ordered; no re-sort". |
| `apps/web/app/lib/conflicts.test.ts` | **Add** `describe('groupByPerson')` — cases in §8. |
| `apps/web/app/lib/entity-tables.tsx` | **Add** `conflictLeaderboardColumns: Column<PersonRow>[]` — six columns per §Columns below. |
| `apps/web/app/routes/conflicts.tsx` | Replace `ConflictCards` with `DataTable<PersonRow>`. Compute `personRows = groupByPerson(links)`; paginate persons; pass `caption`, `getKey={r => r.officialSlug}`; change pagination unit to `лица`; update the stale `~98`-count comment. Keep `Callout`, `PageHeader`, `FactsList`, headline `ShareBar`, empty-state `<p>` unchanged. `conflictHeadline(links)` stays over the **flat** array. Remove `ConflictCards` import. `noindex` meta stays. |
| `apps/web/app/routes/conflict.official.tsx` | **Immediate:** fix the line-13 comment only. **#287:** already renders `ConflictCards` with `omit="official"` — leave structure; add a clearly-commented **#279 hook** for evidence seals + „собственик/управител и днес" labels. |
| `apps/web/app/routes/conflict.company.tsx` | Mirror: keeps `ConflictCards` `omit="company"`; add the same **#279 hook** comment; verify each card header carries ЕИК + company profile link. No structural change for #287. |
| `packages/db/src/queries/related-persons.ts` | **No change.** Grouping is client-side; `SURFACED_OWNERSHIP` and `NEXUS_ORDER` already correct. |
| `apps/web/app/routes/conflicts.render.test.tsx` | Rewrite `.conflict-card`/`aria-posinset` selectors → `table`/`tbody tr`/`caption`/`th[scope="col"]`. **Move** the lazy `CaseDetail` expand + zero-contract-toggle tests to the detail suite. Add grouping/rank/money/flag cases (§8). |
| `apps/web/app/routes/conflict.pages.render.test.tsx` | **Receive** the moved expand/toggle tests; add a company-breakdown link assertion. Keep the family-only guard and own-stake positive control passing **unmodified** (regression gate). |
| `apps/web/app/routes/conflicts.loaders.test.ts` | **No change** — loader shape unchanged. |

**Columns (`conflictLeaderboardColumns`):**
1. `№` — `isRank`.
2. `Длъжностно лице` — `isTitle`; `<Link to={officialHref(...)}>{official}</Link>` + `<span className="small muted">{institution}</span>`.
3. `Дружества` — name-link to `companyProfileHref(eik)` if `companyCount===1`, else a plain count (see Decision D3).
4. `Договори` — `align='num'`; count or „N от M" contemporaneous split (see Decision D4 — reuse `contractsCountLabel`).
5. `Публични средства` — `align='money'`; primary sum + „от {total}" muted line (see Decision D2).
6. `Признаци` — `secondary:true`, **plain-string header**; cell renders `Chip tone='strong'`/`tone='window'`.

### 5. The separate immediate copy/comment fix (fold into PR-B)

**Reality on this branch (verified):** the query is correct and the reader-facing texts are already family-aware. `OFFICIAL_SQL` inherits `SURFACED_OWNERSHIP = interest_class IN ('private_ownership','family_ownership')` and adds only `AND il.person_id = ?` — no `interest_class` narrowing — so the page correctly serves both classes per ADR-0032. The lede/Callout/Section-hint interpolate `declaredStakeNoun(links)`, and `conflict.pages.render.test.tsx:112` (`not.toContain('собствен дял')`) already passes.

**The one remaining defect** is the stale comment at `conflict.official.tsx:13`:

> `// One office-holder's declared ownership links. Reads private-ownership interest_links only. …`

`Reads private-ownership interest_links only` predates ADR-0032 and is a maintenance hazard — a future reader could "fix" the query to `private_ownership` only and silently break the ADR-0032 requirement. Replace with a comment that states self+family are surfaced equally via `SURFACED_OWNERSHIP` and that `OFFICIAL_SQL` adds no `interest_class` narrowing.

**Scope:** comment-only, no rendered-output change, no query change, no test change. Both PM and PO advise **folding it into PR-B** rather than shipping a standalone one-line commit to `main`.

### 6. Accessibility mapping

| Card semantics today | Table equivalent (#287) | Where |
|---|---|---|
| `<ol className="conflict-cards" role="list">` with `<li aria-posinset aria-setsize>` | Native `<table>` — **no fake ARIA roles** on `tr`/`td`. Position/size is inherent. | `DataTable` |
| — | `<caption className="sr-only">…</caption>` | `DataTable.tsx:35`; pass `caption` prop |
| — | `scope="col"` on every `<th>` | `DataTable.tsx:41` (unconditional) |
| card heading | `cell-title` promotes the Длъжностно лице cell to the phone card heading | `isTitle` |
| corner rank | `cell-rank` corner badge on phone | `isRank` |
| per-cell labels | `data-label` on each `<td>` for phone reflow | `DataTable.tsx:66` (string headers only) |

**Phone reordering** works through `isRank`+`isTitle` promotion + `data-label` — no JS. **`Признаци` header must be a plain string** so its `data-label` resolves.

**a11y tests must check:** (a) a single `<table>`; `.conflict-cards`/`.conflict-card` absent on `/conflicts`; (b) `<caption class="sr-only">` present; (c) every `<th>` has `scope="col"`; (d) `Признаци` carries `.col-secondary`; (e) no `aria-posinset`/`aria-setsize`/`role="list"` leaks onto the table. The old `<li aria-posinset>` assertions **must be deleted**, not adapted.

### 7. Preservation checklist (MUST NOT LOSE)

- [ ] **ADR-0032 anonymization** — relative's name never shown/stored/inferrable; relation never asserted beyond „свързано лице". `PersonRow` carries **zero** relative-identifying fields; the Дружества cell shows the **winner** (public record).
- [ ] **noindex** on all three routes.
- [ ] **Monochrome + red/turquoise accent** — `Признаци` uses only `Chip tone='strong'`/`tone='window'`; no new color tokens.
- [ ] **„свързано лице" wording** preserved verbatim in the explanatory Callout.
- [ ] **Empty state** — `links.length === 0` → „Все още няма публикувани връзки.", no table. Check on `links`, not `personRows`.
- [ ] **Explanatory Callout block** („Как се извежда връзката — и какво не твърди") preserved above the list — ADR-0032/ADR-0020 disclosure, legal requirement.
- [ ] **Methodology reachability** — Callout links to `/conflicts/methodology#shown` / `#contest` stay (ADR-0021 E10).
- [ ] **Terminology** — „длъжностно лице", never „официал" (ADR-0022).
- [ ] **404-not-empty-page** on `/conflicts/official/:id` and `/conflicts/company/:eik`.
- [ ] **NEXUS_ORDER after grouping** — rank = strongest (first) link's position; helper does not re-sort.
- [ ] **Read-model boundary (ADR-0020)** — `groupByPerson` introduces **no** join/field beyond published `ConflictLink[]`; `related_persons_internal` never joined.
- [ ] **Per-ЕИК money dedup** in both `conflictHeadline` (unchanged) and `groupByPerson` (new).
- [ ] **#279 out of scope** — no TR seals, no evidence tiers, no „собственик/управител и днес" labels rendered; only commented hooks.

### 8. Test plan

**Break (rewrite in `conflicts.render.test.tsx`):**
- Empty-state test querying `.conflict-cards` → assert no `<table>` + empty-state text.
- „self card AND family card" fixture with distinct slugs → **2 rows**, not 2 cards.
- Source-declaration-link, zero-contract toggle, lazy `CaseDetail` expand tests → **move to `conflict.pages.render.test.tsx`** (that UI now lives only on detail pages).
- Pagination test → `tbody tr` count; **comment the distinct-slugs assumption** so slug reuse can't pass for the wrong reason.
- TR-evidence text assertions survive; any `.conflict-card`-scoped ones move to the detail suite.

**Stay:** meta/noindex/title, `headers()` Cache-Control, empty-state text, FactsList/`.share-bar` (above the table), all of `conflicts.loaders.test.ts`, all of `conflict.pages.render.test.tsx` incl. the family-only guard and own-stake positive control.

**New `groupByPerson` cases (`conflicts.test.ts`):**
1. single link → one row, rank 1.
2. one person, 3 companies → one row, `companyCount===3`, `contractCount` summed, money per-ЕИК MAX-then-sum.
3. two interleaved people (A-strong, B-strong, A-weak in NEXUS order) → A rank 1, B rank 2 — **weak second link does not sink A**.
4. family+self mix for one person → one row; flags per D1; no relation asserted; no relative field surfaced.
5. per-ЕИК money dedup within a person (defensive: two links same ЕИК → MAX, not doubled).
6. flags rule (per D1): person whose *second* link is the contemporaneous one.
7. empty input → `[]`.

**New render cases (`conflicts.render.test.tsx`):** (A) two same-slug links → **1** `tbody tr`; (B) rank = strongest; (C) money cell shows `primary` + „от {total}"; (D) `Признаци` has `.col-secondary`; (E) `<caption>` + all `th[scope="col"]` present.

### 9. Suggested PR breakdown & sequencing

- **PR-B — list → DataTable + grouping (the spine, includes the comment fix).** `PersonRow` + `groupByPerson` (+ unit tests), `conflictLeaderboardColumns`, `conflicts.tsx` swap + pagination unit `лица`, the `conflict.official.tsx:13` comment fix, rewrite `conflicts.render.test.tsx`. No #279 dependency, no DB change.
- **PR-C — move detail to person/company pages.** Move the lazy-expand/toggle/`CaseDetail` render tests to `conflict.pages.render.test.tsx`; confirm the person page carries the company breakdown (ЕИК + `companyProfileHref`), contract split, „Дял при възложителите", year chart, declared period + declaration link; add the company-side mirror; leave a **#279 hook** comment. **May fold into PR-B** if the detail pages already satisfy „worth opening" (they largely do — mostly test relocation).
- **Deferred (#279, not this issue):** TR evidence seals, evidence tiers, dataset expansion, „и към днешна дата" labels — only commented JSX hooks left on the detail pages.

### 10. Risks

- **Rank distortion** if `groupByPerson` re-sorts or ranks by an aggregate — mitigate with insertion-order `Map` + test case 3 + JSDoc note.
- **Money double-count** if the helper sums per-link instead of per-ЕИК MAX — mirror `conflictHeadline`; test case 5.
- **Pattern invisibility** — ranking by strongest link can bury a „many moderate links" person below a „one strong link" person (PM Risk 1). Bounded: `companyCount` is a visible column and the full list fits one page; a secondary sort is a later enhancement, not a blocker.
- **Detail page must be a real, shareable page**, not an in-place accordion (PM) — the person/company routes already are.
- **Pagination denomination change** — `PER_PAGE` now counts persons; FactsList still uses `conflictHeadline(links)` over the flat array (do not switch it to grouped rows).
- **Silent test pass** — the pagination fixture relies on distinct slugs; comment it.
- **`Признаци` `data-label` gap** — non-string header → no `data-label`; use a plain-string header.
- **Test-relocation loss** — if expand/`CaseDetail` tests are deleted from the leaderboard suite but not re-added to the detail suite, lazy-load coverage disappears; the move must be explicit.
- **Дружества broken link** — link only when `companyCount===1`.

---

## Part II — Product review (PM agent)

**Verdict: conditional yes.** The list→detail split matches the target user — journalists / civil-society researchers / watchdog staff who arrive to **scan and rank**, then **investigate one person**. Today those two jobs are collapsed into one layout, so neither is done well. The one-row-per-person table serves the scanning job; ranking a person by their **strongest** link is consistent with `NEXUS_ORDER` and is the right call for „who is most entangled?".

**Why now:** #279 is the forcing function — ~100 links today (merely awkward) become ~330 across 300+ people (unusable as a card wall). Do this **before** #279's data lands, not after. Evidence is structural (code + data model), not behavioural — state it as a reasoned inference.

**Top risks:**
1. **Pattern invisibility** — a person with several moderate links can rank below a person with one exceptional link; `companyCount` is visible but doesn't drive rank. Bounded (list fits one page; secondary sort is a later enhancement).
2. **The detail page must earn the click** — it must deliver the full per-company depth (the entire `CaseDetail` per company), and be a **real, shareable route**, not an in-place drawer (watchdogs share a URL to a specific official's profile).
3. **Signal flags must stay visible at row level** — show the strongest link's flags, not a merged „has ≥1" aggregate, or the scan loses discriminability.

**Non-goals:** sortable-header data-table; summing funds across links in a way that contradicts the in-period-vs-lifetime split; any query/`NEXUS_ORDER` change.

**Success metric to hold it to:** person-detail **open-rate from the list > 25%**; detail-page **bounce < 40%**. If open-rate doesn't move, the separation didn't create value.

---

## Part III — Product review (PO agent)

**Verdict: scope is sound and dependency-free, but NOT ready-to-build as stated** — three blocking product decisions (D1–D3 below) must be recorded first. The task is correctly sized; the gap is product definition of the column set, not engineering complexity.

**Acceptance criteria („Готово е, когато"):**
- **AC-1** — 3 links across 2 persons ⇒ `/conflicts` has exactly **2 rows**; the multi-stake person's Дружества cell is non-empty.
- **AC-2** — rank follows the person's strongest link under `NEXUS_ORDER`; a weak second link does not pull the person down (unit test on `groupByPerson`).
- **AC-3** — Публични средства leads with the contemporaneous (in-period) figure, total as secondary (reuse `fundsCellLabel`).
- **AC-4** — the explanatory Callout („Как се извежда връзката…") + the `/conflicts/methodology#shown` link remain present.
- **AC-5** — noindex, empty-state („Все още няма публикувани връзки" + no table), breadcrumbs do not regress (existing tests pass unmodified).
- **AC-6** — pagination unit renders `лица`, not `връзки`.
- **AC-7** — detail page is „worth opening": an official with a `contractCount>0` link shows a „Виж договорите" toggle that reveals `CaseDetail`; the header names the official (existing `conflict.pages.render.test.tsx` passes unmodified).
- **AC-8** — ADR-0032 does not regress: a family link never renders „собствен дял" in its context and no relative name appears (existing family tests pass unmodified).

**Split & sequencing:** fold the stale-comment fix into PR-B (don't ship a standalone one-liner to `main`); PR-B is the real work (`groupByPerson` + column set + render-test rewrite); pagination `лица` is a trailing commit; the company mirror already renders per-link depth (no change unless a grouped table is wanted there — not stated). **No #279 dependency, no DB change.** Priority: **medium** — ship before #279 data lands, but the current volume isn't broken.

**Already done (not new work):** per-link depth on detail pages (magnitude bar, timeline, authority shares, contract list); the registry-evidence label rendering; the family-aware `declaredStakeNoun` on the official page.

**Belongs to #279 (not here):** „owner/manager today" labels; any new evidence-seal vocabulary; a `Признаци` column that displays evidence-kind chips.

---

## Part IV — Consolidated open decisions (block „ready to build")

These are **product calls**, not engineering defaults. PR-B can be dispatched once D1–D3 are recorded.

| # | Decision | Options | Notes |
|---|---|---|---|
| **D1** *(blocking)* | `Признаци` flag aggregation | **(a)** strongest link's flags *(PM pick — keeps „1 vs 3 contemporaneous" discriminable)* · **(b)** OR across all links *(analysis default, simpler)* | Untestable until chosen; drives `PersonRow.ownInstitution/contemporaneous` semantics. |
| **D2** *(blocking-ish)* | `Публични средства` column | **(a)** per-person sum across companies (per-ЕИК MAX-then-sum) — the issue's „сумата" · **(b)** strongest link's figures only *(PM caution re double-count framing / ADR-0024)* | Issue text says „сумата"; PM flags the cross-link-sum framing. Recommend (a) with the per-ЕИК MAX dedup and the „в декларирания период / от общата" split. |
| **D3** *(blocking — ADR-0032)* | `Дружества` cell for a mixed own+family person | **(a)** plain count „3" · **(b)** „2 + 1 свързано" | **Anonymization implication:** naming a family stake's company in a scannable list column can be *more* identifying than on a contextualized detail page. Monochrome/restraint favours a plain count; detail on the person page. |
| **D4** *(non-blocking)* | `Договори` denominator | total count · „N от M" contemporaneous split | `contractsCountLabel()` already renders „X от Y" — reusing it settles this. |
| **D5** *(non-blocking)* | List Дружества single-company cell — link or not | link the name to `companyProfileHref(eik)` *(matches `ConflictCards.tsx:130`)* · plain text, link only on detail page | Ticket only specifies a link „към профила" on the **detail** breakdown; the list cell is unspecified. Keep the ЕИК + `ExternalEikLink` on the detail page (compact cells stay clean). |

**Success metric (PM):** person-detail open-rate from the list **> 25%**, detail-page bounce **< 40%**.
