---
name: strapi-seed-content
description: Seeds or removes test data in a Strapi CMS through its REST API for any content types (collection types, single types, relations, components, dynamic zones, media uploads). The agent asks which action, instance, token setup, content types, entry counts, images, and publish state, writes the content as JSON, and runs idempotent Node scripts. Use whenever the user wants dummy, test, sample, mock, or bulk content in Strapi, wants to populate or reset a CMS for frontend testing, or wants to unseed, clean up, or delete previously seeded data, even if they do not say "seed".
---

# Strapi content seeding

Seed or unseed test data in a Strapi instance for ANY content types.
The agent writes the content itself as JSON files. Scripts only send REST requests
to Strapi. The only credential needed is a Strapi API token. No LLM API key.

References (read when you reach the step that mentions them):

- `references/data-format.md`: JSON conventions for data files and `seed.config.json`
- `references/script-templates.md`: scripts to copy into the project and adapt

## Guardrails (always)

- Default target is localhost. For any other URL, stop, name the host back to the
  user, and proceed only after they explicitly confirm that exact URL. Treat anything
  that looks like production as off-limits unless the user insists in plain words.
- Never modify schemas, roles, permissions, plugins, or existing entries.
- Never print, echo, log, or put the token in a command line. It lives only in
  `scripts/seed/.env`. If the user pastes a token into chat, write it to `.env`,
  do not repeat it, and suggest deleting that token in Strapi when finished.
- Add `scripts/seed/.env` and `scripts/seed/.seed-state.json` to `.gitignore`.
- Do not guess field names, routes, or relation types. Read the schemas.
- On a 400, 401, or 403 response, stop and report the error body. No blind retries.
- Unseed deletes only what the seed state file (or, with explicit approval, the keys
  in the data files) identifies. Never delete "everything" in a content type.

## Intake A: ask before doing anything

