# GTNH as a first-class pack platform (with Java 25)

**Date:** 2026-08-16
**Status:** Approved for implementation

## Problem

GregTech: New Horizons is half-wired into the panel. `GTNH` is offered as a server type in
[`config/field-catalog/general.js`](../../../src/config/field-catalog/general.js), and
[`services/mods.js`](../../../src/services/mods.js) counts it as pack-managed. Nothing else knows it
exists, which leaves three concrete gaps:

1. **Java.** `pickJavaTag()` resolves GTNH's `1.7.10` to `java8`. Running it on Java 25 — which GTNH
   2.8.0+ supports and recommends, via its bundled lwjgl3ify path — means hand-overriding the image
   tag in Advanced settings.
2. **Pinning.** None of the `GTNH_*` environment variables are catalogued, so `GTNH_PACK_VERSION`
   defaults to `latest` and the image silently upgrades the pack on every restart. That directly
   violates the panel's core "modpacks are always pinned" invariant.
3. **Discovery.** There is no path through the creation wizard that produces a working GTNH server.

## Decisions

| Decision           | Choice                                                                          |
| ------------------ | ------------------------------------------------------------------------------- |
| Scope              | Full first-class pack platform — pinning, update checking, blueprint round-trip |
| Wizard entry point | Featured card on the existing **From modpack** tab                              |
| Release channels   | Stable + beta behind an opt-in toggle. **No** nightly/daily builds              |
| Java selection     | Automatic, derived from the pinned version's `maxJavaVersion`                   |
| Integration shape  | A fourth platform inside `services/packs.js`                                    |

### Non-goals

- **Nightly/daily builds.** `GTNH_USE_DAILY_BUILD` resolves through GitHub Actions artifacts and
  requires a `GH_TOKEN` personal access token — a new secret to manage and a second resolution path
  that bypasses `versions.json`. Excluded; users who want it can set the env vars by hand.
- **A built-in GTNH blueprint.** The featured card is the better front door, and users can export
  their own tuned server as a blueprint once this lands.
- **Refactoring `packs.js` into a provider registry.** Correct only if a fifth and sixth pack source
  appear; YAGNI today.

## Upstream facts this design relies on

Verified 2026-08-16:

- `https://downloads.gtnewhorizons.com/versions.json` is the index the itzg image itself resolves
  against. It is a JSON **object keyed by version string, ordered newest-first**, 37 entries.
- Each entry carries `title` (`"Stable release"` — 16 entries — or `"Beta release"` — 21),
  `description` (HTML containing a changelog `<a href>`), `releaseDate` (`"YYYY/MM/DD"`),
  `maxJavaVersion`, and `mmc` / `server` objects holding `java8Url` and `java17_2XUrl`.
- `maxJavaVersion` across the index is 25 (8 entries, 2.8.0+), 24 (1), 21 (25), 20 (3). The newest
  stable release is **2.8.4** (2025-12-23, cap 25). No indexed version caps below 20, so Java 17 is
  a universally safe fallback.
- **The endpoint returns HTTP 403 without a `User-Agent` header.** The UA is functionally required,
  not cosmetic.
- `TYPE=GTNH` in the image accepts `GTNH_PACK_VERSION` (`latest`, `latest-beta`, or an exact
  version), `GTNH_DELETE_BACKUPS`, `SKIP_GTNH_UPDATE_CHECK`, and the daily-build pair above. On
  Java 17+ the image launches `@java9args.txt -jar lwjgl3ify-forgePatches.jar` itself — **the panel
  never hand-rolls JVM arguments.**
- Upstream resource guidance: 6 GB heap early game (+0.5–1 GB per player), 2–4 cores, 20 GB+ disk.

## Architecture

GTNH becomes a fourth `platform` inside the existing pack machinery. `resolvePack` → `packEnv` →
`applyPack` → `latestFor` already dispatch on platform, and `server_packs` already holds the pin, so
update checking, blueprint export/import and the mods overlay start working by construction rather
than by special-casing.

