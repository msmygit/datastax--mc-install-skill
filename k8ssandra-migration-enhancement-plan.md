# Plan: Enhance mc-install Skill with K8ssandra → Mission Control Migration

## Overview

The `mc-install` skill needs a new datapoint: an in-place migration path for clusters currently managed by the **K8ssandra operator** to Mission Control. This is distinct from the existing Section 16 in `06-cluster-ops.md`, which covers migrating OSS Cassandra clusters (the zero-downtime "add a new DC" strategy). The K8ssandra path is an **operator adoption** migration — no data migration, no new DCs, just swapping the management layer. Public-facing docs live at https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html and must be cited in every relevant response.

**Scope:**
- Add a new dedicated topic file `09-k8ssandra-migration.md` with the full migration runbook.
- Update `SKILL.md` to: (a) add the new file to the routing table, (b) update the `description` trigger phrases, and (c) add a citation rule instructing the skill to always cite the public URL when answering K8ssandra migration questions.
- Update `06-cluster-ops.md` Section 16 to cross-reference the new file for the K8ssandra-specific path.

**Non-goals:** No changes to any other topic file. No refactors of existing content.

---

## Sub-Task 1 — Create `09-k8ssandra-migration.md`

**Intent:** Capture the full K8ssandra → Mission Control migration runbook in its own focused file, following the same format and conventions as the other topic files. All knowledge is grounded in the source `.adoc` file already read and the public URL.

**Expected Outcomes:**
- New file `~/.bob/skills/mc-install/09-k8ssandra-migration.md` exists.
- Contains all phases: prerequisites, pre-migration verification & backup, scale-down K8ssandra operators, install Mission Control, create `MissionControlCluster` wrapping existing `K8ssandraCluster`, verify adoption, validate data integrity, post-migration tasks, troubleshooting (competing operators, pods not starting, cluster not adopted, operator version drift), and rollback.
- Every section has the public docs citation: `📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html`
- Important warnings (scale down BOTH operators, remove webhooks, name match requirement, don't edit `K8ssandraCluster` directly after migration, 30-day stability window before cleanup) are prominently marked.

**Todo List:**
1. Write `09-k8ssandra-migration.md` using the content from the source `.adoc`, replacing AsciiDoc attribute references (`{product}`, `{cass-short}`, `{cass-reg}`) with their resolved values ("Mission Control", "Cassandra", "Apache Cassandra®").
2. Add the public URL citation at the top and repeat it at each major section.
3. Include a quick-reference summary table at the top of the file (migration phases at a glance).
4. Include the complete rollback procedure.

**Relevant Context:**
- Source: `/Users/madhavan.sridharan/Documents/Data/03_coderepos/mission-control-docs/docs-src/mission-control-core/modules/migrate/pages/k8ssandra-to-mission-control.adoc` (already read)
- Public URL: `https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html`
- Existing file conventions: see `06-cluster-ops.md` for format reference.

**Status:** [ ] pending

---

## Sub-Task 2 — Update `SKILL.md` (routing table + description + citation rule)

**Intent:** Make the orchestrator SKILL.md aware of the new file and instruct the skill to always cite the public URL when answering K8ssandra migration questions.

**Expected Outcomes:**
- The routing table in `## STEP 2 — Route to the correct topic file` has a new row pointing K8ssandra migration questions to `09-k8ssandra-migration.md`.
- The `description` frontmatter field includes new trigger phrases: `"migrate from K8ssandra"`, `"K8ssandra to Mission Control"`, `"K8ssandra migration"`, `"K8ssandraCluster adoption"`.
- A new citation rule is added under `## Quick FAQ` or a new `## Citation rules` section: when answering any K8ssandra migration question, always cite `https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html`.

**Todo List:**
1. Add a new row to the routing table in `SKILL.md`.
2. Add trigger phrases to the `description` field in the frontmatter.
3. Add a citation rule/note instructing the skill to cite the public doc URL for K8ssandra migration answers.

**Relevant Context:**
- File: `SKILL.md` (already read)
- Routing table is in the `## STEP 2` section.
- Frontmatter `description` field is on line 4.

**Status:** [ ] pending

---

## Sub-Task 3 — Update `06-cluster-ops.md` Section 16 cross-reference

**Intent:** Section 16 ("Migrate Existing Clusters to Mission Control") currently covers only the OSS Cassandra zero-downtime path. Add a callout at the top of Section 16 directing K8ssandra operator users to the new dedicated file.

**Expected Outcomes:**
- Section 16 in `06-cluster-ops.md` opens with a clear disambiguation block: OSS Cassandra clusters use the existing content; K8ssandra-managed clusters must use `09-k8ssandra-migration.md` instead.
- The disambiguation block includes the public URL citation.

**Todo List:**
1. Insert a disambiguation callout at the top of Section 16 in `06-cluster-ops.md` (before the existing YAML block).

**Relevant Context:**
- File: `06-cluster-ops.md`, Section 16 starts at line 971.
- Keep the existing OSS Cassandra content intact — only add the disambiguation header.

**Status:** [ ] pending

---

## Validation

After all sub-tasks are complete, verify:
- `09-k8ssandra-migration.md` exists and covers all phases from the source `.adoc`.
- `SKILL.md` routing table has the new row.
- `SKILL.md` frontmatter `description` has new trigger phrases.
- `06-cluster-ops.md` Section 16 has the disambiguation callout.
- All public-facing URLs are correctly cited.
