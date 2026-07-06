# BK-20: TMS-ATC Search | Search and autocomplete ATCs

**Ticket:** BK-20 | **Module (= Epic):** BK-13 ATC Library (Atomic Test Components) | **Status:** Shift-Left QA | **Sprint:** n/a — pre-sprint

## Acceptance Criteria (original)

- AC1: Find an ATC by a word in its title
- AC2: Find an ATC by one of its tags
- AC3: Narrow results to a Module subtree
- AC4: More recently updated ATCs rank higher among equal matches
- AC5: An empty query runs no search
- AC6: Results never include ATCs from another workspace

## Team Discussion (from comments)

- [Ely] (2026-05-19): Architect Annotation — full technical spec for search_tsv column, GIN index, trigger, ranking SQL, module subtree filter, workspace scoping, and API shape. Response: `{atc_id, slug, title, module_path, layer, status_dot}`.
- BK-18 PO confirmed (2026-05-27): `atc.created`/`atc.updated` events will be consumed by BK-20 (search index) in Wave 2.

## Parent epic

BK-13: ATC Library (Atomic Test Components)

## Pre-sprint status

Shift-Left refinement: in progress (started 2026-06-01)

## Upstream dependency

BK-18 (TMS-ATC API) is in Ready For Dev — atcs table schema is contractually defined via its Shift-Left QA pass (2026-05-27). Columns required by BK-20 (`title`, `tags`, `updated_at`, `module_id`, `workspace_id`) are confirmed. BK-20 will ADD `search_tsv tsvector` + GIN index + trigger via its own migration.
