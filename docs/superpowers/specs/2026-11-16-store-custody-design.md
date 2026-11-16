# Store custody: the catalogue a tantrum cannot empty (slice 1)

- **Date:** 2026-11-16
- **Status:** design approved section by section in brainstorming (2026-11-15/16); this document is the written spec
- **Issue:** [ReCasaOS/CasaOS#134](https://github.com/ReCasaOS/CasaOS/issues/134) slice 1 only
- **Components:** ReCasaOS/_appstore, recasaos.org, CasaOS-AppManagement, CasaOS-UI, CasaOS-Install
- **Decided by:** nightfullstar (this brainstorming). **(choice)** = designer default, contestable.

This is not the whole of #134. Overlays, compose-aware update badges, the community pipeline, the proposal website, and the crawler are later slices.

## Why

A store on a ReCasaOS box is a zip URL in `app-management.conf`. Until November 2026 the default still named a zip on a person's GitHub repository. That person deleted the repository. Boxes that pointed at it saw 404. Icons and screenshots were hotlinked from the same host, so even a mirrored compose zip left an empty vitrine.

ReCasaOS already merges sources into `recasaos.zip` and carries a dead source forward at **build** time. The box still:

- skips a refresh when `Content-Length` is unchanged (`appstore size not changed`), so a compose-only change of the same size never arrives
- treats GitHub (raw branch, jsdelivr) as the runtime host
- keeps old Big Bear URLs in existing conf files; a migration tool from 0.4.12 **adds** that URL if it is missing
- serves card images from whatever URL `x-casaos.icon` carries, usually the upstream repo

Slice 1 makes the **published catalogue** the thing a box trusts, not a maintainer's git tree. GitHub stays an editorial surface and a mirror. A tantrum against an upstream repo, or against the GitHub copy, must not empty the vitrine.

## Decisions

1. **Threat for this slice:** an upstream maintainer deletes or empties their repository. Defending against ReCasaOS org deletion or a GitHub ban is a later second host of last resort, not a new architecture. **(decided: C)**
2. **Both runtimes exist.** Boxes can fetch the same bytes from `recasaos.org` and from GitHub. The API (org) feeds GitHub; it also serves the catalogue on its own. **(decided)**
3. **Dead URLs:** the installer rewrites known-dead defaults on upgrade; the dashboard offers “switch to the ReCasaOS catalogue” and never writes silently. **(decided: C)**
4. **Icons (and screenshots) live in the zip.** Merge copies them in and rewrites `x-casaos.icon` to an in-zip path. Compose is no longer byte-identical to upstream. **(decided: A)**
5. **New file `store-index.json`.** `catalogue.json` stays the merge report. Old AppManagement ignores unknown files. **(decided: A)**
6. **One zip, index inside, published as a GitHub Release and on recasaos.org.** Same `sha256`. **(decided: approach 1, then inverted so org is primary)**
7. **Out of this slice:** overlay provenance, `revision` as an update badge, 3-way merge, `revoked`, signing, `POST /proposals`, crawler, community source, opt-in vs default community.

## What already exists (code, 2026-11-16)

- **`ReCasaOS/_appstore`:** `catalogue/build.py` merges IceWhale, RoqueOS, and `ReCasaOS/big-bear-casaos` into `store/recasaos.zip` on the `catalogue` branch. Dead sources are carried from the last published zip. A merge below 250 apps or below 90% of the last count is not published. Compose files are copied as shipped. Icons stay as upstream URLs. `catalogue.json` is a source map (`origin`, duplicates, drops).
- **AppManagement** (`service/appstore.go`, `appstore_mirror.go`): every ~10 minutes `UpdateCatalog` `HEAD`s the configured URL via `pickCatalogueSource`. IceWhale URLs fall back to this distribution's copy; the ReCasaOS zip falls back to jsdelivr. Skip is `Content-Length == lastAPPStoreSize`. A failed extract restores `workdir.backup`. Workdir is `/var/lib/casaos/appstore/<host>/<md5(path)>`. Default in `app-management.conf.sample`: `https://raw.githubusercontent.com/ReCasaOS/_appstore/catalogue/store/recasaos.zip`.
- **Migration landmine:** `cmd/migration-tool/migration_0412_and_older.go` still **appends** `https://github.com/bigbeartechworld/big-bear-casaos/archive/refs/heads/master.zip` when that string is absent.
- **UI:** `AppStoreSourceManagement.vue` lists sources, adds and removes them. No successor offer. StoreInfo.icon is passed through as the string in `x-casaos`.
- **recasaos.org:** the public site (`ReCasaOS/recasaos.org`). It does not yet host the catalogue zip.

## What it does

### Record

One publication is one zip whose SHA-256 is the identity, not a URL.

```
recasaos.zip
  store-index.json
  catalogue.json          # unchanged role: merge report
  LICENSES/
  Apps/<id>/docker-compose.yml
  Apps/<id>/icon.png      # or .jpg / .webp / .svg, when the merge could fetch it
  Apps/<id>/screenshot-N  # optional, same rules
```

The zip is written first to **recasaos.org**, then the **same bytes** are pushed to GitHub (Release asset `recasaos.zip` and the `catalogue` branch path `store/recasaos.zip`). If GitHub is down, the org copy is still the publication. If they diverge, a box must not mix them: it takes one successful fetch whose sha256 matches `store-index.json`, or it keeps last-good.

### `store-index.json`

Schema `recasaos.store-index.v1`. Unknown fields: a v1 reader ignores them **(choice)**. A missing file: the zip is a v1 store as today.

```json
{
  "schema": "recasaos.store-index.v1",
  "store_id": "recasaos-catalogue",
  "name": { "en_US": "ReCasaOS catalogue" },
  "url": "https://recasaos.org/store/recasaos.zip",
  "mirrors": [
    "https://github.com/ReCasaOS/_appstore/releases/latest/download/recasaos.zip"
  ],
  "status": "active",
  "successor": null,
  "sha256": "<64 hex of this zip>",
  "apps": [
    {
      "id": "immich",
      "revision": 4,
      "content_hash": "sha256:…"
    }
  ]
}
```

| Field | Rule |
|---|---|
| `status` | `active` \| `moved` \| `deprecated` \| `dead` |
| `successor` | URL; required when `moved` or `deprecated`; **empty** when `dead` |
| `url` | primary fetch (org) |
| `mirrors` | GitHub (and later others); client tries these in order after `url` |
| `sha256` | SHA-256 of the zip bytes (hex, 64 chars) |
| `apps[].id` | compose `name`, once |
| `apps[].revision` | monotone in this store; bumped when rendered compose or store metadata (including icon rewrite) changes |
| `apps[].content_hash` | sha256 of the **store's** rendered compose file |

Slice 1 **writes** `revision` and `content_hash`. The box **does not** open an update badge from them. That is slice 2.

`catalogue.json` is not read by the client.

### Slice 1 API on recasaos.org

Not the proposal API. HTTPS GET of static artifacts **(choice: static files, no extra daemon in this slice)**:

- `GET https://recasaos.org/store/recasaos.zip`
- `GET https://recasaos.org/store/recasaos.zip.sha256` — `<hex>  recasaos.zip\n`
- `GET https://recasaos.org/store/store-index.json` — the same object as inside the zip
- `GET https://recasaos.org/store/health` — `200` and `ok` if the zip and index are present

Write access is the publish job (deploy token). Boxes never POST.

GitHub is fed **from this publication**, not the other way around: after a successful org write, upload Release assets `recasaos.zip`, `recasaos.zip.sha256`, and `store-index.json`, then update the `catalogue` branch (`store/recasaos.zip` and `store/store-index.json`). A GitHub failure is logged; it does not roll back the org copy. The box that falls back to GitHub uses `https://github.com/ReCasaOS/_appstore/releases/latest/download/store-index.json` the same way it uses the org index.

### Box refresh

Every ten minutes, per configured store URL:

1. If the URL is in the **dead table** and already marked dead on this box: do not fetch. Last-good stays. UI may still show the switch offer until accepted or the source is removed.
2. Otherwise `GET` `store-index.json` next to the zip when the host is recasaos.org (`/store/store-index.json`) or, for a zip URL, the index inside a previous last-good plus `HEAD` of the zip. For the ReCasaOS catalogue, try org first, then each `mirrors` entry (built-in order if there is no index yet): GitHub Release, then current raw `catalogue` URL.
3. If `store-index.json` `sha256` equals the sha256 recorded for the current workdir: skip the zip download.
4. Else `HEAD` the zip (ETag, then Last-Modified). `Content-Length` alone must not skip a download.
5. Both hosts fail: **do not delete workdir**. Catalog stays last-good. Source state `unreachable` (not a perpetual 404 row).
6. Download to `workdir.tmp`. If `store-index.json` is present, the zip bytes must hash to `sha256` or the tmp is discarded and last-good remains.
7. Count apps in the new catalog. If count `< 250` or `< 90%` of the last successful count on this box: discard, last-good remains. **(choice: same thresholds as the builder)**
8. On success: today's rename (`workdir` → `workdir.backup` → tmp becomes `workdir`); delete backup only after the new catalog loads.

`pickCatalogueSource` today maps configured URL → mirror. Invert for the ReCasaOS catalogue: configured org URL tries org, then GitHub. IceWhale URLs keep the existing fallback to this distribution's IceWhale copy. The Big Bear URL is not a catalogue with an index; it is handled by the dead table.

### Dead table

Hard-coded in AppManagement and in the installer, **same list**. A `store-index.json` `status`/`successor` on a zip the box actually fetched **overrides** the table. The table exists for sources that cannot publish an index.

| Configured URL | Status | Successor |
|---|---|---|
| `https://github.com/bigbeartechworld/big-bear-casaos/archive/refs/heads/master.zip` | dead | `https://recasaos.org/store/recasaos.zip` |

IceWhale jsdelivr / `_appstore` zips that still answer are **not** in this table; existing `catalogueMirrors` stay.

**Installer (upgrade and new install):** rewrite a conf `appstore =` that **exactly** matches a dead-table URL (or the current GitHub-raw default for recasaos.zip) to `https://recasaos.org/store/recasaos.zip`. Do not touch any other `appstore =` line (user-added stores).

**`app-management.conf.sample`:** default becomes the org URL.

**`migration_0412_and_older`:** must **stop appending** the Big Bear URL. A box that still lacks it must not gain a dead store. **(choice: the tool becomes a no-op that logs “skipped, store is dead”)**

**UI:** when a listed source matches the dead table, or a fetched index says `moved`/`deprecated`/`dead`, show the status and, when `successor` is set, a control “Switch to the ReCasaOS catalogue” (copy: English key, `en_US` and `fr_FR`). Confirm writes the successor URL in place of that source (unregister old, register successor, or rewrite the conf line). Cancel leaves conf unchanged. Never rewrite without that confirm.

After a dead-table 404, stop polling that URL.

### Icons

At merge, for each kept app:

1. If `x-casaos.icon` is an `http`/`https` URL, GET it (timeout 10 s, size cap **128 KiB**). Allowed types: png, jpeg, webp, svg. On success write `Apps/<id>/icon` plus the matching extension (`icon.png`, `icon.jpg`, `icon.webp`, `icon.svg`). On failure, oversize, or other type: omit the file, set `x-casaos.icon` to empty, **keep the app**.
2. Rewrite `x-casaos.icon` in the **zip** compose to that relative filename (`icon.png`, …).
3. Screenshots: same, cap **512 KiB** each, files `screenshot-1.png` (or matching extension). A failed screenshot is skipped; later ones are still tried. Do not drop the app.

AppManagement, when building StoreInfo for the dashboard, if `icon` is not an absolute URL, rewrites it to:

`/v2/app_management/appstore/{storeId}/apps/{appId}/icon`

`storeId` is the existing numeric id in `AppStoreList`. The handler reads `Apps/<appId>/icon.png` (or `.jpg` / `.webp` / `.svg`) from that store's workdir and sets `Content-Type` from the extension. 200 with `Cache-Control: public, max-age=86400`; unknown app or missing file: 404. Token: same as other appstore GETs (signed-in). The browser never requests GitHub for a catalogue icon.

Old AppManagement (no rewrite, no route): the relative `icon.png` does not load; the card has no image; install still works. No back-compat fetch of the old upstream URL. **(decided in architecture)**

### Errors

| Situation | Result |
|---|---|
| recasaos.org down, GitHub up, hashes match | use mirror |
| both down | last-good, `unreachable` |
| sha256 mismatch | last-good, do not publish |
| zip shrinks below 250 or 90% | last-good, do not publish |
| index `moved` / dead table | offer switch, no silent write |
| icon fetch failed at build | app in zip, icon empty |
| GitHub publish failed after org write | org is source of truth; mirror stale |

## Non-goals (this slice)

- Replacing the zip format; requiring IceWhale v2 hosting
- Compose-aware update offers, 3-way merge, quieter “store definition changed” copy
- Overlays / provenance patches
- Community store, Mistral draft, OPA/Trivy, `revoked`, trust tiers
- Store API `POST /proposals`, votes, crawler
- A third host if GitHub **and** recasaos.org both die
- Serving icons for **old** clients that do not rewrite relative paths

## Proof

### `_appstore` (`catalogue/test_build.py`)

- Emitted `store-index.json` validates: `schema`, `store_id`, `url` is the org zip, `mirrors` contains the GitHub Release URL, every app has `id`, `revision`, `content_hash`, zip `sha256` matches the file.
- Fixture app with an HTTP icon → zip contains `Apps/<id>/icon.png` and compose `x-casaos.icon: icon.png`.
- Icon over 128 KiB → no file, empty icon field, app still in the zip.
- Unreachable upstream → apps carried from last-good zip.
- Merge `< 250` or `< 90%` → no publish (existing tests stay, still true with index and icons).

### AppManagement

- `pickCatalogueSource`: org 200 → org; org fail GitHub 200 → GitHub; both fail → error, workdir unchanged.
- ETag unchanged → no download. Same `Content-Length`, different ETag → download.
- sha256 mismatch or app count under threshold → backup restored, catalog not empty.
- Dead-table URL: no fetch loop; successor offered; conf not written in the service (UI/installer write).
- Relative `icon.png` in StoreInfo becomes the gateway path; GET of the route returns the workdir file; unknown id 404.

### UI

- Dead source row shows the switch control. Confirm → primary URL is `https://recasaos.org/store/recasaos.zip`. Cancel → conf unchanged.
- `<img>` for a catalogue app uses the gateway icon path, not `icon.png` as a page-relative URL.

### Installer

- Upgrade: exact dead-table `appstore =` line becomes the org URL. A user-added third URL is left.
- Fresh install: sample/default is the org URL.
- `migration_0412_and_older` does not append Big Bear.

### Manual, once per release of this slice

- Published zip on recasaos.org and on the GitHub Release have the same sha256.
- With org blocked, a box still shows the catalogue from the mirror or last-good.
- Catalogue icon GET on the box is 200 and does not contact GitHub.

## Rollout

1. `_appstore` emits `store-index.json` and inlined icons; publish to recasaos.org then GitHub. Old boxes ignore the new file and may show no icons.
2. AppManagement: dual fetch, ETag/index sha256, last-good, dead table, icon route, StoreInfo rewrite. Tag and pin.
3. UI switch offer. Pin.
4. Installer: default URL, rewrite dead lines, disable Big Bear append. Distribution release.

A box that never upgrades AppManagement keeps last-good of whatever zip it last fetched. It will not learn `moved` from an index it cannot parse; the installer rewrite is what saves those boxes that still name Big Bear and **do** run the new installer.

## Open follow-ups (not this spec)

- Slice 2: honour `revision` / `content_hash` for compose-aware updates and 3-way merge.
- Slice 3: overlays.
- Slice 4: community pipeline and the write side of the Store API.
- Thin index-first fetch if inlined icons make the zip too large (schema already allows `GET store-index.json` separately).
- Second copy off both GitHub and recasaos.org.
