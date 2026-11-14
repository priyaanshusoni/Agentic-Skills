# Script templates

Copy these into `scripts/seed/` in the Strapi project. Only `seed.config.json` should
need editing for a normal run (see `data-format.md`). Plain Node 18+ (Node 20 preferred),
no dependencies.

## Contents

- seed.config.json (starter)
- lib.mjs
- seed.mjs
- upload-media.mjs
- unseed.mjs
- Flags summary
- Adapting to a schema

## seed.config.json (starter)

Replace with the approved plan. Keys under `types` are singular API ids.
`strapiVersion` is the major version of the target instance (the `5` is an example).

```json
{
  "strapiVersion": 5,
  "publishState": "published",
  "order": ["example-type"],
  "mediaDir": "data/media",
  "delayMs": 150,
  "types": {
    "example-type": {
      "kind": "collection",
      "route": "example-types",
      "keyField": "slug",
      "draftAndPublish": true
    }
  }
}
```

## lib.mjs

```js
import fs from "node:fs";
import path from "node:path";
import { fileURLToPath } from "node:url";

export const dir = path.dirname(fileURLToPath(import.meta.url));
export const at = (p) => path.join(dir, p);

export function loadEnv() {
  const file = at(".env");
  if (!fs.existsSync(file)) throw new Error("Missing scripts/seed/.env");
  for (const line of fs.readFileSync(file, "utf8").split("\n")) {
    const m = line.match(/^\s*([A-Z_]+)\s*=\s*(.*?)\s*$/);
    if (m) process.env[m[1]] = m[2].replace(/^["']|["']$/g, "");
  }
  const url = process.env.STRAPI_URL;
  if (!url || !process.env.STRAPI_TOKEN)
    throw new Error("STRAPI_URL and STRAPI_TOKEN are required in .env");
  const host = new URL(url).hostname;
  const local = ["localhost", "127.0.0.1", "::1", "[::1]"].includes(host);
  if (!local && !flag("--allow-remote")) {
    throw new Error(
      `Refusing non-local URL (${host}). Pass --allow-remote only after the user confirmed this exact URL.`,
    );
  }
}

export const flag = (name) => process.argv.includes(name);
export function strFlag(name) {
  const i = process.argv.indexOf(name);
  return i > -1 ? process.argv[i + 1] : undefined;
}
export const numFlag = (name) =>
  strFlag(name) !== undefined ? Number(strFlag(name)) : undefined;

export const config = () =>
  JSON.parse(fs.readFileSync(at("seed.config.json"), "utf8"));
export const readData = (apiId) =>
  JSON.parse(fs.readFileSync(at(`data/${apiId}.json`), "utf8"));
export const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

const statePath = at(".seed-state.json");
export function loadState() {
  const s = fs.existsSync(statePath)
    ? JSON.parse(fs.readFileSync(statePath, "utf8"))
    : {};
  return { entries: {}, single: {}, media: {}, ...s };
}
export const saveState = (s) =>
  fs.writeFileSync(statePath, JSON.stringify(s, null, 2));

// Selected types in config order (or reverse), filtered by --types a,b
export function selectTypes(cfg, reverse = false) {
  const only = strFlag("--types")
    ?.split(",")
    .map((s) => s.trim());
  if (only)
    for (const t of only)
      if (!cfg.types[t]) throw new Error(`Unknown type "${t}" in --types`);
  const list = cfg.order.filter((t) => !only || only.includes(t));
  return reverse ? [...list].reverse() : list;
}

export async function api(method, route, body) {
  const isForm = typeof FormData !== "undefined" && body instanceof FormData;
  const res = await fetch(
    `${process.env.STRAPI_URL.replace(/\/$/, "")}/api/${route}`,
    {
      method,
      headers: {
        Authorization: `Bearer ${process.env.STRAPI_TOKEN}`,
        ...(body && !isForm ? { "content-type": "application/json" } : {}),
      },
      body: body ? (isForm ? body : JSON.stringify(body)) : undefined,
    },
  );
  const text = await res.text();
  if (!res.ok)
    throw new Error(
      `${method} ${route} -> ${res.status} ${text.slice(0, 500)}`,
    );
  return text ? JSON.parse(text) : null;
}

// ---- Version-aware helpers: the only place version differences live ----
// Check the official REST API docs for the target version if anything here looks off.
const CFG = config();
if (!CFG.strapiVersion)
  throw new Error('Set "strapiVersion" (major number) in seed.config.json');
const usesDocumentIds = () => Number(CFG.strapiVersion) >= 5;

// Identifier used in routes and relations: documentId when the API returns one, else id.
export const idOf = (d) => d?.documentId ?? d?.id;

function withParams(route, params) {
  const p = params.filter(Boolean);
  if (!p.length) return route;
  return route + (route.includes("?") ? "&" : "?") + p.join("&");
}
const localeParam = (t) =>
  t.locale ? `locale=${encodeURIComponent(t.locale)}` : null;
const draftParam = () =>
  usesDocumentIds() ? "status=draft" : "publicationState=preview";
const publishParam = (t) =>
  usesDocumentIds() && t.draftAndPublish
    ? `status=${CFG.publishState === "draft" ? "draft" : "published"}`
    : null;

// Route for writes and deletes. Adds locale and, for writes, the publish status if needed.
export const routeFor = (t, suffix = "", { publish = false } = {}) =>
  withParams(`${t.route}${suffix}`, [
    localeParam(t),
    publish ? publishParam(t) : null,
  ]);

// Body for writes. Versions that publish through a body field get it added here.
export function withPublishState(data, t) {
  if (usesDocumentIds() || !t.draftAndPublish || "publishedAt" in data)
    return data;
  return {
    ...data,
    publishedAt: CFG.publishState === "draft" ? null : new Date().toISOString(),
  };
}

// Existing entry id for a key value, or null. Includes drafts so re-runs do not duplicate.
export async function findByKey(t, value) {
  const q = withParams(
    `${t.route}?filters[${t.keyField}][$eq]=${encodeURIComponent(value)}&pagination[pageSize]=1`,
    [t.draftAndPublish ? draftParam() : null, localeParam(t)],
  );
  const r = await api("GET", q);
  return idOf(r?.data?.[0]) ?? null;
}

// Replace { $ref: "type:key" } and { $media: "file" } markers with ids.
// lenient=true (dry run) returns 0 for unresolved markers instead of throwing.
export function resolveRefs(v, state, lenient = false) {
  if (Array.isArray(v)) return v.map((x) => resolveRefs(x, state, lenient));
  if (v && typeof v === "object") {
    if (typeof v.$ref === "string") {
      const [type, key] = v.$ref.split(/:(.*)/s);
      const id = state.entries[type]?.[key];
      if (id) return id;
      if (lenient) return 0;
      throw new Error(`Unresolved $ref "${v.$ref}" (seed "${type}" first)`);
    }
    if (typeof v.$media === "string") {
      const id = state.media[v.$media];
      if (id) return id;
      if (lenient) return 0;
      throw new Error(
        `Unresolved $media "${v.$media}" (run upload-media.mjs first)`,
      );
    }
    return Object.fromEntries(
      Object.entries(v).map(([k, x]) => [k, resolveRefs(x, state, lenient)]),
    );
  }
  return v;
}
```