```
wizard (From modpack tab)
        │  POST /api/servers/from-pack  { platform: 'gtnh', ref: 'gtnh', versionId }
        ▼
packs.resolvePack('gtnh', …) ──▶ gtnhApi.listVersions()/getVersion() ──▶ versions.json (api_cache)
        │
        ├─▶ packs.packEnv()   → TYPE=GTNH, GTNH_PACK_VERSION, SKIP_GTNH_UPDATE_CHECK
        └─▶ packs.applyPack() → server_packs pin (+ max_java_version, channel), pending_recreate
                                        │
                                        ▼
                        servers.resolveImage() → pickJavaTag(mc, type, { maxJavaVersion })
```

### New module: `src/services/gtnhApi.js`

Modelled on [`services/modrinthApi.js`](../../../src/services/modrinthApi.js): `api_cache`-backed
with a 30-minute TTL, a panel `User-Agent`, `AbortSignal.timeout(15000)`, and stale-cache fallback on
429/5xx.

Split into a thin HTTP layer plus a **pure `normalizeIndex(raw)`** so the parsing logic is testable
with no network. Exports:

- `normalizeIndex(raw)` → normalized entries, newest-first (preserving the index's own key order —
  no version comparator is invented, since beta suffixes make string ordering unreliable).
- `listVersions({ includeBeta = false })`
- `getVersion(version)` — throws `httpError(404)` when the key is absent.
- `latest({ includeBeta = false })`

One normalized entry:

```js
{ version: '2.8.4', channel: 'stable', releaseDate: '2025/12/23',
  maxJavaVersion: 25, changelogUrl: 'https://github.com/…', serverUrl: 'https://…Server_Java_17-25.zip' }
```

`channel` maps from `title` (`"Beta release"` → `'beta'`, anything else → `'stable'`).
`changelogUrl` is the first `href` in `description`, accepted only if it parses as `https:` on a
`github.com` host — otherwise `null`.

### `services/packs.js`

`resolvePack('gtnh', ref, { versionId })` returns the same descriptor shape the existing pickers
consume:

```js
{ platform: 'gtnh', projectRef: 'gtnh', projectId: 'gtnh', projectName: 'GT New Horizons',
  versionId: '2.8.4', versionName: '2.8.4', mcVersion: '1.7.10',
  maxJavaVersion: 25, channel: 'stable',
  allVersions: [{ id, name, type: 'stable'|'beta', date, maxJavaVersion }] }
```

- `projectRef` is the constant `'gtnh'` — there is exactly one GTNH project.
- The pack version **is** its own id, so `versionId === versionName`.
- `mcVersion` is hardcoded `'1.7.10'` with a comment: GTNH is a 1.7.10 pack by definition and the
  index does not state a Minecraft version.
- `versionId` is accepted only when it is a key in the fetched index; anything else is a 404.

`packEnv(resolved)` for GTNH returns:

```js
{ TYPE: 'GTNH', GTNH_PACK_VERSION: resolved.versionId, SKIP_GTNH_UPDATE_CHECK: 'true' }
```

`SKIP_GTNH_UPDATE_CHECK` is deliberate: the image's own boot-time update check is redundant against a
pinned version and conflicts with the panel owning the update lifecycle.

The env-strip regex at `packs.js:150` becomes `/^(CF_|MODRINTH_|FTB_|GTNH_)/` so switching platforms
leaves no stale pin behind.

`latestFor` gains a `gtnh` branch comparing the pin against
`gtnhApi.latest({ includeBeta: pin.channel === 'beta' })`, so a stable-tracking server is never
offered a beta.

### Java resolution

`pickJavaTag(mcVersion, type, { maxJavaVersion } = {})` gains a GTNH branch that short-circuits the
`1.7.10 → java8` rule and returns the highest tag the panel ships that fits under the cap:

| `maxJavaVersion` | image tag | why                             |
| ---------------- | --------- | ------------------------------- |
| 25               | `java25`  | GTNH 2.8.0+                     |
| 24               | `java21`  | the panel ships no `java24` tag |
| 21               | `java21`  |                                 |
| 20               | `java17`  | the panel ships no `java20` tag |
| unknown / null   | `java17`  | safe for every indexed version  |

