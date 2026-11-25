---
name: inpaas-entity-management
description: Create or change inPaaS database entities, tables, fields, foreign keys, indexes, and their metadata. Use for schema changes in an inPaaS project; do not use for ordinary source or form-only edits.
---

# Inpaas Entity Management

Use this skill whenever the request creates a table/entity or adds, removes, renames, or changes an entity field, reference, index, primary key, domain, or label.

## Before changing anything

1. Read the project `AGENTS.md`, the development-kit `AGENTS.md`, and `entity-management.md` in the shared inPaaS knowledge base.
2. Download or inspect the current `.entity.json` before proposing a change. Preserve its existing PK, auditing, ownership, aliases, references, indexes, and metadata unless the request explicitly changes them.
3. Determine whether the table is a product resource or an Espec/customization. For an Espec table, propose the name and obtain explicit approval before creation:
   - independent table: `CST_{DESCRICAO}`;
   - child table: `CST_{TABELA_PAI_SEM_PREFIXO}_{DESCRICAO}`.
4. If the requested name diverges from that convention, stop and obtain explicit confirmation of the exception. Record the confirmed exception in the project instructions when they require it.

## Model fields safely

- Follow an equivalent existing entity in the same project for physical name, alias, type, size, `required`, audit and title. Ask for an example when those details cannot be inferred safely.
- External or corporate integration codes use `DS_INTEGRATIONCODE` with alias `integrationcode`.
- Ordinary text uses `VARCHAR`, a `DS_` physical name and a lowercase alias without the prefix.
- Keep `searchable="false"` unless full-text search is explicitly required. Do not infer full-text indexes from ordinary SQL filtering needs.
- Add foreign keys and uniqueness constraints only when the business cardinality requires them. For a one-to-one child record, use a unique index on the parent FK.
- For field removal, rename, data-type narrowing, non-null conversion, PK/FK changes, or any other potentially destructive schema change, first show the exact impact and obtain explicit approval.

## Publish and validate

- Local `.entity.json` editing or downloading does not publish anything. Never use SQL to repair `OBJETOBD`, `CAMPOBD`, labels, constraints, or physical schema.
- Do not publish an entity, create DDL, or alter remote metadata unless the user explicitly authorizes that operation. Use the official entity publication flow only.
- Before publication, validate the JSON and compare the intended diff against the downloaded model. After an authorized publication, re-download the entity and verify the relevant fields, references, indexes, and labels.
- If forms or sources depend on the changed field, inspect their bindings and code. Do not publish those resources unless the request includes them.