## seed.mjs

```js
import {
  loadEnv,
  config,
  readData,
  loadState,
  saveState,
  selectTypes,
  api,
  routeFor,
  findByKey,
  resolveRefs,
  withPublishState,
  idOf,
  sleep,
  flag,
  numFlag,
} from "./lib.mjs";

loadEnv();
const cfg = config();
const dry = flag("--dry-run");
const limit = numFlag("--limit");
const state = loadState();
const failures = [];
let created = 0,
  skipped = 0;

for (const apiId of selectTypes(cfg)) {
  const t = cfg.types[apiId];
  console.log(`\n== ${apiId} (${t.kind}) ==`);
  try {
    if (t.kind === "single") {
      const data = resolveRefs(readData(apiId), state, dry);
      if (dry) {
        console.log("DRY PUT", routeFor(t), JSON.stringify(data).slice(0, 200));
        continue;
      }
      await api("PUT", routeFor(t, "", { publish: true }), {
        data: withPublishState(data, t),
      });
      state.single[apiId] = true;
      saveState(state);
      console.log(`ok    ${apiId} (single type updated)`);
      created++;
      continue;
    }
  } catch (e) {
    failures.push(`${apiId}: ${e.message}`);
    console.error(`fail  ${apiId}`);
    continue;
  }

  state.entries[apiId] ??= {};
  const items = readData(apiId).slice(0, limit ?? undefined);
  for (const item of items) {
    const key = item[t.keyField];
    try {
      if (!key) throw new Error(`entry is missing keyField "${t.keyField}"`);
      const existing = dry ? null : await findByKey(t, key);
      if (existing) {
        state.entries[apiId][key] = existing;
        skipped++;
        console.log(`skip  ${key}`);
        continue;
      }
      const data = resolveRefs(item, state, dry);
      if (dry) {
        console.log(
          "DRY POST",
          routeFor(t),
          JSON.stringify(data).slice(0, 160),
        );
        continue;
      }
      const r = await api("POST", routeFor(t, "", { publish: true }), {
        data: withPublishState(data, t),
      });
      state.entries[apiId][key] = idOf(r.data);
      saveState(state);
      created++;
      console.log(`ok    ${key} (#${idOf(r.data)})`);
      await sleep(cfg.delayMs ?? 150);
    } catch (e) {
      failures.push(`${apiId}/${key ?? "?"}: ${e.message}`);
      console.error(`fail  ${apiId}/${key ?? "?"}`);
    }
  }
}