The cap reaches the matrix through [`servers.resolveImage()`](../../../src/services/servers.js), the
single place the image tag is decided:

```js
const pin = db.get('SELECT max_java_version FROM server_packs WHERE server_id = ?', server.id);
const tag = server.java_tag || pickJavaTag(server.mc_version, server.type, { maxJavaVersion: pin?.max_java_version });
```

Direct SQL rather than `require('./packs')`: `packs.js` requires `servers.js` at the top, so the
reverse edge would need a lazy-require cycle-breaker that a one-column read does not justify.

Consequences: `servers.java_tag` keeps meaning "the user overrode auto" and an Advanced override
still wins; resolution at spec time means changing the pinned version re-resolves Java on the
recreate `applyPack` already flags via `pending_recreate`; and pinning 2.7.4 yields `java21` rather
than a container that cannot boot.

### Data model

Migration `008_gtnh_pack.js`:

```sql
ALTER TABLE server_packs ADD COLUMN max_java_version INTEGER;
ALTER TABLE server_packs ADD COLUMN channel TEXT;
```

Both are null for the existing platforms. `channel` lives on the pin so the update checker knows
whether this server tracks stable or beta. `applyPack` writes both.

Servers created with `TYPE=GTNH` by hand before this feature have no `server_packs` row, so the cap
reads null and they resolve to `java17`, which boots. No backfill.

### Field catalog

Added to [`config/field-catalog/packs.js`](../../../src/config/field-catalog/packs.js):

- `GTNH_PACK_VERSION` — `hidden: true`, "Managed by the modpack installer UI", matching `CF_FILE_ID`.
- `GTNH_DELETE_BACKUPS` — visible, advanced, boolean.
- `SKIP_GTNH_UPDATE_CHECK` — `hidden: true`; the panel sets it.

The `javaTag` field's help text in `general.js` gains a sentence noting that overriding a GTNH server
to `java8` makes the image install GTNH's Java 8 server pack instead of the lwjgl3ify one.

### API surface

No new endpoints. `'gtnh'` joins the platform `z.enum` on `/packs/resolve`, `/servers/:id/packs` and
`/packs/details` in [`web/routes/api.js`](../../../src/web/routes/api.js), and on the blueprint
manifest's pack schema in [`blueprints/index.js`](../../../src/blueprints/index.js). `/packs/search`
stays Modrinth + CurseForge — GTNH is a fixed card, not a search result. `versionId` validates as
`/^[\w.-]{1,64}$/`.

`/api/servers/from-pack` is unchanged: it already resolves, reads `packEnv().TYPE`, creates, pins and
starts, all platform-agnostically.

### Wizard

A featured card `#wz-gtnh-card` sits above the search box in the From-modpack panel, always visible
on that tab. **Set up** resolves `platform=gtnh` and drives the _existing_ selection UI:
`selection = { platform: 'gtnh', ref: 'gtnh', resolved }`, `#wz-pack-selected` reveals, and
`#wz-pack-version` — already labelled "(always pinned)" — fills with GTNH versions.

Two GTNH-only controls sit beside that select:

- an **include beta releases** checkbox (stable-only by default) that refills the options from the
  already-fetched list;
- a read-only **Java** line. Each `allVersions` entry carries its own `maxJavaVersion`, so switching
  version updates "Java 25 — highest supported by 2.8.4" to "Java 21 — 2.7.4 caps at Java 21" with no
  round trip.

Searching Modrinth/CurseForge clears the GTNH selection and vice versa — the picker already assumes a
single selection.

Selecting GTNH **raises** (never lowers) `#wz-ram` to 6144 MB and `#wz-quota` to 20 GB. `resources()`
derives the container limit as `heap × 1.5`, landing at ~9 GB, which matches upstream guidance with no
special-casing. When the host-RAM clamp puts 6 GB out of reach, the card says so rather than silently
creating a server that will OOM on first world load.

