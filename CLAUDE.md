# BSData `wh40k-10e` Schema & Librarian Ingestion Notes

Audience: Librarian maintainers. Generated 2026-05-14 against the synced fork at
`singlethreaded/wh40k-10e` (parent `BSData/wh40k-10e`).
Skill referenced: `librarian-profile/SKILL.md` (catalog walkers, weapon/ability
extraction, rule registry).

---

## 1. File Layout

| Pattern | XML root | Purpose |
|---|---|---|
| `Warhammer 40,000.gst` | `<gameSystem>` | Single root system. Declares `profileTypes`, `costTypes`, `categoryEntries`, force org. |
| `*.cat` (≈48 files) | `<catalogue>` | One per faction *or* shared library (`library="true"`). Library cats hold `sharedSelectionEntries`, `sharedRules`, `sharedProfiles`, `sharedSelectionEntryGroups` consumed by sibling faction cats via `entryLink` / `infoLink` / `catalogueLink`. |

Both file types use the same BattleScribe XML schemas:
- `http://www.battlescribe.net/schema/gameSystemSchema`
- `http://www.battlescribe.net/schema/catalogueSchema`

Canonical (community-maintained) XSDs:
https://github.com/BSData/schemas → `gameSystem.xsd`, `catalogue.xsd`.
Pull these into the librarian repo as the source of truth; they version slowly
(current `battleScribeVersion="2.03"`).

---

## 2. Core element model (what librarian actually reads)

```
catalogue / gameSystem
├── publications
├── profileTypes               # gst only — defines what a "Unit"/"Ranged Weapons" profile looks like
│   └── profileType (id,name)
│       └── characteristicTypes / characteristicType (id, name, defaultValue?)
├── costTypes                  # pts, Crusade Points, etc.
├── categoryEntries            # keyword universe (Infantry, Character, Battleline, faction kw…)
├── forceEntries               # army org slots (gst only)
├── sharedRules                # rule (id, name, hidden, publicationId, page)
│   └── description            # free-text rule body (universal mechanics)
├── sharedProfiles             # reusable Ability/Weapon profiles
├── sharedSelectionEntries     # reusable wargear / enhancements
├── sharedSelectionEntryGroups # reusable swap menus (Enhancements, Weapon Mods…)
└── selectionEntries
    └── selectionEntry  type ∈ {unit, model, upgrade, …}
        ├── costs / cost           # pts value etc.
        ├── categoryLinks          # keywords (primary=true marks the role)
        ├── profiles               # inline profiles
        │   └── profile  typeId→profileType.id
        │       └── characteristics
        │           └── characteristic name=… typeId=…  → text body (M, T, SV…)
        ├── infoLinks              # link to a shared rule / infoGroup
        ├── entryLinks             # link to a shared selectionEntry / group
        ├── selectionEntries       # nested children (models inside a unit)
        ├── selectionEntryGroups   # squad-composition / wargear swap menus
        ├── constraints            # min/max selections, scope=parent|force|roster
        ├── modifiers              # conditional mutations (hide, increment, etc.)
        └── comment                # free text, often used for detachment tags
```

Important conventions:
- **Every node has a four-segment hex `id`** (`xxxx-xxxx-xxxx-xxxx`). IDs are
  stable across revisions *as long as the volunteer doesn't recreate the node*.
  They are the only safe join key — names drift (typos, sentence-case fixes).
- **`targetId` references** resolve cross-file via the gameSystem id. A `cat`
  can reference shared content from any *library* `cat` it links to via
  `<catalogueLink>`.
- **`hidden`** is a viewer flag — librarian generally honours it.
- **`shared="true"`** on a constraint means "applies once across all instances".
- **`primary="true"`** on a `categoryLink` marks the displayable role keyword
  (Battleline, Character, etc.).

### Profile types currently in use (from `Warhammer 40,000.gst`)

| profileType | Characteristics |
|---|---|
| Unit | M, T, SV, W, LD, OC |
| Ranged Weapons | Range, A, BS, S, AP, D, Keywords |
| Melee Weapons | Range(default `Melee`), A, WS, S, AP, D, Keywords |
| Abilities | Description |
| Transport | Capacity |
| (faction-local) C'tan Powers, Triarch Abilities, … | varies |

Faction cats occasionally define **local** profile types (Necrons adds
`C'tan Powers`, `Triarch Abilities`). Librarian must read `profileTypes` from
every cat, not just the gst, when resolving `typeId`.

---

## 3. How Librarian already maps this (per SKILL.md)

- `weapons.py` walks every `.cat`, indexes weapon profiles by name + ability
  profiles by parent unit entry id. Dice expressions are averaged inline.
- `is_shared_rule` flag distinguishes inline `<profile typeName="Abilities">`
  from boilerplate pulled in from `<sharedRules><rule>` (different UI render).
- `UnitOptionsIndex` decodes four distinct BSData encodings of "pick from a
  swap group" (defaultSelectionEntryId, "none", group constraint max=1,
  defaultAmount-per-child Tau pattern). Documented in `gotchas.md`.