saveState(state);
console.log(
  `\nDone${dry ? " (dry run)" : ""}. created=${created} skipped=${skipped} failed=${failures.length}`,
);
if (failures.length) {
  console.error("\nFailures:\n" + failures.join("\n"));
  process.exit(1);
}
```

## upload-media.mjs

```js
import fs from "node:fs";
import path from "node:path";
import {
  loadEnv,
  config,
  at,
  loadState,
  saveState,
  api,
  sleep,
  flag,
} from "./lib.mjs";

loadEnv();
const cfg = config();
const dry = flag("--dry-run");
const dirPath = at(cfg.mediaDir ?? "data/media");
const MIME = {
  ".jpg": "image/jpeg",
  ".jpeg": "image/jpeg",
  ".png": "image/png",
  ".webp": "image/webp",
  ".gif": "image/gif",
  ".svg": "image/svg+xml",
  ".mp4": "video/mp4",
  ".pdf": "application/pdf",
};

if (!fs.existsSync(dirPath)) {
  console.error(`No media folder at ${dirPath}`);
  process.exit(1);
}
const state = loadState();
const files = fs
  .readdirSync(dirPath)
  .filter((f) => !f.startsWith(".") && MIME[path.extname(f).toLowerCase()]);
let failed = 0;

for (const f of files) {
  if (state.media[f]) {
    console.log(`skip  ${f}`);
    continue;
  }
  if (dry) {
    console.log(`DRY upload ${f}`);
    continue;
  }
  try {
    const form = new FormData();
    const type = MIME[path.extname(f).toLowerCase()];
    form.append(
      "files",
      new Blob([fs.readFileSync(path.join(dirPath, f))], { type }),
      f,
    );
    if (!flag("--no-alt")) {
      const alt = path.basename(f, path.extname(f)).replace(/[-_]+/g, " ");
      form.append(
        "fileInfo",
        JSON.stringify({ alternativeText: alt, caption: alt }),
      );
    }
    const r = await api("POST", "upload", form);
    state.media[f] = r[0].id;
    saveState(state);
    console.log(`ok    ${f} (#${r[0].id})`);
    await sleep(cfg.delayMs ?? 150);
  } catch (e) {
    failed++;
    console.error(`fail  ${f}: ${e.message}`);
  }
}
console.log(`\nDone${dry ? " (dry run)" : ""}. failed=${failed}`);
if (failed) process.exit(1);
```

## unseed.mjs

```js
import {
  loadEnv,
  config,
  readData,
  loadState,
  saveState,
  selectTypes,
  api,
  routeFor,
  findByKey,
  sleep,
  flag,
} from "./lib.mjs";

