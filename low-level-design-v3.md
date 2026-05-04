# Centralized Patch Download — Low-Level Design

> **Version:** 4.0 | **Date:** 2026-05-04
> **Product:** ManageEngine Endpoint Central
> **Scope:** DB Schema, REST APIs (payloads & responses), Nginx Config, Meta Files
> **Source of Truth:** [`system-design.md`](system-design.md) · [`download-change-plan.md`](download-change-plan.md) · [`change-plan-uc2-onwards.md`](change-plan-uc2-onwards.md)
> **PoC Evidence:** [`poc-proven-report.md`](poc-proven-report.md) · [`poc5-proof.md`](poc5-proof.md)

---

## Table of Contents

1. [Design Principles](#1-design-principles)
2. [DB Schema](#2-db-schema)
   - 2.1 [ER Diagram — SS Tables](#21-er-diagram--ss-tables)
   - 2.2 [ER Diagram — Probe Tables](#22-er-diagram--probe-tables)
   - 2.3 [Table Definitions](#23-table-definitions)
   - 2.4 [Status Enumerations](#24-status-enumerations)
   - 2.5 [SyMParameter Keys](#25-symparameter-keys)
   - 2.6 [websettings.conf Keys](#26-websettingsconf-keys)
   - 2.7 [Table Summary](#27-table-summary)
3. [REST APIs](#3-rest-apis)
   - 3.1 [URL Convention & Auth](#31-url-convention--auth)
   - 3.2 [Settings APIs (Existing Cleanup Settings — Extended)](#32-settings-apis-existing-cleanup-settings--extended)
   - 3.3 [Store Validation API](#33-store-validation-api)
   - 3.4 [Probe → SS APIs (via PushToSummaryProcessor)](#34-probe--ss-apis-via-pushtosummaryprocessor)
   - 3.5 [Patch Store Management APIs (SS Admin)](#35-patch-store-management-apis-ss-admin)
   - 3.6 [Dependency Package APIs (SS Admin)](#36-dependency-package-apis-ss-admin)
   - 3.7 [Monitoring & Admin APIs](#37-monitoring--admin-apis)
   - 3.8 [Store Path Change API](#38-store-path-change-api)
   - 3.9 [Probe Settings Pull API](#39-probe-settings-pull-api)
   - 3.10 [Nginx Auth Servlets](#310-nginx-auth-servlets)
4. [Nginx Configuration](#4-nginx-configuration)
   - 4.1 [SS-Side Nginx](#41-ss-side-nginx)
   - 4.2 [Probe-Side Nginx (Self-Proxy Pattern)](#42-probe-side-nginx-self-proxy-pattern)
   - 4.3 [Template Placeholders & Mode Mapping](#43-template-placeholders--mode-mapping)
5. [Meta File Structures](#5-meta-file-structures)
   - 5.1 [Common Store Directory Layout](#51-common-store-directory-layout)
   - 5.2 [Per-Probe client-data Layout](#52-per-probe-client-data-layout)
   - 5.3 [XML Schemas & Samples](#53-xml-schemas--samples)
   - 5.4 [File Naming Conventions](#54-file-naming-conventions)
   - 5.5 [What Lives Where — Summary](#55-what-lives-where--summary)
6. [Settings Propagation Flow](#6-settings-propagation-flow)
7. [Reconciliation with Prior LLD Versions](#7-reconciliation-with-prior-lld-versions)

---

## 1. Design Principles

These principles are non-negotiable constraints from `system-design.md` that drive every schema and API decision below:

| # | Principle | LLD Impact |
|---|-----------|------------|
| 1 | **No push events for centralized download** — pure polling model (§10.3) | No `PATCH_STORE_UPDATED` / `ON_DEMAND_DOWNLOAD_FAILED` events. No event tables. |
| 2 | **No vendor fallback** — SS download failure = deployment failure (§13.3, §14) | No `FALLBACK_TO_VENDOR` column. No fallback API. |
| 3 | **No new collection statuses** — collection stays in "Draft - Download in progress" (§7.3) | No `502` / `503` status codes. |
| 4 | **Pending-patch state is in-memory** in `PatchDownloadListener` maps (§7.1 note) | No `CollectionPendingPatches` table. Startup recovery from collection definition (§9.5). |
| 5 | **No `OnDemandDownloadRequest`/`OnDemandPatchRequest` tables** — SS dedup via `PatchStoreLocation.STATUS_ID` on SS (§10.2) | Two tables eliminated. |
| 6 | **3-way download mode** (1=Probe, 2=SS+caching, 3=SS+routing) — not a boolean toggle (§7.1) | `DOWNLOAD_MODE` INTEGER column on existing `PatchStoreCleanupSettings`. |
| 7 | **Settings via existing cleanup settings API** — no dedicated settings table (§7.1, §8.1) | No `CentralizedDownloadSettings` table. Extend `PatchCleanupSettingsController`. |
| 8 | **Settings propagation via customer metadata XML** — not event push (§15.1) | `centralized-download-settings.xml` via `PatchMetaUtil`. |
| 9 | **SS restart required** for mode changes — Nginx config regenerated from templates at startup (§16.1) | No hot-reload for mode switch. |

---

## 2. DB Schema

### 2.1 ER Diagram — SS Tables

```mermaid
erDiagram
    %% ── MODIFIED TABLE ─────────────────────────────────────
    PatchStoreCleanupSettings {
        BIGINT CLEANUP_ID PK
        BOOLEAN NOTIFY_IF_SPACE_EXCEED
        INTEGER MAX_SPACE_LIMIT
        BOOLEAN SUPERSEDE_CLEANUP
        BOOLEAN DELETE_NOT_REQUIRED_PATCHES
        INTEGER DELETE_PATCHES_BY_PERIOD
        INTEGER DOWNLOAD_MODE "NEW: 1=Probe 2=SS+Cache 3=SS+Route"
    }

    %% ── REUSED TABLE (from Probe, no schema changes) ───────
    PatchStoreLocation_SS {
        BIGINT PATCHID PK
        INTEGER LANGUAGEID PK
        CHAR255 WEBPATH
        CHAR255 STOREPATH
        BIGINT PATCH_SIZE
        CHAR100 STATUS
        INTEGER STATUS_ID FK
        BIGINT DOWNLOAD_TIME "Repurposed as deletion-mark time for PENDING_DELETE"
        NCHAR2000 REMARKS
        INTEGER RETRY_COUNT
    }

    %% ── REUSED TABLES (from Probe, no schema changes) ──────
    RedhatCertDetails_SS {
        BIGINT ID PK
        NCHAR50 EDITION UK
        CHAR255 CERT_ALIAS
        BIGINT CERT_EXPIRY
        BOOLEAN IS_VALID
    }

    SuseProductKeys_SS {
        BIGINT KEY_ID PK
        BIGINT CUSTOMER_ID FK
        NCHAR100 OS_EDITION
        SCHAR500 PRODUCT_KEY
        INTEGER IS_VALID
    }

    %% ── EXISTING TABLES (FK reference only) ────────────────
    CustomerInfo {
        BIGINT CUSTOMER_ID PK
    }

    %% ── RELATIONSHIPS ──────────────────────────────────────
    CustomerInfo ||--o{ SuseProductKeys_SS : "owns keys"
```

### 2.2 ER Diagram — Probe Tables

```mermaid
erDiagram
    %% ── MODIFIED TABLE ─────────────────────────────────────
    PatchStoreCleanupSettings {
        BIGINT CLEANUP_ID PK
        BOOLEAN NOTIFY_IF_SPACE_EXCEED
        INTEGER MAX_SPACE_LIMIT
        BOOLEAN SUPERSEDE_CLEANUP
        BOOLEAN DELETE_NOT_REQUIRED_PATCHES
        INTEGER DELETE_PATCHES_BY_PERIOD
        INTEGER DOWNLOAD_MODE "NEW: synced from SS"
    }

    %% ── EXISTING TABLE (unchanged schema) ──────────────────
    PATCHSTORELOCATION {
        BIGINT PATCHID PK
        INTEGER LANGUAGEID PK
        CHAR255 WEBPATH
        CHAR255 STOREPATH
        BIGINT PATCH_SIZE
        CHAR100 STATUS
        INTEGER STATUS_ID FK
        BIGINT DOWNLOAD_TIME
    }

    %% No CollectionPendingPatches table.
    %% Pending state is in-memory in PatchDownloadListener maps.
    %% Rebuilt from collection definition on Probe restart (§9.5).
```

> **No `CollectionPendingPatches` table.** Per system-design.md §7.1: pending-patch state is tracked in-memory in `PatchDownloadListener.collectionToPatchStatusMap` / `nonCollectionToPatchStatusMap`. On Probe restart, the listener is repopulated from the collection definition (§9.5).

### 2.3 Table Definitions

#### 2.3.1 PatchStoreCleanupSettings — Modified (SS + Probe)

> **Existing singleton table** (`CLEANUP_ID` auto-generated). One new column added. Full CRUD infrastructure already exists: `PatchCleanupSettingsUtil`, `CleanupSettings` model, `PatchCleanupSettingsService`, `PatchCleanupSettingsController`, `PatchStoreCleanupXmlGenerator`.

**Existing columns (unchanged):**

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| `CLEANUP_ID` | BIGINT (PK, auto) | — | Singleton row identifier |
| `NOTIFY_IF_SPACE_EXCEED` | BOOLEAN | `false` | Notify when patch repo exceeds space limit |
| `MAX_SPACE_LIMIT` | INTEGER | `-1` | Maximum limit allotted for the patch store |
| `SUPERSEDE_CLEANUP` | BOOLEAN | `false` | Cleanup superseded patches |
| `DELETE_NOT_REQUIRED_PATCHES` | BOOLEAN | `false` | Cleanup old installed/not-applicable patches |
| `DELETE_PATCHES_BY_PERIOD` | INTEGER | `6` | Cleanup age threshold in months (allowed: 1,2,3,6,12,24,36) |

**New column:**

| Column | Type | Default | Nullable | Description |
|--------|------|---------|----------|-------------|
| **`DOWNLOAD_MODE`** | INTEGER | `1` | NO | `1`=Probe download (legacy), `2`=SS download with Probe caching, `3`=SS download without Probe caching |

**Data dictionary addition** (in `data-dictionary.xml`, inside `<table name="PatchStoreCleanupSettings">`):

```xml
<column name="DOWNLOAD_MODE">
    <description>Patch download mode: 1=Probe download (legacy),
        2=Summary Server download with Probe caching,
        3=Summary Server download without Probe caching</description>
    <data-type>INTEGER</data-type>
    <max-size>2</max-size>
    <default-value>1</default-value>
    <nullable>false</nullable>
    <unique>false</unique>
    <allowed-values>
        <value>1</value>
        <value>2</value>
        <value>3</value>
    </allowed-values>
</column>
```

**Why not a dedicated table:** `PatchStoreCleanupSettings` is already the single-row config for patch store behavior. Existing CRUD, API endpoints, and XML propagation all extend with one column — zero new infrastructure.

---

#### 2.3.2 PatchStoreLocation (SS — Reused from Probe, No Schema Changes)

> **Reused table** — same structure as the existing Probe-side `PatchStoreLocation` (defined in `data-dictionary.xml`, `BinaryRepository` schema). Added to `data-dictionary-ss.xml` with zero modifications. Tracks per-patch download lifecycle on SS. Both SS and Probe use the same table name in their respective databases — they are never joined.
>
> **Why reuse (not a new table):**
> - Same conceptual role: "which patches do I have, what's their status, where are they on disk?"
> - Same PK: composite `(PATCHID, LANGUAGEID)`
> - Same DAO/Util reuse: `PatchStoreLocationUtil`, `PatchStoreLocationHandler`, and all existing query patterns work directly
> - The download pipeline (`PatchDownloadManager`) that SS reuses (system-design.md §2.2) writes to `PatchStoreLocation` natively — no adapter needed
> - Checksum is already stored separately in `BINARYCHECKSUMVALUES` table via `BinaryChecksumUtil.addOrUpdateBinaryCheckSumValues()` — no inline `CHECKSUM_VAL` needed
> - `PENDING_DELETE_AT` column is unnecessary — when status flips to `PENDING_DELETE`, `DOWNLOAD_TIME` is overwritten with the current timestamp (original download time is irrelevant for a patch being deleted). `DeferredCleanupTask` queries: `WHERE STATUS_ID = PENDING_DELETE AND DOWNLOAD_TIME < (now - grace_period)`

| Column | Type | Default | Nullable | Description |
|--------|------|---------|----------|-------------|
| **`PATCHID`** | BIGINT (PK) | — | NO | Patch identifier |
| **`LANGUAGEID`** | INTEGER (PK) | `1` | NO | Language variant (`0`=all languages, `1`=English, etc.) |
| `WEBPATH` | CHAR(255) | — | NO | SS-internal relative URL (e.g., `/common-store/{fileName}`) |
| `STOREPATH` | CHAR(255) | — | NO | Physical file path on SS disk |
| `PATCH_SIZE` | BIGINT | `0` | NO | Size in bytes |
| `STATUS` | CHAR(100) | — | NO | String status (`AVAILABLE`, `DLOAD_FAILED`, etc.) |
| `STATUS_ID` | INTEGER (FK) | `99` | NO | FK → `ConfigStatusDefn.STATUS_ID` |
| `DOWNLOAD_TIME` | BIGINT | `-1` | NO | Epoch ms when downloaded. **Repurposed** as deletion-mark time when `STATUS_ID = PENDING_DELETE`. |
| `REMARKS` | NCHAR(2000) | — | YES | Failure reason (e.g., "Checksum mismatch after 3 retries") |
| `REMARKS_ARGS` | NCHAR(250) | — | YES | Format arguments for REMARKS |
| `RETRY_COUNT` | INTEGER | `0` | NO | Download retry attempts |

**Indexes:**

| Index | Columns | Purpose |
|-------|---------|---------|
| PK | `(PATCHID, LANGUAGEID)` | Composite primary key |
| `IDX_PSL_STATUS` | `(STATUS_ID)` | Bulk queries: all QUEUED, all FAILED, etc. |
| `IDX_PSL_PENDING_DELETE` | `(STATUS_ID, DOWNLOAD_TIME)` | `DeferredCleanupTask`: `WHERE STATUS_ID = PENDING_DELETE AND DOWNLOAD_TIME < (now - grace)` |

**Soft-delete grace period — no extra column needed:**

```java
// DeferredCleanupTask (SS, runs every 15 min):
// When marking PENDING_DELETE:
psl.setStatusId(PENDING_DELETE);
psl.setDownloadTime(System.currentTimeMillis());  // repurpose as deletion-mark time
PatchStoreLocationUtil.update(psl);

// When checking grace period:
long cutoff = System.currentTimeMillis() - (CLEANUP_GRACE_PERIOD_MINUTES * 60_000L);
// SELECT * FROM PatchStoreLocation WHERE STATUS_ID = PENDING_DELETE AND DOWNLOAD_TIME < cutoff
```

**DDL file:** `data-dictionary-ss.xml` (copy from existing `data-dictionary.xml` — no modifications)

---

#### 2.3.3 RedhatCertDetails (SS — Reused from Probe, No Schema Changes)

> **Reused table** — same structure as the existing Probe-side `RedhatCertDetails`. Added to `data-dictionary-ss.xml` with zero modifications. One row per RedHat edition — stores certs forwarded from Probes for mTLS against `cdn.redhat.com`.
>
> **Why no `SOURCE_PROBE_ID` column:**
> - Download logic doesn't need it — SS uses the cert regardless of which Probe sent it
> - Dedup is keyed on `EDITION` (unique constraint) — source is irrelevant
> - Audit trail is maintained via log line at receive time: `"Received cert for edition={} from probeId={}"`
> - Adding a FK to `ProbeDetails` creates unnecessary coupling for pure audit data

| Column | Type | Default | Nullable | Description |
|--------|------|---------|----------|-------------|
| **`ID`** | BIGINT (PK, auto) | — | NO | Auto-generated |
| `EDITION` | VARCHAR(50) (UNIQUE) | — | NO | `Server` / `Workstation` / `Desktop` |
| `CERT_ALIAS` | VARCHAR(255) | `""` | NO | Keystore alias |
| `CERT_EXPIRY` | BIGINT | `-1` | YES | Expiry timestamp (`-1` = not set) |
| `IS_VALID` | BOOLEAN | `true` | NO | Whether cert is currently valid |

**Unique constraint:** `EDITION`
**DDL file:** `data-dictionary-ss.xml` (copy from existing Probe `data-dictionary.xml`)

---

#### 2.3.4 SuseProductKeys (SS — Reused from Probe, No Schema Changes)

> **Reused table** — same structure as the existing Probe-side `SuseProductKeys`. Added to `data-dictionary-ss.xml` with zero modifications. Stores SUSE registration tokens forwarded from Probes. `SuseAuthtokenTask` (also reused from Probe) fetches auth tokens from `scc.suse.com` and stores them in `SuseAuthTokens`.
>
> **Why no `SOURCE_PROBE_ID` column:**
> - Same reasoning as RedHat — token works regardless of source
> - Dedup is keyed on `(PRODUCT_KEY, CUSTOMER_ID)` — source is irrelevant
> - Audit via logging at receive time

| Column | Type | Default | Nullable | Description |
|--------|------|---------|----------|-------------|
| **`KEY_ID`** | BIGINT (PK, auto) | — | NO | Auto-generated |
| `CUSTOMER_ID` | BIGINT (FK) | — | NO | → `CustomerInfo.CUSTOMER_ID` |
| `OS_EDITION` | NCHAR(100) | — | NO | SUSE product edition |
| `PRODUCT_KEY` | SCHAR(500) | — | NO | SCC registration token |
| `IS_VALID` | INTEGER | `0` | NO | Whether token is valid |

**DDL file:** `data-dictionary-ss.xml` (copy from existing Probe `data-dictionary.xml`)

---

#### 2.3.5 PATCHSTORELOCATION (Probe — Existing, No Schema Changes)

> Existing table — unchanged. When centralized download is enabled, populated by the deployment gate (§9.2), polling scheduler (§9.4), and missing-patch scheduler (§17.2) via **direct UPSERT** (bypassing download queue).

| Column | Type | Notes for Centralized Download |
|--------|------|-------------------------------|
| `PATCHID` | BIGINT (PK) | Same as legacy |
| `LANGUAGEID` | INTEGER (PK) | Same as legacy |
| `WEBPATH` | CHAR(255) | Set to `https://{probe}:{port}/store/{fileName}` — DS/Agents download from Probe Nginx |
| `STATUS` | CHAR(100) | `AVAILABLE` when file confirmed in common store |
| `STATUS_ID` | INTEGER | `PATCH_DLOAD_AVAILABLE` constant |
| `DOWNLOAD_TIME` | BIGINT | Time when file was discovered in common store |

**No `SOURCE_TYPE` column added.** Per system-design.md §7.2: `WEBPATH` value already distinguishes source (`/store/{file}` = centralized, vendor URL = legacy). No query filters by source type.

---

### 2.4 Status Enumerations

#### PatchStoreLocation STATUS/STATUS_ID (SS — same pattern as Probe)

> Uses the existing `STATUS`/`STATUS_ID` pattern from `ConfigStatusDefn`. String values match existing Probe constants. Two new statuses added to `ConfigStatusDefn` for soft-delete lifecycle.

| STATUS (string) | STATUS_ID | Existing? | Description |
|-----------------|-----------|-----------|-------------|
| `DLOAD_REQUESTED` | (existing) | ✅ | Queued for download from vendor |
| `DLOAD_RUNNING` | (existing) | ✅ | Download in progress |
| `AVAILABLE` | (existing) | ✅ | Downloaded, checksum validated, ready for serving |
| `DLOAD_FAILED` | (existing) | ✅ | Download failed after retries. `.failed` marker written. |
| `PENDING_DELETE` | (new) | ❌ | Soft-deleted. `DOWNLOAD_TIME` overwritten with deletion-mark time. Physical removal after grace period. |
| `DELETED` | (new) | ❌ | Physically deleted from common store. |

#### Collection Status — No New Values

> **No new collection statuses.** Per system-design.md §7.3: collection stays in existing "Draft - Download in progress" while patches are pending from SS. The listener's `performPostDownloadCompletion` fires automatically when no patch is `INITIATED`/`INPROGRESS`.

#### Download Mode Values

| Value | Name | Nginx Behavior | Description |
|-------|------|---------------|-------------|
| `1` | `probe` | Standard `alias $store` | Legacy — each Probe downloads from vendor independently |
| `2` | `summary-caching` | Self-proxy + `proxy_cache patch_cache` | SS downloads centrally; Probe Nginx caches locally |
| `3` | `summary-routing` | Self-proxy + `proxy_cache off` | SS downloads centrally; Probe acts as pass-through |

---

### 2.5 SyMParameter Keys

> Stored in existing `SyMParameter` table (key-value store). Only one new key is introduced. The common store path reuses the existing `store.loc` key (set via `/dcapi/patch/settings/storeLocation`). Timeout and cleanup grace period are not configurable via SyMParameter — they use hardcoded defaults.

| Key | Example Value | New? | Written By | Read By | Description |
|-----|---------------|------|-----------|---------|-------------|
| `store.loc` | `\\\\NAS\\PatchStore` | **EXISTING** | SS admin via `PUT /dcapi/patch/settings/storeLocation` | `CentralizedDownloadUtil.getCommonStorePath()`, Nginx template `%store.loc%`, websettings.conf `store.loc` | When `DOWNLOAD_MODE >= 2`, this is the common store path. Same key used today for Probe's local patch store. |
| `probe_cache_max_size` | `50g` | **NEW** | SS admin via `PUT /dcapi/patch/settings/cleanupSettings` | Nginx template `%centralized.cache.max.size%` | Max disk for Probe-side Nginx `proxy_cache` (only relevant for mode=2) |

**Params explicitly NOT introduced:**

| Eliminated Key | Why Not Needed |
|----------------|----------------|
| `store.loc` | Reuse existing `store.loc` — same purpose, same API, same infrastructure |
| `on_demand_timeout_minutes` | Hardcoded default (30 min). Not admin-configurable in initial release. |
| `cleanup_grace_period_minutes` | Hardcoded default (30 min). Not admin-configurable in initial release. |

---

### 2.6 websettings.conf Keys

> Written by `WebServerUtil.addOrUpdateProperty()`. Read by Nginx template resolution at startup. 5 new keys introduced; `store.loc` is existing (reused as common store path when mode ≥ 2).

| Key | New? | Default (mode=1) | Mode 2 (caching) | Mode 3 (routing) | Purpose |
|-----|------|-------------------|-------------------|-------------------|---------|
| `store.loc` | **EXISTING** | Local patch store | `\\NAS\PatchStore` | `\\NAS\PatchStore` | Reused as common store path. Nginx `alias` in internal server reads from this. |
| `centralized.download.enabled` | **NEW** | `#` | (empty) | (empty) | Prefix for centralized Nginx lines |
| `centralized.download.disabled` | **NEW** | (empty) | `#` | `#` | Prefix for standard `/store/` alias lines |
| `centralized.cache.enabled` | **NEW** | `#` | (empty) | `#` | Prefix for `proxy_cache patch_cache` line |
| `centralized.cache.disabled` | **NEW** | `#` | `#` | (empty) | Prefix for `proxy_cache off` line |
| `centralized.cache.max.size` | **NEW** | N/A | `50` (GB) | N/A | `max_size` for `proxy_cache_path` |

---

### 2.7 Table Summary

| Table | Location | Type | DDL File |
|-------|----------|------|----------|
| `PatchStoreCleanupSettings` | SS + Probe | **Modified** (add `DOWNLOAD_MODE`) | `data-dictionary.xml` |
| `PatchStoreLocation` | SS | **Reused from Probe (no schema changes)** — `DOWNLOAD_TIME` repurposed for soft-delete grace period | `data-dictionary-ss.xml` |
| `RedhatCertDetails` | SS | **Reused from Probe (no schema changes)** — no `SOURCE_PROBE_ID` needed | `data-dictionary-ss.xml` |
| `SuseProductKeys` | SS | **Reused from Probe (no schema changes)** — no `SOURCE_PROBE_ID` needed | `data-dictionary-ss.xml` |
| `SuseAuthTokens` | SS | **Reused from Probe** (populated by `SuseAuthtokenTask`) | `data-dictionary-ss.xml` |
| `PatchKeystoreDetails` | SS | **Reused from Probe** (PKCS12 keystore) | `data-dictionary-ss.xml` |
| `PATCHSTORELOCATION` | Probe | **Existing (no changes)** | — |

**Tables explicitly NOT created:**

| Table | Why Not |
|-------|---------|
| `SSRedhatCertDetails` | Reuse Probe's `RedhatCertDetails` — same schema. `SOURCE_PROBE_ID` is pure audit; use logging instead. |
| `SSSuseProductKeys` | Reuse Probe's `SuseProductKeys` — same schema. `SOURCE_PROBE_ID` and `LAST_SYNCED` are audit; use logging. |
| `PatchStoreLocation (SS)` | Reuse existing `PatchStoreLocation` — same schema, same DAO, same `ConfigStatusDefn` pattern. Checksum in `BINARYCHECKSUMVALUES`. Soft-delete grace via repurposed `DOWNLOAD_TIME`. |
| `CentralizedDownloadSettings` | `DOWNLOAD_MODE` on existing `PatchStoreCleanupSettings` + SyMParameter keys |
| `OnDemandDownloadRequest` | SS dedup via `PatchStoreLocation.STATUS_ID` check (§10.2). No tracking rows. |
| `OnDemandPatchRequest` | Same — eliminated. |
| `CollectionPendingPatches` (Probe) | Pending state in-memory in `PatchDownloadListener` maps. Rebuilt on restart from collection definition (§9.5). |

---

## 3. REST APIs

### 3.1 URL Convention & Auth

**Base path:** All centralized download APIs use `/dcapi/centralizedDownload` **except** settings, which use the existing cleanup settings path.

**Auth patterns:**

| Caller | Auth Mechanism | Headers |
|--------|---------------|---------|
| **SS Admin (browser)** | Session cookie + CSRF | Standard DC session auth |
| **Probe → SS** (PushToSummaryProcessor) | API key headers | `SUMMARY_API_KEY`, `PROBE_ID`, `HS_KEY`, `PROBE_NAME`, `SUMMARY_SERVER_REQUEST`, `USER_DOMAIN` |
| **SS → Probe** (REST call) | SS auth headers | `SUMMARY_API_KEY`, `PROBE_ID` |
| **DS/Agent → Probe Nginx** | `auth_request` subrequest | Agent.key / basic auth |
| **Probe → SS Nginx** | `auth_request` subrequest | `SUMMARY_API_KEY`, `PROBE_ID`, `HS_KEY` |

**API Summary:**

| # | Method | Endpoint | Caller | Use Case |
|---|--------|----------|--------|----------|
| 1 | `GET` | `/dcapi/patch/settings/cleanupSettings` | SS Admin | Fetch settings (extended with `downloadMode`, `probeCacheMaxSize`) |
| 2 | `PUT` | `/dcapi/patch/settings/cleanupSettings` | SS Admin | Update settings (triggers customer metadata XML generation) |
| 3 | `PUT` | `/dcapi/patch/settings/storeLocation` | SS Admin | Set common store path (existing API — used to persist `commonStoreDir`) |
| 3 | `POST` | `/dcapi/centralizedDownload/validateStore` | SS Admin | Validate store path (writable, space) |
| 4 | `POST` | `/dcapi/centralizedDownload/onDemandRequest` | Probe → SS | Request priority download of missing patches |
| 5 | `POST` | `/dcapi/centralizedDownload/reportCorrupted` | Probe → SS | Report checksum-invalid patch in common store |
| 6 | `POST` | `/dcapi/centralizedDownload/dependencyPackages` | Probe → SS | Forward Linux dependency package metadata |
| 7 | `POST` | `/dcapi/centralizedDownload/redhatCert` | Probe → SS | Forward RedHat mTLS certificate |
| 8 | `POST` | `/dcapi/centralizedDownload/suseKeys` | Probe → SS | Forward SUSE registration codes |
| 9 | `POST` | `/dcapi/centralizedDownload/upload` | Probe → SS / SS Admin | Upload patch binary to common store |
| 10 | `POST` | `/dcapi/centralizedDownload/patchesRedownload` | SS Admin | Re-trigger download for failed patches |
| 11 | `DELETE` | `/dcapi/centralizedDownload/patches` | SS Admin | Soft-delete patches (grace period before physical removal) |
| 12 | `POST` | `/dcapi/centralizedDownload/dependencyRedownload` | SS Admin | Re-trigger download for failed dependency packages |
| 13 | `DELETE` | `/dcapi/centralizedDownload/dependency` | SS Admin | Delete dependency packages from common store |
| 14 | `GET` | `/dcapi/centralizedDownload/stats` | SS Admin | Common store statistics |
| 15 | `GET` | `/dcapi/centralizedDownload/probeStatus` | SS Admin | Per-Probe sync/access status |
| 16 | `POST` | `/dcapi/centralizedDownload/retryOnDemand/{collectionId}` | SS Admin | Re-send on-demand request for stuck collection |
| 17 | `POST` | `/dcapi/centralizedDownload/cancelDeployment/{collectionId}` | SS Admin | Cancel stuck collection |
| 18 | `PUT` | `/dcapi/centralizedDownload/changePath` | SS Admin | Change common store path (validate → migrate → switch) |
| 19 | `GET` | `/dcapi/centralizedDownload/settings` | Probe → SS | Probe pulls centralized settings on startup/sync |

---

### 3.2 Settings APIs (Existing Cleanup Settings — Extended)

> Settings are managed through the **existing** `PatchCleanupSettingsController`, not a separate endpoint. The `CleanupSettings` model is extended with `downloadMode` and `probeCacheMaxSize` fields. The common store path is set via the existing `/dcapi/patch/settings/storeLocation` API.

#### GET `/dcapi/patch/settings/cleanupSettings`

Returns current settings. Extended response includes centralized download fields.

**Auth:** SS Admin session

**Response `200 OK`:**

```json
{
    "cleanupId": 1,
    "notifyIfSpaceExceed": false,
    "maxSpaceLimit": -1,
    "supersedeCleanup": true,
    "deleteNotRequiredPatches": false,
    "deletePatchesByPeriod": 6,
    "downloadMode": 1,
    "probeCacheMaxSize": "50g"
}
```

| Field | Type | Values | Description |
|-------|------|--------|-------------|
| `downloadMode` | int | `1` / `2` / `3` | 1=Probe download, 2=SS+caching, 3=SS+routing |
| `probeCacheMaxSize` | string | e.g. `"50g"` | Probe Nginx cache size (only relevant for mode=2) |

---

#### PUT `/dcapi/patch/settings/cleanupSettings`

Updates settings. When `downloadMode` changes, writes `DOWNLOAD_MODE` column + SyMParameters + generates customer metadata XML for Probe propagation.

**Auth:** SS Admin session

**Request:**

```json
{
    "cleanupId": 1,
    "notifyIfSpaceExceed": false,
    "maxSpaceLimit": -1,
    "supersedeCleanup": true,
    "deleteNotRequiredPatches": false,
    "deletePatchesByPeriod": 6,
    "downloadMode": 2,
    "probeCacheMaxSize": "50g"
}
```

**Response `200 OK`:**

```json
{
    "status": "success",
    "message": "Settings updated. Server restart required for mode change to take effect.",
    "restartRequired": true
}
```

**Response `400 Bad Request`:**

```json
{
    "status": "error",
    "errorCode": "STORE_PATH_REQUIRED",
    "message": "Common store path must be set via /dcapi/patch/settings/storeLocation before enabling download mode 2 or 3"
}
```

**Validation rules:**
- `downloadMode` must be `1`, `2`, or `3`
- When `downloadMode >= 2`: `store.loc` in websettings.conf must already be set (via storeLocation API)
- When `downloadMode == 2`: `probeCacheMaxSize` must be a valid Nginx size string (e.g. `"50g"`)

**Side effects on save:**
1. Persist `DOWNLOAD_MODE` to `PatchStoreCleanupSettings` via `PatchCleanupSettingsUtil`
2. Persist `probe_cache_max_size` to SyMParameter via `SyMUtil.updateSyMParameter()`
3. Update `websettings.conf` via `WebServerUtil.addOrUpdateProperty()` (all 6 keys from §2.6)
4. Generate `centralized-download-settings.xml` via `PatchMetaUtil.addOrUpdateCustomerMeta()` → `DCMetaDataUtil.generateCustomerMetaDataForAllManagedCustomers()`
5. Generate `PatchStoreCleanupXmlGenerator.generateXML()` → propagates to DS via MasterRepository
6. **⚠️ SS restart required** for Nginx config to regenerate from templates

---

#### PUT `/dcapi/patch/settings/storeLocation`

**Existing API** — used to set the common store path. Same API that today sets the Probe's local patch store location. When centralized download is enabled (`downloadMode >= 2`), this path is the common store directory that all Probes access.

**Auth:** SS Admin session

**Request:**

```json
{
    "storeLocation": "\\\\NAS\\PatchStore"
}
```

**Response `200 OK`:**

```json
{
    "status": "success",
    "message": "Store location updated"
}
```

**Side effects:**
1. Validate path: exists, writable, sufficient space (via `WebServerUtil.hasWriteAccess()`)
2. Persist to `websettings.conf` as `store.loc` via `WebServerUtil.storeProperWebServerSettings()`
3. Write `tempstoredir` SyMParameter (marker for pending path change)
4. **⚠️ SS restart required** — new path takes effect after restart (same as existing store path change behavior)

---

### 3.3 Store Validation API

#### POST `/dcapi/centralizedDownload/validateStore`

Dry-run validation of a proposed store path — checks writable + sufficient disk space on SS. Does not persist.

**Auth:** SS Admin session

**Request:**

```json
{
    "commonStorePath": "\\\\NAS\\PatchStore"
}
```

**Response `200 OK`:**

```json
{
    "valid": true,
    "totalSpaceGB": 500,
    "freeSpaceGB": 320,
    "writable": true
}
```

**Response `200 OK` (validation failed — not an HTTP error):**

```json
{
    "valid": false,
    "reason": "NOT_WRITABLE",
    "message": "Path exists but is not writable by the service account"
}
```

---


### 3.4 Probe → SS APIs (via PushToSummaryProcessor)

> These endpoints are called by Probes via `PushToSummaryProcessor` (push-to-summary queue, DB-backed, async, single-threaded). Auth: Probe API key headers. Probes **do not inspect HTTP responses** — all result discovery is via polling the common store.

#### POST `/dcapi/centralizedDownload/onDemandRequest`

Probe requests priority download of missing patches for a deployment.

**Auth:** Probe API key headers

**Request:**

```json
{
    "patchIds": [101, 102, 103],
    "collectionId": 12345,
    "probeId": 1001,
    "requestTime": 1740000000000
}
```

**Response `200 OK`:**

```json
{
    "accepted": [101, 103],
    "alreadyAvailable": [102],
    "estimatedTimeMinutes": 5
}
```

| Field | Description |
|-------|-------------|
| `accepted` | Patches queued for priority download (not yet in common store) |
| `alreadyAvailable` | Patches already `STATUS=AVAILABLE` in `PatchStoreLocation (SS)` |
| `estimatedTimeMinutes` | Rough ETA based on queue depth |

**SS processing:**
1. Check `PatchStoreLocation (SS)` → split `accepted` vs `alreadyAvailable`
2. For accepted where `STATUS = FAILED`: clear `.failed` marker, reset to `QUEUED`
3. Skip if `STATUS IN (QUEUED, DOWNLOADING)` — already being handled
4. Queue accepted patches with **highest priority** (front of download queue)
5. **No tracking rows** — dedup via SS `PatchStoreLocation.STATUS_ID`
6. **No push event** — Probe discovers result via 5-min polling scheduler

---

#### POST `/dcapi/centralizedDownload/reportCorrupted`

Probe reports that a patch file in the common store has an invalid checksum.

**Auth:** Probe API key headers

**Request:**

```json
{
    "patchId": 205,
    "collectionId": 12345,
    "probeId": 1001,
    "corruptedLanguageIds": [1, 3],
    "requestTime": 1740000000000
}
```

**Response `200 OK`:**

```json
{
    "status": "success",
    "message": "Corrupted files will be deleted and re-queued for download"
}
```

**SS processing:**
1. Delete corrupted file(s) from common store
2. Delete any `.failed` markers for the affected language IDs
3. Reset SS `PatchStoreLocation.STATUS_ID` to `DLOAD_REQUESTED`
4. Re-queue download from vendor with highest priority
5. Probe's polling scheduler continues monitoring

---

#### POST `/dcapi/centralizedDownload/dependencyPackages`

Probe forwards Linux dependency package metadata to SS.

**Auth:** Probe API key headers

**Request:**

```json
{
    "probeId": 1001,
    "packages": [
        {
            "packageId": 1,
            "productId": 300180,
            "packageName": "iputils-ping_20190709-3ubuntu1_amd64.deb",
            "checksum": "ce08339e42c42bd624113b5cbf33110797e0241bdb3e3b65c5fb7bb058bf7be0",
            "checksumType": "sha256",
            "downloadUrl": "http://archive.ubuntu.com/ubuntu/pool/main/i/iputils/iputils-ping_20190709-3ubuntu1_amd64.deb",
            "osFlavor": "ubuntu"
        }
    ]
}
```

**Response `200 OK`:**

```json
{
    "status": "success",
    "inserted": 1,
    "duplicatesSkipped": 0
}
```

**SS processing:**
1. Dedup-insert into SS-side `PACKAGEINFO` on `(package_name, product_id, checksum)`
2. Trigger `SSDependencyDownloadTask` for affected flavor (async, immediate)
3. Scheduled fallback: task runs every 10 min regardless

---

#### POST `/dcapi/centralizedDownload/redhatCert`

Probe forwards a RedHat mTLS certificate for SS-side CDN authentication.

**Auth:** Probe API key headers
**Content-Type:** `multipart/form-data` (via `MultiPartUtilImpl`)

**Request (multipart):**

| Part | Type | Description |
|------|------|-------------|
| `certFile` | Binary (ZIP) | Archive containing `client.pem`, `client-key.pem`, `ca.pem` |
| `edition` | Text | `Server` / `Workstation` / `Desktop` |
| `probeId` | Text | Source Probe ID |
| `certExpiry` | Text | Epoch ms of certificate expiry |

**Response `200 OK`:**

```json
{
    "status": "success",
    "edition": "Server",
    "keystoreAlias": "patch_keystore_server",
    "message": "RedHat certificate stored successfully"
}
```

**SS processing:**
1. Check existing cert for this edition: if current cert has later expiry → skip
2. Extract ZIP → import PEMs into PKCS12 keystore via `PatchKeystoreService`
3. UPSERT `RedhatCertDetails` by `EDITION`
4. Store keystore password in `PatchKeystoreDetails`

---

#### POST `/dcapi/centralizedDownload/suseKeys`

Probe forwards SUSE registration codes to SS.

**Auth:** Probe API key headers

**Request:**

```json
{
    "probeId": 1001,
    "customerId": 5001,
    "keys": [
        {
            "productKey": "XXXX-XXXX-XXXX-XXXX",
            "osEdition": "server",
            "productVersion": "15.4"
        }
    ]
}
```

**Response `200 OK`:**

```json
{
    "status": "success",
    "inserted": 1,
    "updated": 0
}
```

**SS processing:**
1. UPSERT `SuseProductKeys` on `(PRODUCT_KEY, CUSTOMER_ID)`
2. Run `SuseAuthtokenTask` to fetch auth tokens from `scc.suse.com`
3. Tokens stored in `SuseAuthTokens`, consumed by `SuseSettingsUtil.appendSUSEToken()` at download time

---

#### POST `/dcapi/centralizedDownload/upload`

Accepts multipart upload; stores binary in common store. Used by both SS admin directly and Probe's `ProbeUploadForwarder`.

**Auth:** Probe API key headers or SS Admin session
**Content-Type:** `multipart/form-data`

**Request (multipart):**

| Part | Type | Description |
|------|------|-------------|
| `file` | Binary | Patch binary file |
| `patchId` | Text | Patch identifier |
| `languageId` | Text | Language variant (`1`=English, `0`=all) |
| `fileName` | Text | Target file name in common store |
| `checksum` | Text | Expected SHA256 checksum |
| `probeId` | Text | Source Probe ID (or `"SS"` for direct upload) |

**Response `200 OK`:**

```json
{
    "status": "success",
    "patchId": 400010,
    "storedAs": "400010-custom-patch.exe",
    "checksumValid": true
}
```

**Response `400 Bad Request`:**

```json
{
    "status": "error",
    "errorCode": "CHECKSUM_MISMATCH",
    "message": "Uploaded file checksum does not match expected value"
}
```

**SS processing:**
1. Store binary in `{commonStore}/{fileName}`
2. Validate checksum (SHA256)
3. Update `PatchStoreLocation` (SS) with `STATUS_ID = AVAILABLE`
4. **No push event** — other Probes discover via 30-min missing-patch scheduler (§17.2)
5. Uploading Probe gets result synchronously via REST response

---

### 3.5 Patch Store Management APIs (SS Admin)

#### POST `/dcapi/centralizedDownload/patchesRedownload`

Re-triggers download for selected patches.

**Auth:** SS Admin session

**Request:**

```json
{
    "patchIds": [101, 205, 310]
}
```

**Response `200 OK`:**

```json
{
    "status": "success",
    "requeued": 3,
    "message": "3 patches queued for re-download"
}
```

**Side effects:**
1. Reset `PatchStoreLocation.STATUS_ID` to `DLOAD_REQUESTED` for each patch
2. Delete any `.failed` markers: `{commonStore}/.patch-status/{patchId}_{langId}.failed`
3. Queue patches in `ss-patch-download-data`

---

#### DELETE `/dcapi/centralizedDownload/patches`

Soft-delete patches. Physical removal after grace period by `DeferredCleanupTask`.

**Auth:** SS Admin session

**Request:**

```json
{
    "patchIds": [101, 205]
}
```

**Response `200 OK`:**

```json
{
    "status": "success",
    "markedForDeletion": 2,
    "gracePeriodMinutes": 30,
    "message": "2 patches marked for deletion. Physical removal after 30 minutes."
}
```

**Side effects:**
1. Set `PatchStoreLocation.STATUS_ID = PENDING_DELETE`
2. Set `DOWNLOAD_TIME = System.currentTimeMillis()` (repurposed as deletion-mark time)
3. `DeferredCleanupTask` physically deletes after `cleanup_grace_period_minutes`
4. After physical deletion: regenerate `deleted-patches.xml` in common store

---

### 3.6 Dependency Package APIs (SS Admin)

#### POST `/dcapi/centralizedDownload/dependencyRedownload`

Re-triggers download for selected dependency packages.

**Auth:** SS Admin session

**Request:**

```json
{
    "packageIds": [1, 2, 3]
}
```

**Response `200 OK`:**

```json
{
    "status": "success",
    "requeued": 3
}
```

---

#### DELETE `/dcapi/centralizedDownload/dependency`

Delete dependency packages from common store.

**Auth:** SS Admin session

**Request:**

```json
{
    "packageIds": [1, 2]
}
```

**Response `200 OK`:**

```json
{
    "status": "success",
    "deleted": 2
}
```

---

### 3.7 Monitoring & Admin APIs

#### GET `/dcapi/centralizedDownload/stats`

Returns common store statistics.

**Auth:** SS Admin session

**Response `200 OK`:**

```json
{
    "totalPatches": 1250,
    "byStatus": {
        "DLOAD_REQUESTED": 15,
        "DLOAD_RUNNING": 3,
        "AVAILABLE": 1200,
        "DLOAD_FAILED": 12,
        "PENDING_DELETE": 8,
        "DELETED": 12
    },
    "totalFileSizeBytes": 53687091200,
    "totalFileSizeFormatted": "50.0 GB",
    "diskUsage": {
        "totalSpaceGB": 500,
        "freeSpaceGB": 320,
        "usedByStoreGB": 50
    }
}
```

---

#### GET `/dcapi/centralizedDownload/probeStatus`

Returns per-Probe connectivity status.

**Auth:** SS Admin session

**Response `200 OK`:**

```json
{
    "probes": [
        {
            "probeId": 1001,
            "probeName": "Probe-US-East",
            "online": true,
            "lastSettingsSyncTime": 1713168000000
        },
        {
            "probeId": 1002,
            "probeName": "Probe-EU-West",
            "online": true,
            "lastSettingsSyncTime": 1713167000000
        }
    ]
}
```

---

#### POST `/dcapi/centralizedDownload/retryOnDemand/{collectionId}`

Re-sends an on-demand request for a stuck collection.

**Auth:** SS Admin session

**Response `200 OK`:**

```json
{
    "status": "success",
    "collectionId": 12345,
    "message": "On-demand request re-sent for collection 12345"
}
```

---

#### POST `/dcapi/centralizedDownload/cancelDeployment/{collectionId}`

Cancels a stuck collection.

**Auth:** SS Admin session

**Response `200 OK`:**

```json
{
    "status": "success",
    "collectionId": 12345,
    "message": "Deployment cancelled for collection 12345"
}
```

---

### 3.8 Store Path Change API

#### PUT `/dcapi/centralizedDownload/changePath`

Changes the common store path while centralized download remains enabled.

**Auth:** SS Admin session

**Request:**

```json
{
    "newCommonStorePath": "\\\\NAS2\\PatchStore",
    "migrateFiles": true
}
```

**Response `200 OK` (migration started):**

```json
{
    "status": "success",
    "phase": "MIGRATION_STARTED",
    "message": "File migration in progress. SS restart required after completion.",
    "filesToMigrate": 1200,
    "estimatedSizeGB": 50
}
```

**Response `200 OK` (migrateFiles=false):**

```json
{
    "status": "success",
    "phase": "SWITCH_COMPLETE",
    "message": "Path changed. All PatchStoreLocation entries reset to DLOAD_REQUESTED. SS restart required.",
    "patchesReset": 1200
}
```

**Processing phases:** See system-design.md §16.4.2 for Phase A (Validate) → Phase B (Migrate) → Phase C (Atomic Switch).

> **Note:** Probe access to new path is NOT validated programmatically. The UI displays a warning instructing the admin to ensure all Probes have file-level access to the new path before proceeding.

---

### 3.9 Probe Settings Pull API

#### GET `/dcapi/centralizedDownload/settings`

Probe pulls centralized download settings from SS at startup and after DB sync.

**Auth:** Probe API key headers

**Response `200 OK`:**

```json
{
    "downloadMode": 2,
    "commonStoreDir": "\\\\NAS\\PatchStore",
    "probeCacheMaxSize": "50g"
}
```

**Probe-side processing:**
1. Write `DOWNLOAD_MODE` to local `PatchStoreCleanupSettings`
2. Write `store.loc` and `probe_cache_max_size` to local SyMParameter
3. If `DOWNLOAD_MODE` changed: trigger Nginx config regeneration + reload

---

### 3.10 Nginx Auth Servlets

> **Not REST APIs** — mapped as servlets for Nginx `auth_request` subrequests. Return HTTP status codes only (no body).

#### Probe-side: `GET /common-store-auth`

| Aspect | Detail |
|--------|--------|
| **Location** | Probe Nginx → `auth_request` subrequest for `/store/` |
| **Validates** | DS/Agent credentials (Agent.key / basic auth) |
| **Returns** | `200` (allow) or `401` (deny) |
| **Implementation** | Servlet in Probe's `web.xml` |

#### SS-side: `GET /common-store-auth`

| Aspect | Detail |
|--------|--------|
| **Location** | SS Nginx → `auth_request` subrequest for `/common-store/` |
| **Validates** | `SUMMARY_API_KEY`, `PROBE_ID`, `HS_KEY` against `SUMMARYSERVERAPIKEYDETAILS` |
| **Returns** | `200` (allow) or `401` (deny) |
| **Implementation** | `SSStoreAuthValidator` servlet |

---

## 4. Nginx Configuration

### 4.1 SS-Side Nginx

SS Nginx serves the common store as static files with `open_file_cache` (not `proxy_cache`). DS/Agents do **not** download from SS directly — they use Probe `/store/`. SS `/common-store/` is for Probe→SS fallback only.

```nginx
open_file_cache          max=2000 inactive=24h;
open_file_cache_valid    1h;
open_file_cache_min_uses 1;
open_file_cache_errors   on;

location /common-store/ {
    alias %ss.common.store.loc%/;
    sendfile           on;
    tcp_nopush         on;
    tcp_nodelay        on;
    output_buffers     4 256k;
    add_header Cache-Control "public, max-age=2592000, immutable" always;
    etag on;
    limit_rate_after   50m;
    limit_rate         100m;
    gzip off;
    auth_request /common-store-auth;
    add_header Content-Disposition "attachment" always;
    limit_conn common_store_conn 10;
    limit_conn common_store_total 200;
    limit_conn_status 503;
}
```

### 4.2 Probe-Side Nginx (Self-Proxy Pattern)

#### Block 1: `http {}` context — `nginx-pre-product.conf.template`

```nginx
%centralized.download.enabled%proxy_cache_path "%store.loc%/../../NginxCache/patch-store-cache"
%centralized.download.enabled%    levels=1:2
%centralized.download.enabled%    keys_zone=patch_cache:10m
%centralized.download.enabled%    max_size=%centralized.cache.max.size%g
%centralized.download.enabled%    inactive=30d
%centralized.download.enabled%    use_temp_path=off;

%centralized.download.enabled%server {
%centralized.download.enabled%    listen       127.0.0.1:9090;
%centralized.download.enabled%    server_name  localhost;
%centralized.download.enabled%    location /internal-patch-store/ {
%centralized.download.enabled%        alias "%store.loc%/";
%centralized.download.enabled%        sendfile on;
%centralized.download.enabled%        tcp_nopush on;
%centralized.download.enabled%        tcp_nodelay on;
%centralized.download.enabled%        open_file_cache max=2000 inactive=24h;
%centralized.download.enabled%        open_file_cache_valid 1h;
%centralized.download.enabled%        open_file_cache_min_uses 1;
%centralized.download.enabled%        open_file_cache_errors on;
%centralized.download.enabled%        allow 127.0.0.1;
%centralized.download.enabled%        deny all;
%centralized.download.enabled%    }
%centralized.download.enabled%}

%centralized.download.enabled%map $upstream_cache_status $cache_status_display {
%centralized.download.enabled%    "" "DISABLED";
%centralized.download.enabled%    default $upstream_cache_status;
%centralized.download.enabled%}
```

#### Block 2: Modified `/store/` location — `nginx-static-agent.conf.template`

```nginx
location /store/{
    %nginx.dev.mode%set $loc_match "/store/";
    %disable.http.basic.auth%auth_basic "Private Property";
    auth_basic_user_file ../../conf/Tomcat/Agent.key;
    if ( $ssl_client_verify ~ ^(?!(%client.cert.auth.level%)) ){return 403 "CLIENT CERT AUTH ERROR";}

    # Standard mode (mode=1):
    %centralized.download.disabled%gzip off;
    %centralized.download.disabled%alias $store;
    %centralized.download.disabled%autoindex %nginx.autoindex.value%;
    %centralized.download.disabled%allow all;

    # Centralized mode (mode=2 or 3):
    %centralized.download.enabled%proxy_pass http://127.0.0.1:9090/internal-patch-store/;
    %centralized.download.enabled%%centralized.cache.enabled%proxy_cache patch_cache;
    %centralized.download.enabled%%centralized.cache.disabled%proxy_cache off;
    %centralized.download.enabled%proxy_cache_valid 200 30d;
    %centralized.download.enabled%proxy_cache_valid 404 1m;
    %centralized.download.enabled%proxy_cache_key $uri;
    %centralized.download.enabled%proxy_cache_lock on;
    %centralized.download.enabled%proxy_cache_lock_timeout 300s;
    %centralized.download.enabled%add_header X-Cache-Status $cache_status_display always;
    %centralized.download.enabled%add_header Cache-Control "public, max-age=2592000, immutable" always;
    %centralized.download.enabled%add_header Content-Disposition "attachment" always;
    %centralized.download.enabled%proxy_buffering on;
    %centralized.download.enabled%proxy_max_temp_file_size 0;
    %centralized.download.enabled%proxy_connect_timeout 10s;
    %centralized.download.enabled%proxy_read_timeout 600s;
    %centralized.download.enabled%proxy_send_timeout 600s;
    %centralized.download.enabled%gzip off;
}
```

### 4.3 Template Placeholders & Mode Mapping

| Placeholder | Mode 1 (Probe) | Mode 2 (SS+Caching) | Mode 3 (SS+Routing) |
|-------------|----------------|----------------------|---------------------|
| `%centralized.download.enabled%` | `#` | (empty) | (empty) |
| `%centralized.download.disabled%` | (empty) | `#` | `#` |
| `%centralized.cache.enabled%` | `#` | (empty) | `#` |
| `%centralized.cache.disabled%` | `#` | `#` | (empty) |
| `%centralized.cache.max.size%` | N/A | `50` | N/A |
| `%store.loc%` | Local patch store | `\\NAS\PatchStore` | `\\NAS\PatchStore` |

| X-Cache-Status | Meaning |
|----------------|---------|
| `MISS` | Fetched from common store (network share), now cached on Probe disk |
| `HIT` | Served from local Probe cache |
| `DISABLED` | Caching off — every request reads from network share |

---

## 5. Meta File Structures

### 5.1 Common Store Directory Layout

```
{ss_common_storedir}/
├── KB5001234.msu                            ← Regular Windows patch
├── 400009-mpam-fe-defender-x64.exe          ← Common URL patch
├── office/
│   └── {patchId}/
├── linux/
│   ├── redhat-dependencies/
│   ├── ubuntu-dependencies/
│   ├── suse-dependencies/
│   ├── debian-dependencies/
│   ├── redhat/dependency/
│   │   ├── product-dep-101.xml
│   │   └── ProdDepMetaInfo.xml
│   ├── ubuntu/dependency/
│   ├── suse/dependency/
│   └── debian/dependency/
├── deleted-patches/
│   ├── windows/deleted-patches.xml
│   ├── mac/deleted-patches.xml
│   └── linux/
│       ├── deleted-patches.xml
│       └── deleted-dep-packages.xml
└── .patch-status/
    ├── 102_1.failed
    └── 205_3.failed
```

> **No `.ss-store-sentinel` file.** Probe access validation is not done programmatically — admin is instructed via UI to ensure all Probes have file-level access before enabling.

### 5.2 File Naming Conventions

| File Type | Pattern | Location |
|-----------|---------|----------|
| Regular patch | `{fileName}` | `{commonStore}/` |
| Common URL patch | `{patchId}-{fileName}` | `{commonStore}/` |
| Office Click-to-Run | `{patchId}/` (directory) | `{commonStore}/office/` |
| Linux RPM/DEB dep | `{packageName}.rpm/.deb` | `{commonStore}/linux/{flavor}-dependencies/` |
| Dependency meta | `product-dep-{productId}.xml` | `{commonStore}/linux/{flavor}/dependency/` |
| Dependency index | `ProdDepMetaInfo.xml` | `{commonStore}/linux/{flavor}/dependency/` |
| Deleted patches meta | `deleted-patches.xml` | `{commonStore}/deleted-patches/{platform}/` |
| Deleted deps meta | `deleted-dep-packages.xml` | `{commonStore}/deleted-patches/linux/` |
| Failure marker | `{patchId}_{languageId}.failed` | `{commonStore}/.patch-status/` |

### 5.3 What Lives Where — Summary

#### In the Common Store (all Probes have file-level access)

| Content | Format | Written By | Read By |
|---------|--------|-----------|---------|
| Patch binaries (all types) | Binary | SS `SSPatchDownloadService` | Probe Nginx (via self-proxy) |
| Linux dependency meta XMLs | XML | SS `SSLinuxProductXMLGenerationTask` | Probe (file-level read) |
| Deleted-patches meta XMLs | XML | SS `DeferredCleanupTask` | Probe missing-patch scheduler |
| Per-patch failure markers | Plain text | SS (on download failure) | Probe polling scheduler |

#### NOT in Common Store (per-Probe, generated locally)

| Content | Why Not Shared |
|---------|----------------|
| `patch-products.zip` | Each Probe generates from local DB |
| `Product-{id}.xml` | Contains Probe-specific `WEBPATH` |
| `globalmetadata.xml` | Per-Probe sync timestamps |

---

## 6. Settings Propagation Flow

```
Admin saves settings on SS
  ↓
  ├── 1. PUT /dcapi/patch/settings/storeLocation → store.loc (websettings.conf, existing flow)
  ├── 2. PUT /dcapi/patch/settings/cleanupSettings → DOWNLOAD_MODE + probe_cache_max_size
  │      ├── PatchStoreCleanupSettings.DOWNLOAD_MODE → DB
  │      ├── SyMParameter: probe_cache_max_size → DB
  │      ├── websettings.conf → updated (5 new keys + store.loc already set)
  │      ├── PatchMetaUtil.addOrUpdateCustomerMeta("centralized-download-settings")
  │      │    → centralized-download-settings.xml generated
  │      └── PatchStoreCleanupXmlGenerator.generateXML() → DS propagation
  └── 3. ⚠️ SS restart required

Probe picks up on next metadata sync cycle (or startup):
  ↓
  ├── GET /dcapi/centralizedDownload/settings (direct REST pull)
  ├── Write DOWNLOAD_MODE to local PatchStoreCleanupSettings
  ├── Write store.loc, probe_cache_max_size to local SyMParameter
  └── If DOWNLOAD_MODE changed: trigger Nginx config regeneration + reload
```

> **No push events.** Enable/disable takes effect on next Probe restart or DB sync cycle.
>
> **Probe access is not validated programmatically.** The UI displays: _"Ensure all Probes have file-level access (network share / mount) to the common store path before enabling."_

---

## 7. Reconciliation with Prior LLD Versions

| # | Change | LLD v2 | This LLD v4 | Rationale |
|---|--------|--------|-------------|-----------|
| 1 | **Settings table** | `CentralizedDownloadSettings` | **Eliminated.** `DOWNLOAD_MODE` on existing `PatchStoreCleanupSettings` + SyMParameter. | system-design.md §7.1 |
| 2 | **Settings API** | Dedicated `/dcapi/centralizedDownload` GET/PUT | **Extended existing** `/dcapi/patch/settings/cleanupSettings` | system-design.md §8.1 |
| 3 | **Download mode** | Boolean toggle | **3-way integer** (1/2/3) | system-design.md §7.1 |
| 4 | **Vendor fallback** | `FALLBACK_TO_VENDOR` + API | **Eliminated.** No vendor fallback. | system-design.md §13.3 |
| 5 | **Collection statuses** | `502`, `503` | **No new statuses.** | system-design.md §7.3 |
| 6 | **CollectionPendingPatches** | New Probe table | **Eliminated.** In-memory listener maps. | system-design.md §7.1 |
| 7 | **OnDemandDownloadRequest** | Two SS tables | **Eliminated.** Dedup via `PatchStoreLocation.STATUS_ID`. | system-design.md §10.2 |
| 8 | **Push events** | 4 event types | **All eliminated.** Pure polling. | system-design.md §10.3 |
| 9 | **Settings propagation** | Event push | **Customer metadata XML** | system-design.md §15.1 |
| 10 | **PatchStoreLocation (SS)** | Reused from Probe | **Reused, zero schema changes.** `DOWNLOAD_TIME` repurposed. | Same DAO, same `ConfigStatusDefn`. |
| 11 | **RedhatCertDetails (SS)** | `SSRedhatCertDetails` (new, with SOURCE_PROBE_ID) | **Reused from Probe, no schema changes.** No `SOURCE_PROBE_ID`. | Audit via logging. |
| 12 | **SuseProductKeys (SS)** | `SSSuseProductKeys` (new, with SOURCE_PROBE_ID) | **Reused from Probe, no schema changes.** No `SOURCE_PROBE_ID`. | Audit via logging. |
| 13 | **Probe access validation** | `validateProbeAccess` API + sentinel file | **Eliminated.** UI instructs admin to ensure access. | No programmatic validation needed. |
| 14 | **Corrupted patch API** | Not in v2 | **Added** `POST /dcapi/centralizedDownload/reportCorrupted` | system-design.md §10.5 UC-1.8 |
| 15 | **Store path change API** | Not in v2 | **Added** `PUT /dcapi/centralizedDownload/changePath` | system-design.md §16.4 |
