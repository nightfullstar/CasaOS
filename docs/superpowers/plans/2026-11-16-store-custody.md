# Store custody (slice 1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A box fetches the ReCasaOS catalogue from recasaos.org (GitHub as mirror), keeps last-good when a source dies, inlines icons in the zip, and migrates the dead Big Bear URL without silently rewriting user-added stores.

**Architecture:** `_appstore` emits `store-index.json` and rewrites icons into the zip, then publishes identical bytes to recasaos.org and a GitHub Release. AppManagement tries org then GitHub, skips on index sha256 / ETag (not Content-Length), restores `workdir.backup` on hash or shrink failure, and serves icons from the unpacked workdir. UI offers a successor switch. Installer and conf.sample point at the org URL; the 0.4.12 migration stops appending Big Bear.

**Tech Stack:** Python 3 (`catalogue/build.py` unittest), Go (AppManagement `go test ./...`), Vue 3 / vitest (CasaOS-UI), installer shell. No new Go or npm dependencies.

## Global Constraints

- Spec: `CasaOS/docs/superpowers/specs/2026-11-16-store-custody-design.md` (this workspace) and [ReCasaOS/CasaOS#134](https://github.com/ReCasaOS/CasaOS/issues/134) slice 1 only.
- Primary catalogue URL: `https://recasaos.org/store/recasaos.zip`
- GitHub Release assets: `recasaos.zip`, `recasaos.zip.sha256`, `store-index.json`
- Dead URL (exact match): `https://github.com/bigbeartechworld/big-bear-casaos/archive/refs/heads/master.zip` → successor = org zip
- Do not implement overlays, revision badges, 3-way merge, community API, crawler, or `revoked`.
- `_appstore`, `recasaos.org`, and `CasaOS-Install` are **not** in this workspace; clone them beside `ReCasaOS/` when those tasks run.
- Do not commit unless the user asks. Skip every Commit step.

## Files

| Path | Role |
|---|---|
| `CasaOS-AppManagement/service/appstore_mirror.go` | Org URL, mirrors, dead table, `pickCatalogueSource` order |
| `CasaOS-AppManagement/service/appstore.go` | ETag / index sha256 skip, shrink + hash refuse, last-good |
| `CasaOS-AppManagement/service/compose_app.go` | Rewrite relative icon to gateway path in StoreInfo |
| `CasaOS-AppManagement/route/v2/appstore.go` + `openapi.yaml` | `GET .../appstore/{id}/apps/{app}/icon` |
| `CasaOS-AppManagement/build/sysroot/etc/casaos/app-management.conf.sample` | Default org URL |
| `CasaOS-AppManagement/cmd/migration-tool/migration_0412_and_older.go` | Stop appending Big Bear |
| `CasaOS-UI/src/components/Apps/AppStoreSourceManagement.vue` | Switch offer |
| `_appstore/catalogue/build.py` | Index + icon rewrite |
| `recasaos.org` static `/store/` | Host zip + index |
| `CasaOS-Install` | Rewrite dead `appstore =` on upgrade |

---

### Task 1: Dual fetch and dead table (AppManagement)

**Files:**
- Modify: `CasaOS-AppManagement/service/appstore_mirror.go`
- Modify: `CasaOS-AppManagement/service/appstore_mirror_internal_test.go`
- Test: same `_internal_test.go`

**Interfaces:**
- Produces:
  - `const recasaosCatalogue = "https://recasaos.org/store/recasaos.zip"`
  - `const recasaosCatalogueGitHub = "https://github.com/ReCasaOS/_appstore/releases/latest/download/recasaos.zip"`
  - `const recasaosCatalogueGitHubIndex = "https://github.com/ReCasaOS/_appstore/releases/latest/download/store-index.json"`
  - `const recasaosCatalogueIndex = "https://recasaos.org/store/store-index.json"`
  - `var deadStores map[string]string` — dead URL → successor
  - `func SuccessorOf(url string) (successor string, dead bool)`
  - `pickCatalogueSource` for `recasaosCatalogue`: try org, then GitHub Release zip (not jsdelivr)
- Keep IceWhale `catalogueMirrors` behaviour for IceWhale URLs.

Work in `/home/ikanat/GIT/ReCasaOS/CasaOS-AppManagement`.

- [ ] **Step 1: Write the failing test**

Add to `appstore_mirror_internal_test.go`:

```go
func TestTheRecasaosCataloguePrefersTheOrgThenGitHub(t *testing.T) {
	logger.LogInitConsoleOnly()
	asked := []string{}
	head := func(url string) (*http.Response, error) {
		asked = append(asked, url)
		if url == recasaosCatalogue {
			return &http.Response{StatusCode: http.StatusBadGateway}, nil
		}
		return &http.Response{StatusCode: http.StatusOK, ContentLength: 9}, nil
	}
	source, _, err := pickCatalogueSource(recasaosCatalogue, head)
	if err != nil || source != recasaosCatalogueGitHub {
		t.Fatalf("got %q %v", source, err)
	}
	if len(asked) < 2 || asked[0] != recasaosCatalogue {
		t.Fatalf("org first: %v", asked)
	}
}

func TestBigBearIsDeadAndNamesTheOrgSuccessor(t *testing.T) {
	logger.LogInitConsoleOnly()
	const dead = "https://github.com/bigbeartechworld/big-bear-casaos/archive/refs/heads/master.zip"
	got, ok := SuccessorOf(dead)
	if !ok || got != recasaosCatalogue {
		t.Fatalf("got %q %v", got, ok)
	}
}
```

Replace `TestTheRecasaosCatalogueHasACopyOnAnotherHost` so it no longer expects jsdelivr for the new org URL. Keep a test that the **legacy** raw GitHub `catalogue/store/recasaos.zip` URL still has a fallback (GitHub Release or org).

- [ ] **Step 2: Run the test and see it fail**

```bash
cd /home/ikanat/GIT/ReCasaOS/CasaOS-AppManagement
go test ./service -run 'TestTheRecasaosCataloguePrefersTheOrgThenGitHub|TestBigBearIsDeadAndNamesTheOrgSuccessor|TestTheRecasaosCatalogueHasACopyOnAnotherHost' -count=1
```

Expected: FAIL — `recasaosCatalogue` still the GitHub-raw URL; `SuccessorOf` undefined.

- [ ] **Step 3: Implement**

In `appstore_mirror.go`:

- Change `recasaosCatalogue` to `https://recasaos.org/store/recasaos.zip`.
- Set `catalogueMirrors[recasaosCatalogue]` to the GitHub Release zip URL.
- Also map the **legacy** raw `https://raw.githubusercontent.com/ReCasaOS/_appstore/catalogue/store/recasaos.zip` to org first if you want one hop, or to the Release URL; spec: configured org tries org then GitHub. Legacy raw should try org, then Release **(choice: legacy raw → org, then Release)**.
- Add `deadStores` and `SuccessorOf`.
- `pickCatalogueSource` stays: HEAD configured URL, on failure HEAD mirror. For org URL that is enough if `catalogueMirrors[org] = githubRelease`.

- [ ] **Step 4: Re-run tests**

```bash
go test ./service -count=1
```

Expected: PASS.

- [ ] **Step 5: Commit** — skip unless asked.

---

### Task 2: Skip on ETag / index sha256; refuse shrink and bad hash

**Files:**
- Modify: `CasaOS-AppManagement/service/appstore.go`
- Create: `CasaOS-AppManagement/service/appstore_refresh_internal_test.go`

**Interfaces:**
- Consumes: `pickCatalogueSource`, workdir backup rename already in `UpdateCatalog`
- Produces: `lastETag string` (and/or `lastSHA256`) on `appStore`; skip when ETag matches **or** index sha256 matches workdir record; do **not** skip on Content-Length alone; if unzipped app count `< 250` or `< 90%` of last successful count, restore backup; if `store-index.json` `sha256` ≠ zip bytes, restore backup.

- [ ] **Step 1: Failing tests** in `appstore_refresh_internal_test.go`

Cover, with a fake `head`/`get` and a temp workdir:

1. Same Content-Length, different ETag → download attempted (assert a download flag).
2. Same ETag → no download.
3. Zip whose `store-index.json` sha256 does not match the bytes → existing catalog files still present.
4. New catalog with 1 app when last successful count was 10 (`min` use 2 in test via a test hook or by setting last count) → refused, old catalog remains.

Keep production thresholds 250 / 90% as in the spec; tests inject counts through unexported fields on `appStore` (`lastAppCount int`).

- [ ] **Step 2: Run, expect FAIL** (`lastETag` unused; Content-Length still skips).

```bash
go test ./service -run TestRefresh -count=1
```

- [ ] **Step 3: Implement in `UpdateCatalog`**

- `HEAD` via `pickCatalogueSource`; read `ETag` header.
- If `ETag != "" && ETag == s.lastETag` { return nil }.
- Remove the `res.ContentLength == s.lastAPPStoreSize` skip (or only use it when ETag is absent **and** Last-Modified is absent **and** no index sha256 — spec: Content-Length alone must not skip; so delete that branch).
- After download, if `store-index.json` exists, hash zip or unpacked equivalent: the download helper extracts a zip; hash the downloaded zip file before extract if `downloadHelper` can be pointed at a file. If not, hash concatenated app compose bytes only as a last resort — **prefer hashing the zip file**. Inspect `pkg/utils/downloadHelper` and hash before extract.
- Count apps after `BuildCatalog`; compare to `s.lastAppCount` when `lastAppCount > 0`.
- On failure after rename: existing backup defer already restores; return error **before** `isSuccessful = true`.
- On success: store `lastETag`, `lastSHA256`, `lastAppCount`.

- [ ] **Step 4:** `go test ./service -count=1` PASS.

- [ ] **Step 5: Commit** — skip unless asked.

---

### Task 3: Icon route and StoreInfo rewrite

**Files:**
- Modify: `CasaOS-AppManagement/api/app_management/openapi.yaml` (new GET)
- Modify: `CasaOS-AppManagement/route/v2/appstore.go`
- Modify: `CasaOS-AppManagement/service/compose_app.go` (`StoreInfo`)
- Create: `CasaOS-AppManagement/route/v2/appstore_icon_internal_test.go`
- Create: `CasaOS-AppManagement/service/store_icon_internal_test.go`
- Run `go generate` if this repo's codegen is from the yaml.

**Interfaces:**
- Produces: `GET /v2/app_management/appstore/{id}/apps/{app}/icon` → file from workdir `Apps/{app}/icon.{png|jpg|webp|svg}`
- `StoreInfo`: if `Icon` is not `http://` or `https://`, set it to `/v2/app_management/appstore/{id}/apps/{name}/icon`

Need the store numeric id inside StoreInfo rewrite: Catalog() is per-store; when listing, the management layer knows the id. Prefer rewriting in `AppStoreManagement` when assembling the compose app store info list, not inside `ComposeApp.StoreInfo`, so `ComposeApp` stays store-id-agnostic.

- [ ] **Step 1: Failing tests**

`store_icon_internal_test.go`: relative `icon.png` in x-casaos becomes the gateway path when the list is built with store id 0.

`appstore_icon_internal_test.go`: workdir with `Apps/immich/icon.png` → 200 `image/png`; missing → 404.

- [ ] **Step 2: Run, FAIL** (route 404 / path unchanged).

- [ ] **Step 3: Implement** openapi path, handler, rewrite at list/detail assembly. `Content-Type` from extension. `Cache-Control: public, max-age=86400`. Auth: same JWT as other appstore GETs.

- [ ] **Step 4:** `go test ./service ./route/v2 -count=1` PASS. `go generate ./...` if yaml changed, then tests still pass.

- [ ] **Step 5: Commit** — skip unless asked.

---

### Task 4: Default conf and stop Big Bear append

**Files:**
- Modify: `CasaOS-AppManagement/build/sysroot/etc/casaos/app-management.conf.sample`
- Modify: `CasaOS-AppManagement/cmd/migration-tool/migration_0412_and_older.go`
- Create: `CasaOS-AppManagement/cmd/migration-tool/migration_0412_and_older_test.go` if that package can be tested; otherwise a small test in `cmd/migration-tool` that `IsMigrationNeeded` is false when Big Bear is absent.

**Interfaces:**
- Sample `appstore = https://recasaos.org/store/recasaos.zip`
- `IsMigrationNeeded` returns false always (or true only if we are *removing* Big Bear — spec: no-op that logs skipped). **Do not append.**

- [ ] **Step 1:** Test that `Migrate` does not add the Big Bear string to a conf that lacks it.

- [ ] **Step 2:** FAIL (current Migrate appends).

- [ ] **Step 3:** `IsMigrationNeeded` → false; `Migrate` → nil without write. Optional: if conf **contains** the dead URL, rewrite that line to org (this duplicates installer; spec puts rewrite in installer **and** AppManagement dead table. Doing rewrite here on upgrade of the **appmanagement package** is extra safety **(choice: yes, if the file contains the exact dead URL, replace that line; do not append)**.

- [ ] **Step 4:** tests PASS.

- [ ] **Step 5: Commit** — skip unless asked.

---

### Task 5: UI successor offer

**Files:**
- Modify: `CasaOS-UI/src/components/Apps/AppStoreSourceManagement.vue`
- Create: `CasaOS-UI/src/components/Apps/AppStoreSourceManagement.dead.spec.js` (match neighbouring spec style)
- Modify: `CasaOS-UI/src/assets/lang/en_US.json` and `fr_FR.json` (parity)

**Interfaces:**
- AppManagement `AppStoreList` must expose `status` / `successor` **or** the UI matches the dead URL client-side. Spec: UI when listed source matches dead table **or** fetched index says moved. Until list API grows fields, UI may special-case the Big Bear URL string **(choice: special-case in UI for slice 1, plus show successor if metadata later includes it)**.
- Copy key: `"Switch to the ReCasaOS catalogue"` / French equivalent.
- Confirm: unregister dead id, register `https://recasaos.org/store/recasaos.zip` (or rewrite). Cancel: no API write.

- [ ] **Step 1:** Spec: dead URL row renders the button; confirm calls register with org URL; cancel does not.

- [ ] **Step 2:** FAIL.

- [ ] **Step 3:** Implement. `<img>` for store apps already binds `:src="item.icon"` — after Task 3 that is the gateway path; add a spec that a relative icon from the API is used as-is (not prefixed with the page URL incorrectly). If the API still returns `icon.png`, the img would break — Task 3 must land first.

- [ ] **Step 4:** `pnpm test` (or the project's vitest command for that spec) PASS. Lang key parity test PASS.

- [ ] **Step 5: Commit** — skip unless asked.

---

### Task 6: `_appstore` index and icons

**Files:** (clone `https://github.com/ReCasaOS/_appstore` beside this workspace if missing)
- Modify: `catalogue/build.py`
- Modify: `catalogue/test_build.py`

**Interfaces:**
- After `merge`, write `store-index.json` as specified. `url` org zip; `mirrors` GitHub Release zip. `sha256` of packed zip. Per-app `revision` starting at 1, or last+1 when `content_hash` of compose changes vs previous manifest.
- Fetch icon URLs; caps 128 KiB / 512 KiB; rewrite compose.

- [ ] **Step 1:** Extend `test_build.py` as in the spec Proof section (index fields, icon rewrite, oversize, carry-over still works, sharp drop still refused).

- [ ] **Step 2:** `python catalogue/test_build.py` FAIL.

- [ ] **Step 3:** Implement in `build.py` without breaking existing merge tests.

- [ ] **Step 4:** `python catalogue/test_build.py` PASS.

- [ ] **Step 5: Commit** — skip unless asked.

Publish wiring (org + Release) is Task 8.

---

### Task 7: Installer rewrite

**Files:** clone `CasaOS-Install` if missing. Find the upgrade path that writes `app-management.conf` (grep `appstore`).

- Exact line `appstore = https://github.com/bigbeartechworld/big-bear-casaos/archive/refs/heads/master.zip` → org URL.
- Exact legacy raw recasaos.zip default → org URL.
- Any other `appstore =` line unchanged.
- New install uses sample from AppManagement tag that already has the org default (Task 4), or installer copies the org URL.

- [ ] **Step 1:** Shell or Go test of a fixture conf with three `appstore =` lines (dead, user, legacy raw) → only dead and legacy raw rewritten.

- [ ] **Step 2–4:** implement and run that test.

- [ ] **Step 5: Commit** — skip unless asked.

---

### Task 8: Publish to recasaos.org then GitHub

**Files:** `_appstore` workflow on `sync`; `recasaos.org` static `public/store/` or equivalent.

After a green merge that produces a zip:

1. Write `store/recasaos.zip`, `store/recasaos.zip.sha256`, `store/store-index.json`, and `store/health` (`ok`) on recasaos.org (deploy token).
2. Upload the same three files as GitHub Release assets; update `catalogue` branch.
3. If step 2 fails, step 1 remains; log; do not delete org files.

Manual proof once: sha256 equal on both hosts.

- [ ] Steps: add/adjust GitHub Actions; do not put tokens in the repo. Document required secrets in `_appstore` README.

- [ ] **Commit** — skip unless asked.

---

## Spec coverage

| Spec section | Task |
|---|---|
| Dual runtime org + GitHub | 1, 8 |
| `store-index.json` | 6 |
| ETag / no Content-Length skip / last-good / shrink / hash | 2 |
| Dead table + installer + migration | 1, 4, 7 |
| Icons in zip + box route | 3, 6 |
| UI switch | 5 |
| recasaos.org GET surface | 8 |
| Non-goals | not tasked |