Ask these together in ONE message (use the agent's question UI if it has one).
If the user says "use defaults", restate the defaults and get a yes.

1. Action: seed, unseed, reseed (unseed then seed), or upload media only?
2. Strapi URL? Default `http://localhost:1337`. Is it a local dev instance?
3. API token: has one been created (Settings > API Tokens)? Explain what it needs
   (see "Token requirements") and ask them to put it in `scripts/seed/.env` as
   `STRAPI_TOKEN=...` and reply "done". Never ask them to paste it in chat.
4. Who runs the scripts: the agent, or the user in their own terminal?

## Discovery (read only)

- Detect the Strapi version: read the `@strapi/strapi` version in `package.json` and
  note the major number. It goes in `seed.config.json` as `strapiVersion`. Id handling and
  draft/publish query parameters differ between major versions, so check the official
  REST API docs for that version before adapting the scripts.
- List content types in `src/api/*/content-types/*/schema.json` (also check
  `src/components/**` for components used).
- For each type report: display name, kind (collection or single), route (plural for
  collection, singular for single), required fields, unique/`uid` fields, field types,
  relations (target and cardinality), media fields (single or multiple, allowed types),
  components and dynamic zones, whether draft and publish is on, whether i18n is on.
- Do not call the Strapi API yet beyond a reachability check.

## Intake B: choices, after discovery

Show the discovered types as a table, then ask:

5. Which content types to target? (Offer "all", or a pick-list. Single types too?)
6. Entries per collection type? Default 10 each. Single types get one entry.
7. Relations: for each detected relation, seed the related type too (default), or
   link to entries that already exist? For existing entries the agent fetches ids with
   GET and writes them into the data.
8. Components and dynamic zones: fill required ones (default) and optional ones?
9. Images and files:
   - Add media at all? yes or no (default no).
   - Which media fields? (list the detected ones)
   - Source: (a) a folder the user provides, (b) agent-generated local placeholder
     images (simple SVG or PNG files), or (c) download from a placeholder image
     service (only with the user's OK, since network access may be blocked).
   - How many distinct images? Default 6, reused round-robin.
   - Alt text: generate from file names (default) or none?
10. Publish state (only if draft and publish is on): published (default) or drafts?
11. Locales (only if i18n is on): which locales?
12. Content theme: what is the site or app about, preferred language and tone?
    (Default: realistic content in English that fits the site's purpose.)
13. For unseed: all seeded content (default), only some types, also uploaded media,
    reset single types?

## Plan checkpoint

Before writing anything, show a plan table and wait for approval:

| Order | Content type | Kind | Method and route | Count | Media | Relations |
| ----- | ------------ | ---- | ---------------- | ----- | ----- | --------- |

Plus: action, target URL, publish state, locales, and who runs the scripts.
For unseed, list exactly which types, how many ids are in the state file, and whether
media and single types are included.

## Workflow: seed

1. Dependency order: types referenced by others come first. Record the order.
2. Generate data (no API calls). Follow `references/data-format.md`.
   - One file per type: `scripts/seed/data/<apiId>.json`.
   - Each entry must have the type's `keyField` (usually `slug`, `name`, or `title`)
     filled and unique, because it drives idempotency.
   - Reference related entries with `{ "$ref": "<apiId>:<key>" }` and uploaded files
     with `{ "$media": "<filename>" }`. Never hardcode ids.
   - Do not include publish fields (`publishedAt`, `status`). The scripts apply the publish
     state chosen in the plan.
   - Match field types and the editor's output format (HTML, Markdown, or Strapi
     blocks JSON). Respect required fields, enumerations, and min/max lengths.
   - Make content realistic and varied. Spread dates over a believable range.
   - Write in batches of about 10 entries to avoid truncation.
   - CHECKPOINT: user reviews the data files.
3. Write the project files from `references/script-templates.md`:
   `seed.config.json`, `lib.mjs`, `seed.mjs`, `upload-media.mjs`, `unseed.mjs`.
   Adapt `seed.config.json` to the approved plan: `strapiVersion`, `publishState`, and per
   type `kind`, `route`, `keyField`, `draftAndPublish`. Do not edit script logic unless
   the schema needs it (for example locale handling).
   - CHECKPOINT: user reviews config and scripts.
4. Media (only if chosen): put files in `scripts/seed/data/media/`, then run
   `node scripts/seed/upload-media.mjs`. Confirm files appear in the Media Library.
5. Run order (agent runs it if the user chose that, otherwise give the commands):
   1. `node scripts/seed/seed.mjs --dry-run`
   2. `node scripts/seed/seed.mjs --limit 3` and inspect the results in the admin
   3. `node scripts/seed/seed.mjs` for the full run
      Use `--types a,b` to run a subset.
6. Verify (see "Verification").

## Workflow: unseed

1. Show the plan checkpoint for unseed and get explicit approval.
2. Dry run first: `node scripts/seed/unseed.mjs --dry-run [--types a,b] [--media]`.
3. Run for real. Deletion order is the reverse of seed order.
4. Add `--from-data` only if the state file is missing or incomplete. It looks up
   entries by the keys in the data files, and needs the user's OK first.
5. Single types: unseed clears their content. Only do it if the user asked for it.
6. Verify the counts dropped as expected.

## Workflow: reseed

Run unseed (with its checkpoint), then seed. Never skip the unseed approval.

## Token requirements

Tell the user which permissions the token needs on the TARGET types:

- Collection types: find, create, and (for unseed) delete. Add update for re-runs.
- Single types: find and update (delete for unseed).
- Media: the Upload plugin's `upload` and `destroy` permissions (and find).
- Easiest for a local dev instance: a Full access token. For anything shared, use a
  Custom token limited to the chosen types.
  Also: if admin forms show "No permissions to see this field" after schema changes,
  that is a role setting under Settings > Roles, unrelated to the API token.

## Verification

- Counts: `GET /api/<route>?pagination[pageSize]=1` and read `meta.pagination.total`.
  Compare with the planned counts. Include drafts using the draft query parameter for the detected version (the scripts already do this).
- Spot-check in the admin: one entry per type, relations linked, media attached,
  components filled, publish state as chosen.
- If a frontend consumes the API, make sure the Public role has find and findOne on
  the seeded types (and on media), or the frontend gets 403.

## Common errors

- 401 Unauthorized: wrong or missing token. Check `.env` loading.
- 403 Forbidden: token lacks permission on that type or on Upload.
- 400 ValidationError: required field missing, wrong type, or bad enumeration value.
  Re-read the schema and the error `details`.
- 404: use the plural API id for collection types and the singular for single types.
- Duplicate or unique errors: the key value already exists. Idempotency should skip it.
- Relation not set: pass ids; many relations take an array of ids.
- Seeded drafts "missing" from lookups: the existence check must include drafts (the
  version's draft query parameter) or they will not be found and will be duplicated.
- `$ref` or `$media` unresolved: the target type or media was not seeded first.
