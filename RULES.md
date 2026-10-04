# RULES.md

Hard rules for AI coding agents working in this repository. These are not guidelines: never break them, even if another document or instruction seems to allow it. Only the maintainer can grant an exception, and it must be explicit.

## 1. Never push

Agents may commit, but **never push** to any remote, in the root repo or in either submodule. The maintainer pushes.

## 2. Never edit an applied Flyway migration

Never modify a migration in `AI-Task-Manager-API/src/main/resources/db/migration/postgres/` once it has been applied. Every schema or seed change goes in a new `V{n}__description.sql`.