### Updates and upgrades

`checker.checkAll` needs no loop change — it already calls `packs.latestFor` per server.
`packChangelogUrl` gains a `gtnh` case returning the per-version changelog link from the index, which
is a real diff rather than the "all files" page other platforms fall back to.

The upgrade orchestrator is generic apart from one number: `upgrade.js:116` allows CurseForge and
Modrinth 20 minutes to install and everything else 10. A GTNH server pack is ~1–2 GB and its first
boot builds a 1.7.10 world with several hundred mods, so 10 minutes would produce spurious
"install failed" rollbacks. GTNH gets a 30-minute bucket.

## Error handling

| Situation                                   | Behavior                                                                                                               |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Index unreachable / 429 / 5xx               | Serve the stale `api_cache` copy; with no cache, the GTNH card shows an error and the rest of the wizard keeps working |
| Index unreachable during a check            | Existing `catch` in `checkAll` keeps the previous cached result                                                        |
| Pinned version vanishes from the index      | Running server untouched (env already set); UI reports "pinned version no longer listed"; no exception                 |
| `maxJavaVersion` missing                    | Falls back to `java17`                                                                                                 |
| `versionId` not a key in the index          | `httpError(404)` — never passed through to container env                                                               |
| Changelog href not `https:` on `github.com` | `null`                                                                                                                 |

`worldVersionWarnings` reads the `Version` tag from `level.dat`, which 1.7.10 worlds predate, so it
returns no warning rather than a false alarm. No change needed.

## Testing

The suite runs on a clean clone with no Docker and no network; this preserves that. A trimmed real
`versions.json` is committed as a fixture.

- **`test/gtnhApi.test.js`** — `normalizeIndex`: channel mapping, changelog-href extraction including
  the reject-non-github case, `includeBeta` filtering, newest-first ordering, an entry missing
  `maxJavaVersion`.
- **`test/javaMatrix.test.js`** (extend) — GTNH with caps 25/24/21/20 → `java25`/`java21`/`java21`/
  `java17`; unknown cap → `java17`; a non-GTNH type ignores the new option so existing rows cannot
  regress.
- **`test/packs-gtnh.test.js`** — `resolvePack`/`packEnv` against a stubbed `gtnhApi`: descriptor
  shape, `SKIP_GTNH_UPDATE_CHECK`, an unknown version rejected, and applying GTNH over a CurseForge
  pin stripping the `CF_` vars.
- **`test/routes.test.js`** (extend) — `'gtnh'` accepted by the platform enums; an invalid version
  string rejected with 400.

`test/migrate.test.js` covers migration `008` by construction, since it runs the whole chain.

## Files touched

| File                                  | Change                                                            |
| ------------------------------------- | ----------------------------------------------------------------- |
| `src/services/gtnhApi.js`             | **new** — index client + `normalizeIndex`                         |
| `src/services/packs.js`               | `gtnh` branch in `resolvePack`/`packEnv`/`latestFor`; strip regex |
| `src/services/javaMatrix.js`          | `maxJavaVersion` option + GTNH branch                             |
| `src/services/servers.js`             | `resolveImage` reads the pin's cap                                |
| `src/db/migrations/008_gtnh_pack.js`  | **new** — two columns on `server_packs`                           |
| `src/config/field-catalog/packs.js`   | three `GTNH_*` fields                                             |
| `src/config/field-catalog/general.js` | `javaTag` help text note                                          |
| `src/web/routes/api.js`               | `'gtnh'` in three platform enums; `versionId` pattern             |
| `src/blueprints/index.js`             | `'gtnh'` in the manifest pack platform enum                       |
| `src/updates/checker.js`              | `packChangelogUrl` gtnh case                                      |
| `src/updates/upgrade.js`              | 30-minute install bucket for gtnh                                 |
| `views/wizard.hbs`                    | featured card + beta toggle + Java line markup                    |
| `public/js/pages/wizard.js`           | card wiring, version fill, resource floor                         |
| `test/*`                              | as listed above, plus the `versions.json` fixture                 |
