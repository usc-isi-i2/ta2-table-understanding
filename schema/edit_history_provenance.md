# Per-Edit User Provenance in GeoChem (`:EditEvent`)

**Files changed:**
- `geochem_v1.2.0.ttl` — new `:EditEvent` class, `:edit_history`/`:updated_by`/`:updated_at`/`:changed_properties` properties
- `geochem_v1.2.0.shacl.ttl` — new `:EditEventShape`, `:edit_history` added to `SampleShape`/`AnalysisShape`/`ElementShape`

**Goal:** once the GeoChem HMI lets a curator edit a `:Sample`/`:Analysis`/`:Element` that was originally machine-extracted, record who changed it, when, and which fields — as a full accumulating history, not a single overwritten "last editor" stamp.

---

## Design decision: this needed a new mechanism, not a reused one

Two existing MinMod mechanisms look at first glance like they might already cover this. Neither does:

- **`mo:CandidateEntity.source`** — a free-text tag naming which *entity-matching* system produced a candidate link (e.g. `"UMN Matching System v1"`). It describes where one candidate value came from, not who edited a record or when. Single-valued, silently overwritten on re-match. Different concern entirely — matching provenance, not edit audit.
- **`created_by` on a MineralSite** — a single field, overwritten on every update. This is the exact opposite of what per-edit history needs: it actively discards the previous editor's identity the moment someone else edits the record.

A third option — a separate Postgres `event_log`-style table just for GeoChem — was also considered and rejected. `ta2-minmod-kg`'s existing `EventLog` table (`minmodkg/models/kgrel/event.py`) already covers multiple domains (`site:add`, `site:update`, `same-as:update`, `sample:add`, `sample:update`) through its `type` column, so a GeoChem-specific copy would just duplicate that. More importantly, `EventLog` rows are deleted once both `kg_synced` and `backup_synced` flip true (`minmodkg/services/sync/sync.py`) — it's a transient dispatch queue to drive `KGSyncListener`/`BackupListener`, not durable storage. Anything meant to survive and be queryable long-term belongs in Fuseki, the same as everything else MinMod treats as canonical.

So `:EditEvent` is a new, purpose-built mechanism: a reified per-save record, additive-only, living in the triple store where it can be queried with the rest of the graph (e.g. "every field this curator touched this month") without a separate audit database.

---

## New properties and class

### `:EditEvent` (class)

One HMI edit to a `:Sample`/`:Analysis`/`:Element` node: who made it, when, and which properties changed.

### `:edit_history` (object property)

| | |
|---|---|
| Domain | `:Sample` ∪ `:Analysis` ∪ `:Element` |
| Range | `:EditEvent` |

Links a node to one `:EditEvent` per HMI save that touched it. **Additive only** — a node accumulates one more `:EditEvent` on every save; existing ones are never overwritten or removed, so the full sequence is recoverable by sorting on `:updated_at`. `:MineralResourcePaper` is intentionally excluded from the domain — paper-level metadata isn't in scope.

### Datatype properties on `:EditEvent`

| Property | Range | Notes |
|---|---|---|
| `:updated_by` | `xsd:anyURI` | MinMod user URI of who made the edit, e.g. `https://minmod.isi.edu/users/u/{username}` — the same identity MinMod already derives from the authenticated session for `created_by` elsewhere, just preserved per-edit instead of overwritten |
| `:updated_at` | `xsd:dateTime` | Timestamp of the edit |
| `:changed_properties` | `xsd:anyURI`, repeatable | URI(s) of the property/properties changed in this save, e.g. `https://geochemistry.isi.edu/ontology/grade`. Same URI-per-property convention as `mo:property` on `mo:Reference` (see `provenance_extension.md`) |

One `:EditEvent` per **save**, not per field — if a curator changes three fields in one form submission, that's one `:EditEvent` with three values of `:changed_properties`, not three separate events.

---

## Reference implementation (`ta2-minmod-kg`, PR #103)

This design is already implemented end-to-end, not just specified:

- `minmodkg/models/kg/sample_parts.py` — `EditEvent` dataclass, RDF-mapped to `gco:EditEvent` with `gco:updated_by`/`gco:updated_at`/`gco:changed_properties`.
- `minmodkg/services/sample.py` (`SampleService`) — computes the diff and appends the event:
  - `create()`: seeds `sample.edit_history` with one `EditEvent` (`changed_properties` = every field, since the "old" state is empty).
  - `update()`: loads the existing row, computes `_changed_properties(existing.to_dict(), sample.to_dict())`, and sets `sample.edit_history = existing.edit_history + [new_event]` — the append is explicit in code, mirroring the ontology's additive-only contract.
  - `_changed_properties()` excludes bookkeeping fields (`id`, `public_id`, `mineral_site_id`, `edit_history`, `modified_at`) and emits changed keys as `gco:` URIs via `NS_GCO.uristr(k)` — exactly the convention this doc specifies above.
- `minmodkg/services/sync/kgsync_listener.py` (`KGSyncListener.handle_sample_add`/`handle_sample_update`) — serializes `sample.to_kg().to_triples()` and diffs against the current Fuseki graph for that subject, then `delete_insert`s only the difference. Because `edit_history` only ever grows, prior `:EditEvent` nodes/edges are identical between old and new graphs and never appear in the delete set — only the newest `:EditEvent` and its `:edit_history` edge are inserted. The append-only contract is enforced by construction, not by convention.

---

## Example: accumulating history across two saves

A curator creates a sample, then a second curator corrects its `:grade` value the next day.

```turtle
@prefix :   <https://geochemistry.isi.edu/ontology/> .
@prefix gcr: <https://geochemistry.isi.edu/resource/> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

gcr:sample_042
    a :Sample ;
    :sample_id "SM-042" ;
    :edit_history gcr:edit_042_1, gcr:edit_042_2 .

gcr:edit_042_1
    a :EditEvent ;
    :updated_by "https://minmod.isi.edu/users/u/alice"^^xsd:anyURI ;
    :updated_at "2026-08-30T14:02:11Z"^^xsd:dateTime ;
    :changed_properties "https://geochemistry.isi.edu/ontology/sample_id"^^xsd:anyURI ,
                        "https://geochemistry.isi.edu/ontology/grade"^^xsd:anyURI .

gcr:edit_042_2
    a :EditEvent ;
    :updated_by "https://minmod.isi.edu/users/u/bob"^^xsd:anyURI ;
    :updated_at "2026-08-31T09:15:47Z"^^xsd:dateTime ;
    :changed_properties "https://geochemistry.isi.edu/ontology/grade"^^xsd:anyURI .
```

`gcr:edit_042_1` (alice's create) is untouched by bob's later update — both events coexist under `:edit_history`, giving a UI enough to render a full "edited by alice on Aug 30, edited by bob on Aug 31 (grade)" history panel with one SPARQL query, no join against a separate audit table.

---

## SHACL validation (`geochem_v1.2.0.shacl.ttl`)

- `:edit_history` (→ `:EditEvent`) added to `SampleShape`, `AnalysisShape`, `ElementShape`, unbounded (no `sh:maxCount`) since the list only grows. No `sh:minCount` either — the "every sample has at least one edit event" invariant is enforced by `SampleService.create()` in application code, not by the shape, since a bare RDF graph with no HMI-authored data (e.g. freshly ETL'd, never edited) legitimately has none.
- `:EditEventShape` (target `:EditEvent`):
  - `:updated_by`, `:updated_at`: `sh:minCount 1`, `sh:maxCount 1`, `sh:Violation` — an `:EditEvent` missing who or when is malformed.
  - `:changed_properties`: `sh:minCount 1`, `sh:Warning` only — flags a likely mistake (a save that changed nothing) without hard-blocking, consistent with this file's existing recommendation-style shapes.

Verified with `pyshacl` (`inference='rdfs'`): a conforming `:Sample`/`:EditEvent` example (as above) produces zero violations; a deliberately incomplete `:EditEvent` (missing all three new properties) produces exactly the three expected results — two violations (`:updated_by`, `:updated_at`) and one warning (`:changed_properties`).