- Defensive stats (invuln, FNP, damage reduction, wounds override) are
  regex-extracted from ability `Description` characteristics in
  `stat_parsers.py`. Phase-split where the prose says "against ranged/melee".
- `rule_registry.py` is a curated override layer for things the regex layer
  can't parse cleanly (Doctrina Imperatives "OR" choices, etc.).

That existing pipeline is the consumer for any schema change here.

---

## 4. Quirks that *will* bite ingestion

These are observed in the current data and your `gotchas.md`/`leader_grants`/
`attachments` modules already paper over some of them:

1. **Name casing drift between BSData and Wahapedia.** Already handled via
   `wahapedia_options.py` casefolded joins.
2. **Per-model wargear inside one squad** (storm shields, Mistshield). Resolved
   by sub-splitting model groups on `(direct_keywords, bearer_text_signature)`.
3. **Ability prose carries mechanics.** Damage reduction, FNP, invuln, weapon
   keyword grants, leader auras — all only discoverable by regexing the
   Description characteristic. There is no structured field for "this is a 4+
   invulnerable save".
4. **"Bearer has Wounds N" patterns** must be parsed without over-crediting
   the whole squad. Two-layer design in `unit_stats.py`.
5. **Up-to-N leaders** (Cadian Shock Troops) is encoded in prose, not
   constraint XML. `attachments._max_leaders` greps it.
6. **Multi-mode weapons** (Plasma standard/supercharge) appear as multiple
   profiles under one `selectionEntry`; `ResolvedWeapon` holds a list.
7. **Detachment rules** live under faction cats in `infoGroup`s referenced by
   units via `infoLink type="infoGroup"`. Detachment name is on the
   `categoryEntry` / `selectionEntryGroup`.
8. **Dice strings (`D6`, `2D3+1`)** appear as raw text in numeric-looking
   characteristics (A, D, sometimes Range). Already averaged on ingest.
9. **`hidden=true` modifiers** can flip a unit's visibility depending on the
   detachment selected — points-cost modifiers also live as `modifier`s.
   Librarian today ignores these and reads the static `<cost>`.
10. **Volunteer typos** in keywords and ability text propagate quickly because
    everything is free text.

---

## 5. Should we ETL into a database?

**Recommendation: yes, build a daily ETL into a local SQLite (or DuckDB) cache
keyed by BSData node `id`s, and keep the raw XML as the system-of-record.**
Reasoning:

- Librarian already walks all 48 cats on startup. That's parse, index-build,
  regex, leader-merge, every cold start. A pre-computed table is faster and
  cheaper, and lets you ship a *deterministic snapshot* with each librarian
  release rather than chasing an upstream HEAD.
- ETL gives you a natural place to attach the curated overrides
  (`rule_registry`, attachment max-leaders, defensive-stat parses) so they're
  validated against the data at ingest time rather than at query time.
- A normalized store unblocks features that are awkward today: cross-faction
  search, weapon similarity, "which units have FNP 5+", arbitrary filters,
  diffing between BSData revisions.
- Single-writer / many-reader fits SQLite perfectly; no service to operate.
  DuckDB if you want analytical queries (joins across weapons × units).

Proposed shape (minimal, mirrors what `weapons.py` already builds in memory):

```
catalog_meta(catalog_id PK, name, revision, gameSystemRevision, ingested_at)
unit(entry_id PK, catalog_id, name, primary_category, raw_keywords JSON,
     m,t,sv,w,ld,oc, points, hidden, source_path)
model_group(group_id PK, unit_entry_id FK, label, count_min, count_max,
            stats JSON, is_character)
weapon_profile(profile_id PK, owning_entry_id, name, range_, attacks,
               skill, strength, ap, damage_avg, damage_raw, type,
               keywords JSON)
ability(profile_id PK, unit_entry_id FK, name, description, is_shared_rule,
        derived_mods JSON)        -- parsed FnP/invuln/etc cached here
rule(rule_id PK, catalog_id, name, description, publication, page)
info_link(src_id, target_id, kind)                  -- raw graph edge
entry_link(src_id, target_id, kind, flatten)
detachment(name PK, catalog_id, rules JSON)
swap_slot(unit_entry_id, encoding, slot JSON)       -- the four-encoding union
ingest_log(run_id, started_at, source_sha, …)
```

Keep `derived_mods` denormalized as JSON — the shapes diverge per ability and
you already model them as Python dataclasses in
`abilities.WeaponModification` / `stat_parsers`. Don't fight that.

What I would **not** put in the DB: the modifier/condition graph. It's only
useful at full-roster validation time, and the BSData runtime semantics are
non-trivial. Keep that querying against the XML.

### Refresh cadence
Daily is fine but probably overkill. The right trigger is upstream activity:
a GitHub Action (or cron in librarian) that:
1. `gh repo sync` your fork.
2. Computes `git rev-parse origin/main` and compares to the last ingested SHA.
3. If different, runs the ETL into a new SQLite file, then atomically swaps
   `librarian.db` → `librarian.db.next` → rename.
