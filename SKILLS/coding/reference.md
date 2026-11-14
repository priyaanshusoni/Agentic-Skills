# Data format

Conventions used by the seed scripts. All paths are relative to `scripts/seed/`.

## Contents

- Folder layout
- seed.config.json
- Data files
- References between entries ($ref)
- Media references ($media)
- Components and dynamic zones
- Rich text fields
- Locales and publish state
- State file

## Folder layout

```
scripts/seed/
├── .env                  (STRAPI_URL, STRAPI_TOKEN; never commit)
├── .seed-state.json      (ids created by the scripts; never commit)
├── seed.config.json
├── lib.mjs
├── seed.mjs
├── upload-media.mjs
├── unseed.mjs
└── data/
    ├── <apiId>.json      (one file per content type)
    └── media/            (optional image files to upload)
```

## seed.config.json

```json
{
  "strapiVersion": 5,
  "publishState": "published",
  "order": ["group", "item", "site-settings"],
  "mediaDir": "data/media",
  "types": {
    "group": {
      "kind": "collection",
      "route": "groups",
      "keyField": "slug",
      "draftAndPublish": true
    },
    "item": {
      "kind": "collection",
      "route": "items",
      "keyField": "slug",
      "draftAndPublish": true
    },
    "site-settings": { "kind": "single", "route": "site-settings" }
  }
}
```

- `strapiVersion`: major version number of the target instance (from `package.json`).
  The scripts use it for id handling and draft/publish parameters. The `5` above is only
  an example value.
- `publishState`: `published` or `draft`. Applied to types with `draftAndPublish: true`.
- `draftAndPublish`: set per type to match the schema's draft and publish setting.
- `order`: seed order. Types that others reference come first. Unseed uses the reverse.
- Keys under `types` are the type's singular API id and match the data file name.
- `route`: plural API id for collection types, singular API id for single types.
- `keyField`: a field that is unique per entry (`slug`, `name`, `title`, `email`).
  Used to skip entries that already exist. Required for collection types.
- Add `"locale": "en"` on a type if i18n is enabled and you seed one locale.

## Data files

A collection type file is an array of entries. A single type file is one object.
Entries contain exactly the fields from the schema, plus the reference markers below.

```json
[
  { "name": "Group One", "slug": "group-one", "description": "..." },
  { "name": "Group Two", "slug": "group-two", "description": "..." }
]
```

Do not include `id`, `documentId`, `createdAt`, `updatedAt`, `createdBy`, or publish
fields (`publishedAt`, `status`). The scripts apply the publish state from `publishState`.
Ordinary custom date fields (for example `eventDate`) are fine.

## References between entries ($ref)

Never hardcode ids. Point at another seeded entry by its type and key value:

```json
{
  "title": "Item One",
  "slug": "item-one",
  "group": { "$ref": "group:group-one" },
  "related": [{ "$ref": "item:item-two" }, { "$ref": "item:item-three" }]
}
```

Format: `"$ref": "<apiId>:<keyValue>"`. The script replaces the object with the id from
the state file (a number or a string, depending on the Strapi version). The referenced type must be seeded earlier in `order`.
One-relations take a single `$ref`. Many-relations take an array of `$ref` objects.
To link to an entry that already existed before seeding, fetch its id with GET and
write the plain id (for example `"group": 4`).

## Media references ($media)

Reference an uploaded file by its file name in `data/media/`:

```json
{
  "title": "Item One",
  "cover": { "$media": "cover-1.jpg" },
  "gallery": [{ "$media": "cover-1.jpg" }, { "$media": "cover-2.jpg" }]
}
```

Single media fields take one `$media`. Multiple media fields take an array.
Run `upload-media.mjs` first so the ids exist in the state file.
If media is not wanted, leave these fields out (and make sure they are not required).

## Components and dynamic zones

Components are nested objects matching the component schema. Repeatable components are
arrays. `$ref` and `$media` markers work inside components too.

```json
{
  "seo": { "metaTitle": "Item One", "metaDescription": "..." },
  "blocks": [
    { "__component": "shared.rich-text", "body": "<p>...</p>" },
    {
      "__component": "shared.image-row",
      "images": [{ "$media": "cover-1.jpg" }]
    }
  ]
}
```

Dynamic zone entries need `__component` set to `<category>.<componentName>` exactly
as it appears in the schema.

## Rich text fields

Match what the field and editor store:

- Rich text field using a WYSIWYG plugin (for example CKEditor): an HTML string.
- Markdown field: a Markdown string.
- Strapi built-in Blocks editor: JSON, not a string, for example
  `[{ "type": "paragraph", "children": [{ "type": "text", "text": "Hello" }] }]`.
- Plain text or short text: a string, respecting length limits.

## Locales and publish state

- If i18n is on, the scripts add the locale parameter for types that set `locale` in the
  config. Seed each locale as a separate run, as the plan says.
- If draft and publish is on for a type, set `draftAndPublish: true` in the config. The
  scripts then apply `publishState` using the right mechanism for `strapiVersion`, and
  existence checks include drafts so re-runs do not create duplicates.

## State file

`.seed-state.json` records what the scripts created, so re-runs skip existing entries and
unseed knows what to delete (ids are numbers or strings, depending on the version):

```json
{
  "entries": {
    "group": { "group-one": "<id>" },
    "item": { "item-one": "<id>" }
  },
  "single": { "site-settings": true },
  "media": { "cover-1.jpg": "<id>" }
}
```

If it is lost, `unseed.mjs --from-data` can rebuild ids by looking up each key from the
data files (needs the user's approval, since it matches by key).
