# GeoChem Ontology v1.2.1 — Changes

Three incremental, additive-only changes to `geochem_v1.2.1.ttl`/`geochem_v1.2.1.shacl.ttl` (renamed from `geochem_v1.2.0.ttl`/`geochem_v1.2.0.shacl.ttl`), from the post-meeting follow-up tracked in `ta2-minmod-kg` issue #18/#19. `owl:versionInfo` bumped `1.2.0` → `1.2.1` in both files. None of these change or restrict any existing property — nothing that already conforms to v1.2.0 stops conforming to v1.2.1.

All three are already implemented, tested, and live in `ta2-minmod-kg` (`geochem/sample-backend`, PR #103) — this is the ontology-side record of that same work.

---

## 1. Soft delete: `:is_deleted`/`:deleted_by`/`:deleted_at`

**Added**, domain `owl:unionOf (:Sample :Analysis :Element mo:MineralSite)`:
- `:is_deleted` (`xsd:boolean`) — soft-delete flag. The record is never removed; this just marks it inactive. Setting it back to `false` undeletes — there's no separate delete/undelete verb, it's a normal field like any other.
- `:deleted_by` (`xsd:anyURI`) — MinMod user URI of who set `:is_deleted`, server-derived from the session, never client-supplied. Same convention as `:EditEvent`'s `:updated_by`.
- `:deleted_at` (`xsd:dateTime`) — timestamp `:is_deleted` was last changed.

**Why dedicated fields, not derived from `:edit_history`:** `mo:MineralSite` has no edit-history mechanism at all (its own `mo:created_by` is a single overwritten field, by design — see `edit_history_provenance.md`'s rationale for why per-edit history was rejected for it). Dedicated fields keep "who/when deleted" queryable uniformly across all four classes regardless of which have richer history.

**Why all four classes in one union, not per-class properties:** HMI's own delete action covers `deposit`/`sample`/`analysis`/`element` (confirmed against the real curation UI) — same three fields, same semantics, at every level that's actually deletable through it. `method`-level delete stays out of scope: there's no shared `:AnalyticalMethod` class to hang a delete flag on yet (`:Analysis` still carries its method fields inline — see the open item in `geochem_vs_minmod.md`/issue #19).

---

## 2. Sample-level location: `mo:location_info`'s domain widened to `:Sample`

**Before:** `mo:location_info`'s domain was `mo:MineralSite` only (via `rdfs:subPropertyOf mo:mineral_site_objectproperty`).

**After:** domain is `owl:unionOf (mo:MineralSite :Sample)`. `mo:LocationInfo` itself is unchanged and reused as-is — no new location class, no bespoke lat/lon pair.

**Why:** a sample can be reported at a more precise point than its parent site (e.g. a specific drill-hole collar within a larger deposit polygon) — this records that difference without duplicating or overriding the site's own location. Same technique already used for `mo:Reference` (widened to `:Sample ∪ :Analysis ∪ :Element` in the per-field provenance extension) — a precedent for widening an existing MinMod property's domain onto GeoChem classes rather than inventing a parallel property.

**Deliberately not added:** a flattened/indexed copy analogous to `MineralSite`'s `location_view` (used for bounding-box queries). No query need for that at the sample level has come up yet; add one if it does.

---

## 3. `:strat_unit_name`, independent of `:strat_unit_uid`

**Added:** `:strat_unit_name` (`xsd:string`, domain `:Sample`) — the stratigraphic unit as reported in the source, free text.

**Why a new property, not a repurposing of `:strat_unit_uid`:** `:strat_unit_uid` is a normalized identifier (already declared as such); a source's own free-text stratigraphic-unit label is a different kind of value, not a raw form of the same field. `:strat_unit_uid` is untouched — this is a pure addition alongside it, not a rename or conversion.

---

## Verification

All three changes were exercised end-to-end against the real `ta2-minmod-kg` API (native Postgres, no Docker) before being committed here: `is_deleted` set/cleared via `POST /papers/publish` (sample/analysis/element) and `PUT /mineral-sites/{id}` (deposit), with `deleted_by`/`deleted_at` confirmed to stamp and clear correctly; `location` and `strat_unit_name` patched and round-tripped through `GET`. Also validated with `pyshacl` (`inference="rdfs"`, both the ontology and shapes graphs loaded) as part of wiring real SHACL validation into `/papers/publish` — none of the three additions produce new SHACL violations against existing conforming data, since no shapes constrain them yet.