loadEnv();
const cfg = config();
const dry = flag("--dry-run");
const state = loadState();
let deleted = 0,
  failed = 0;

async function remove(label, route, onOk) {
  if (dry) {
    console.log(`DRY DELETE ${route}  (${label})`);
    return;
  }
  try {
    await api("DELETE", route);
    onOk?.();
    saveState(state);
    deleted++;
    console.log(`deleted ${label}`);
    await sleep(cfg.delayMs ?? 150);
  } catch (e) {
    failed++;
    console.error(`fail    ${label}: ${e.message}`);
  }
}

// Reverse order: entries that reference others go first.
for (const apiId of selectTypes(cfg, true)) {
  const t = cfg.types[apiId];
  console.log(`\n== ${apiId} (${t.kind}) ==`);

  if (t.kind === "single") {
    // Clears the single type's content. Only runs if it was seeded by these scripts.
    if (state.single[apiId])
      await remove(apiId, routeFor(t), () => {
        delete state.single[apiId];
      });
    else console.log("skip  (not seeded by these scripts)");
    continue;
  }

  const ids = new Map(Object.entries(state.entries[apiId] ?? {}));
  if (flag("--from-data")) {
    // Fallback if the state file is incomplete: look entries up by key from the data files.
    for (const item of readData(apiId)) {
      const key = item[t.keyField];
      if (key && !ids.has(key)) {
        const id = await findByKey(t, key);
        if (id) ids.set(key, id);
      }
    }
  }
  if (!ids.size) {
    console.log("nothing to delete");
    continue;
  }
  for (const [key, id] of ids) {
    await remove(`${apiId}/${key} (#${id})`, routeFor(t, `/${id}`), () => {
      delete state.entries[apiId]?.[key];
    });
  }
}

if (flag("--media")) {
  console.log("\n== media ==");
  for (const [name, id] of Object.entries(state.media)) {
    await remove(`media/${name} (#${id})`, `upload/files/${id}`, () => {
      delete state.media[name];
    });
  }
}

console.log(
  `\nDone${dry ? " (dry run)" : ""}. deleted=${deleted} failed=${failed}`,
);
if (failed) process.exit(1);
```

## Flags summary

| Script             | Flags                                                                  |
| ------------------ | ---------------------------------------------------------------------- |
| `seed.mjs`         | `--dry-run`, `--limit N`, `--types a,b`, `--allow-remote`              |
| `upload-media.mjs` | `--dry-run`, `--no-alt`, `--allow-remote`                              |
| `unseed.mjs`       | `--dry-run`, `--types a,b`, `--media`, `--from-data`, `--allow-remote` |

`--allow-remote` is only for a non-local URL the user explicitly confirmed.

## Adapting to a schema

- Locale: set `"locale": "<code>"` on a type to seed that locale. Run once per locale if
  several are needed, changing the config between runs.
- Relations take the ids stored in the state file (numbers or documentId strings,
  depending on the version). Many-relations take an array. Confirm the relation format in
  the REST docs for the detected version.
- Single types use the singular API id as the route. Set `route` accordingly in the config.
- Version differences (ids, draft lookups, publish state) live in one block in `lib.mjs`
  (`idOf`, `draftParam`, `publishParam`, `withPublishState`). If the target version behaves
  differently from what these assume, fix them there and nowhere else.
- If the instance sits behind a path prefix (for example `/cms`), include it in `STRAPI_URL`.
- Do not change the guard in `loadEnv()` that refuses non-local URLs.
- If a type has no good unique field, add a `uid` or a unique `name` to the data and use
  it as `keyField`. Without a key, idempotency and unseed-by-key cannot work for that type.