4. Posts a one-line summary (units added/removed, points changed, schema
   diffs) somewhere visible.

For local dev, ship the SQLite snapshot in the repo (or release artefact) so
contributors don't need to re-parse on first run.

---

## 6. Are transforms worth doing, or a waste?

Mostly yes, but with a strong "do not normalize prose" caveat.

**Worth doing at ETL time** (cheap, high payoff):
- Cache parsed defensive stats per unit (invuln, FNP, damage reduction, wounds
  override). Today these are regex'd on every render.
- Pre-compute attachments (`leader_id → led_id`) and persist alongside an
  override table so the UI's manual_pairs / blocked_ids can store cleanly.
- Pre-resolve `entryLink`/`infoLink`/`catalogueLink` graph into flat
  `(src_id, target_id, kind)` rows. Saves all cross-file lookups at query time.
- Average dice expressions once; keep `damage_raw` next to `damage_avg`.
- Normalize keyword lists (split, trim, dedupe, casefold) into a join table.
  This is where typos like `"Lethal Hits "` (trailing space) bite.
- Stamp every row with `source_sha` so you can diff two ingests.

**Worth doing but harder** (still recommended):
- A "structured ability" table that links each ability `Description` text to
  the inferences `stat_parsers.py` makes from it, plus a `confidence` and a
  `human_override` slot. This is the place to *gradually* migrate prose into
  structure without forking from upstream.
- A detachment → applicable-units materialization. The current path through
  `infoGroup` references is correct but slow to traverse interactively.

**Not worth doing** (will rot):
- Rewriting ability prose into your own DSL. You'll be chasing volunteer
  edits forever and your "improvements" can't be PR'd back upstream because
  BSData's runtime semantics are tied to BattleScribe's modifier engine.
- Inventing new IDs. Stick to BSData ids; if you must mint, prefix
  (`librarian:<sha>:…`) so they never collide.

---

## 7. Schema-change detection

The XML schema rarely changes, but the *shape of usage* does — new
`profileType`s appear (C'tan Powers), new attributes get added to existing
elements, new `categoryEntry`s show up. A volunteer can also reshape a unit
(e.g. split a squad into per-model wargear) without changing the XSD.

Two layers:

**A. XSD-level drift (rare, breaking).**
- Pin a copy of `BSData/schemas/{catalogue,gameSystem}.xsd` in librarian.
- Validate every cat on ingest with `lxml.etree.XMLSchema`.
- If validation fails: **stop the swap, keep the previous DB, alert.**
- Periodically (weekly cron) fetch the latest XSDs and `diff` against the
  pinned copy. Any diff = ticket.

**B. Usage drift (common, often silent).**
This is the one that actually trips the librarian pipeline. Run a structural
fingerprint at ingest:

```python
fingerprint = {
  "profile_types":  sorted({(pt.name, tuple(c.name for c in pt.chars))}),
  "category_entries": sorted({ce.name for ce in cats}),
  "selection_entry_attrs": sorted(seen_attrs_on(selectionEntry)),
  "characteristic_names_by_profile_type": {...},
  "rule_attrs": sorted(...),
  "shared_rule_count_by_catalog": {...},
  "constraint_field_values": sorted(...),
  "modifier_types": sorted(...),
}
```

Diff `fingerprint(current)` vs `fingerprint(last_good)` and surface:
- New profileType → likely needs a `weapons.py` adapter and a profile-type
  whitelist update.
- New characteristic name under a known profileType → may need a parser hook
  (e.g. GW adds a "Movement Cap" field).
- New `constraint.field` value → swap-encoding catalog needs an entry.
- New `categoryEntry` → check faction map.
- Drop or rename → potentially a unit went missing in the latest data.

This catches the silent-failure case where a unit suddenly stops appearing
because its `profileType` changed name from "Unit" to "Unit Profile" or
similar. It also catches additions early enough to write a rule-registry
override before a release.

Suggested implementation: ~150 lines of Python, runs after the daily sync,
writes `schema_fingerprint.json` next to the DB and a diff comment to the
sync action. Wire it as a hard fail if the diff exceeds a threshold (say,
any profileType/characteristic add) so silent shape changes can't ship
unreviewed.

---

## 8. Concrete next steps (in order)

1. Vendor `catalogue.xsd` + `gameSystem.xsd` from `BSData/schemas` into
   `librarian/data/bsdata_schemas/`. Validate on ingest.
2. Build the SQLite ETL (~200 lines on top of existing `weapons.py` walker).
   Reuse the dataclasses; the ETL is just "walk → dump rows".
3. Add the fingerprint diff step; fail the action on profileType /
   characteristic add/remove until acknowledged.
4. Move startup reads in `src/profile/` to query the DB (keep XML fallback
   for one release as a safety net).
5. Schedule the GitHub Action: `gh repo sync` → if SHA changed → ETL →
   fingerprint diff → publish artefact. Daily is fine; on-demand via
   `workflow_dispatch` is better.

Estimated effort: 1-2 evenings for the ETL + fingerprint, half a day to wire
it into librarian, plus ongoing rule-registry maintenance which you already
do.
