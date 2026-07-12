# Contribution [2]: Grant Read Access on Chronon-Created Tables

**Contribution Number:** [2]
**Student:** [Alice]
**Issue:** [airbnb/chronon#1033](https://github.com/airbnb/chronon/issues/1033)
**Status:** Phase I — Issue Selected

---

## Why I Chose This Issue

[#why-i-chose-this-issue](#why-i-chose-this-issue)

I chose `airbnb/chronon` for my second cycle — it's a real, actively used open-source feature platform for ML (used in production at Airbnb, Stripe, Netflix, and others), which meant it had a maintained `CONTRIBUTING.md`, code style guide, and PR review process rather than an abandoned repo. Within the open issues, I compared three candidates before settling on this one:

- **#1033 (chosen)** — a maintainer (`pengyu-hou`) had already replied to the issue, agreed the use case made sense, and sketched the shape of the implementation (a `grantPublicAccess` method in `TableUtils.scala`, following the existing `alterTableProperties` pattern, gated by a new Spark conf flag). That's a strong signal this is a real, wanted change and not something that will stall in review.
- **#934** (runner-up) — a hardcoded Iceberg partition column bug in the same file, but no maintainer reply yet, so the fix direction is unvalidated.
- **#999** — a Python/Bazel test-collision bug with a clear root cause already diagnosed, but it requires standing up Bazel specifically, which is extra environment overhead on top of learning the repo.

I picked #1033 because the maintainer engagement lowers my risk of building the wrong thing, the scope is contained to one file (`spark/src/main/scala/ai/chronon/spark/TableUtils.scala`), and it's a genuinely useful feature (tables Chronon creates are currently unreadable by other users/services like Jupyter or Superset without manual permission fixes).

---

## Understanding the Issue

[#understanding-the-issue](#understanding-the-issue)

### Problem Description

[#problem-description](#problem-description)

When Chronon creates tables (via Spark/Hive), default permissions restrict read access to the creator only. Other users and downstream services (Jupyter, Superset, other jobs) can't read the tables without someone manually granting access afterward.

### Expected Behavior

[#expected-behavior](#expected-behavior)

A new configurable Spark conf flag, `chronon.table_write.grant_public_access` (default `false`), should let Chronon automatically issue a `GRANT SELECT` statement right after a table is created, so downstream consumers can read it without manual intervention.

### Current Behavior

[#current-behavior](#current-behavior)

No such flag or grant logic exists. Table creation paths in `TableUtils.scala` don't touch permissions at all — access has to be fixed up manually after the fact.

### Affected Components

[#affected-components](#affected-components)

- `spark/src/main/scala/ai/chronon/spark/TableUtils.scala` — where table creation happens, and where the new `grantPublicAccess` method (and its call site right after `createTableSql` succeeds) will live. The existing `alterTableProperties` method (~L816) is the pattern to mirror.

### Open Design Question (from the maintainer)

`pengyu-hou` asked directly in the issue thread: does granting to `PUBLIC` work for the reporter's use case, or does this need to support a configurable role instead of a hardcoded `PUBLIC` target? This needs to be resolved — either by proposing a configurable-role design in my Phase II plan, or by explicitly scoping to `PUBLIC` only for v1 and noting the extension point. I'll address this explicitly when I post my plan as a comment on the issue.

---

## Reproduction Process

[#reproduction-process](#reproduction-process)

*Not yet started — Phase II.* This isn't a bug with a failing repro; it's a missing feature, so "reproducing" means confirming current behavior (table created with default/creator-only permissions, no grant issued) against the local dev environment once it's set up.

---

## Solution Approach

[#solution-approach](#solution-approach)

### Analysis

Not yet started in depth — pending local environment setup and a close read of `TableUtils.scala` around `createTableSql` and `alterTableProperties`.

### Proposed Solution (draft, subject to Phase II plan)

Add a `grantPublicAccess(tableName: String)` method to `TableUtils.scala`, following the maintainer's sketch: read a new `chronon.table_write.grant_public_access` Spark conf flag (default `false`); if enabled, issue `GRANT SELECT ON TABLE <table> TO PUBLIC` (or a configurable role, pending the open design question above) immediately after table creation succeeds; catch and log-only on failure so a grant failure never blocks the table-creation job itself.

### Implementation Plan

Not yet written — this is the deliverable for Phase II ("Reproduce and Plan"). It will cover: exact files touched, whether the target role is hardcoded `PUBLIC` or configurable, the test(s) to add (regression/behavior test per `CONTRIBUTING.md`'s testing requirements), and the plan will be posted as a comment on issue #1033 before I start writing code, so the maintainer can redirect me early if needed.

---

## Testing Strategy

[#testing-strategy](#testing-strategy)

*To be defined in Phase II.* Per `airbnb/chronon`'s `CONTRIBUTING.md`, new functionality needs test coverage — likely a unit test on `TableUtils` confirming the grant SQL is issued (or not) based on the conf flag, using the existing `DataFrameGen` fuzzing helper if table data needs to be synthesized.

---

## Implementation Notes

[#implementation-notes](#implementation-notes)

### Week 1 Progress

[#week-1-progress](#week-1-progress)

- Compared `airbnb/chronon` (canonical repo, 1k★, active `CONTRIBUTING.md`) against the `zipline-ai/chronon` commercial fork; chose `airbnb/chronon` for stronger community/maintainer engagement.
- Shortlisted and compared issues #1033, #934, #999, #988; selected **#1033** based on existing maintainer buy-in.
- Read `CONTRIBUTING.md`, `Code_Guidelines.md`, and `devnotes.md` to understand PR conventions, code style (hot-path vs. control-path rules — doesn't apply much to `TableUtils.scala`), and local dev setup (Bazel or SBT, Thrift 0.13 pinned).
- **Still open (TODO, blocking Phase I completion):** fork `airbnb/chronon` to my account, comment on issue #1033 to signal intent to work on it, confirm no one else has since claimed it.

### Code Changes

[#code-changes](#code-changes)

None yet — Phase I only.

---

## Pull Request

[#pull-request](#pull-request)

**PR Link:** Not yet opened.

**Status:** Phase I — issue selected, environment and fork not yet set up.

---

## Learnings & Reflections

[#learnings--reflections](#learnings--reflections)

### Technical Skills Gained

[#technical-skills-gained](#technical-skills-gained)

[To fill in as I go]

### Challenges Overcome

[#challenges-overcome](#challenges-overcome)

[To fill in as I go]

### What I'd Do Differently Next Time

[#what-id-do-differently-next-time](#what-id-do-differently-next-time)

[To fill in as I go]

---

## Resources Used

[#resources-used](#resources-used)

- [airbnb/chronon CONTRIBUTING.md](https://github.com/airbnb/chronon/blob/main/CONTRIBUTING.md)
- [airbnb/chronon devnotes.md](https://github.com/airbnb/chronon/blob/main/devnotes.md)
- [airbnb/chronon Code Guidelines](https://chronon.ai/Code_Guidelines.html)
- [Chronon Getting Started docs](https://chronon.ai/Getting_Started.html)
- [Issue #1033](https://github.com/airbnb/chronon/issues/1033)
