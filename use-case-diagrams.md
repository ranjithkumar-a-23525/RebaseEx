# Centralized Patch Download — Use Case Flow Diagrams

---

## UC-1.1: Deployment-Triggered Download — Patch Not in Common Store

```mermaid
flowchart TD
    A(["🖥️ PROBE: Deployment Initiated"]) --> B{"🖥️ PROBE: Check Common Store<br/>for all patches"}
    B -->|All found| Z1(["Skip to UC-1.7:<br/>Direct PSL add"])
    B -->|Some missing| C["🖥️ PROBE: Send On-Demand request<br/>to SS via push-to-summary"]
    C --> D["🖥️ PROBE: Update status →<br/>WAITING_FOR_SS_DOWNLOAD 502"]
    D --> E["🖥️ PROBE: Persist to<br/>COLLECTIONPENDINGPATCHES"]
    E --> F["🖥️ PROBE: Start 5-min<br/>Polling Scheduler"]
    F --> G{"🖥️ PROBE: Poll Common Store"}
    G -->|File exists| H["🖥️ PROBE: Mark download success<br/>Add to PSL<br/>Remove from pending"]
    G -->|.failed marker exists| I["🖥️ PROBE: Read failure reason<br/>Mark DOWNLOAD_FAILED"]
    G -->|Neither exists| J{"🖥️ PROBE: Timeout reached?<br/>30 min / 6 ticks"}
    J -->|No| G
    J -->|Yes| K["🖥️ PROBE: Mark DOWNLOAD_FAILED<br/>Reason: SS timed out"]
    H --> L{"🖥️ PROBE: All patches ready?"}
    L -->|No| G
    L -->|Yes| M["🖥️ PROBE: Generate patch-products.zip<br/>Deploy to DS/Agents]
    M --> N(["🖥️ PROBE: Cancel Polling ✓"])
    I --> N
    K --> N

    style A fill:#4CAF50,color:#fff
    style N fill:#2196F3,color:#fff
    style I fill:#f44336,color:#fff
    style K fill:#f44336,color:#fff
```

---

## UC-1.2: Agent-Requested Patch Download

```mermaid
flowchart TD
    A(["🖥️ PROBE: Agent requests patch<br/>not in PSL"]) --> B["🖥️ PROBE: Forward request to SS<br/>via push-to-summary"]
    B --> C["🖥️ PROBE: Add to collection-to-patch<br/>status map"]
    C --> D["🖥️ PROBE: Polling Scheduler monitors<br/>common store"]
    D --> E{"🖥️ PROBE: Check Common Store"}
    E -->|File exists| F["🖥️ PROBE: Mark success → Add to PSL<br/>Fire post-download callbacks"]
    E -->|.failed marker| G["🖥️ PROBE: Mark failure<br/>Post-download activity"]
    E -->|Timeout| H["🖥️ PROBE: Mark failed<br/>Post-download activity"]
    E -->|Neither| D

    I["🖥️ PROBE: Missing Patch Scheduler<br/>every 30 min"] -.->|May discover<br/>file first| F

    style A fill:#4CAF50,color:#fff
    style F fill:#2196F3,color:#fff
    style G fill:#f44336,color:#fff
    style H fill:#f44336,color:#fff
    style I fill:#FF9800,color:#fff
```

---

## UC-1.3: Missing Patch Scheduler — Background Discovery

```mermaid
flowchart TD
    A(["🖥️ PROBE: Scheduler fires<br/>every 30 min"]) --> B["🖥️ PROBE: Query SCOPEAFFPATCHSTATUS<br/>LEFT JOIN PATCHSTORELOCATION"]
    B --> C["🖥️ PROBE: Find patches needed<br/>but not in PSL"]
    C --> D{"🖥️ PROBE: For each missing patch"}
    D --> E{"🖥️ PROBE: File exists in<br/>common store?"}
    E -->|"Yes & size > 0"| F["🖥️ PROBE: Add to PSL directly<br/>Fire post-download callbacks<br/>Log: found in common store"]
    E -->|No| G["Skip — SS will handle<br/>via SSDownloadMissingPatchTask<br/>or on-demand"]
    F --> D
    G --> D
    D -->|All processed| H(["🖥️ PROBE: Done ✓"])

    style A fill:#FF9800,color:#fff
    style H fill:#2196F3,color:#fff
```

> **Note:** This scheduler is a passive scanner — it never sends download requests to SS.

---

## UC-1.4: Redownload Triggered for Patch

```mermaid
flowchart TD
    A(["🖥️ PROBE: Admin triggers<br/>Redownload"]) --> B["🖥️ PROBE: Forward to SS via<br/>push-to-summary<br/>highest priority"]
    B --> C["🖥️ PROBE: Add to download listener"]
    C --> D["🖥️ PROBE: Polling Scheduler monitors"]
    D --> E{"🖥️ PROBE: Check Common Store"}
    E -->|File exists| F(["🖥️ PROBE: Success ✓"])
    E -->|.failed marker| G(["🖥️ PROBE: Failure ✗"])
    E -->|Timeout| H(["🖥️ PROBE: Failure — timed out ✗"])
    E -->|Neither| D

    style A fill:#4CAF50,color:#fff
    style F fill:#2196F3,color:#fff
    style G fill:#f44336,color:#fff
    style H fill:#f44336,color:#fff
```

---

## UC-1.5: Download Failure on SS

```mermaid
flowchart TD
    A(["🌐 SS: Download from<br/>vendor fails"]) --> B{"🌐 SS: Retry exhausted?<br/>3 retries for connection<br/>Immediate for 404/checksum"}
    B -->|No| C["🌐 SS: Retry download"]
    C --> B
    B -->|Yes| D["🌐 SS: Update SSPATCHSTORELOCATION<br/>STATUS = FAILED"]
    D --> E["🌐 SS: Write failure marker:<br/>.patch-status/{patchId}_{langId}.failed"]
    E --> F["🌐 SS: No push event to Probe"]
    F --> G["🖥️ PROBE: Polling detects<br/>.failed marker ≤5 min"]
    G --> H["🖥️ PROBE: Reads failure reason"]
    H --> I["🖥️ PROBE: Mark patch as failed<br/>Collection → DOWNLOAD_FAILED"]

    style A fill:#f44336,color:#fff
    style I fill:#f44336,color:#fff
```

---

## UC-1.6: Failure Marker Already Exists

```mermaid
flowchart TD
    A(["🖥️ PROBE: Encounters patch<br/>with .failed marker"]) --> B{"🖥️ PROBE: Already in<br/>collectionToPatchStatusMap?"}
    B -->|"No — Deployment/Agent"| C["🖥️ PROBE: Send on-demand request to SS"]
    C --> D["🌐 SS: Detects FAILED status →<br/>Clear .failed marker →<br/>Reset to QUEUED →<br/>Re-queue highest priority"]
    D --> E["🖥️ PROBE: Polls normally"]
    B -->|Yes| F["🖥️ PROBE: Skip duplicate request<br/>Continue polling"]
    
    G(["🖥️ PROBE: Redownload trigger"]) --> H["🌐 SS: Delete failure marker<br/>unconditionally"]
    H --> I["🖥️ PROBE: Send redownload to SS<br/>Normal flow resumes"]

    style A fill:#FF9800,color:#fff
    style G fill:#FF9800,color:#fff
```

---

## UC-1.7: Patch Already Available — Checksum Valid

```mermaid
flowchart TD
    A(["🖥️ PROBE: Patch found in<br/>Common Store"]) --> B{"🖥️ PROBE: Validate checksum"}
    B -->|Regular patch| C["🖥️ PROBE: SHA256 from BINARYCHECKSUMVALUES<br/>MD5 fallback for >4GB"]
    B -->|Linux dep pkg| D["🖥️ PROBE: Checksum from PACKAGEINFO<br/>or meta XML"]
    C --> E{"🖥️ PROBE: Valid?"}
    D --> E
    E -->|"Yes ✓"| F["🖥️ PROBE: Skip download entirely<br/>Add to PSL<br/>Post-download activity"]
    E -->|"No ✗"| G(["Go to UC-1.8:<br/>Checksum Invalid"])

    style A fill:#4CAF50,color:#fff
    style F fill:#2196F3,color:#fff
    style G fill:#f44336,color:#fff
```

---

## UC-1.8: Patch Available but Checksum Invalid

```mermaid
flowchart TD
    A(["🖥️ PROBE: Checksum mismatch<br/>detected"]) --> B["🖥️ PROBE: Send corruption<br/>report to SS"]
    B --> C["🌐 SS: Delete corrupted file<br/>+ failure markers"]
    C --> D["🌐 SS: Re-queue download<br/>from vendor — highest priority"]
    D --> E["🖥️ PROBE: Polling resumes<br/>Normal flow"]

    style A fill:#f44336,color:#fff
    style E fill:#2196F3,color:#fff
```

---

## UC-1.9: Download Timeout

```mermaid
flowchart TD
    A(["🖥️ PROBE: Polling reaches maxPolls<br/>6 ticks × 5 min = 30 min"]) --> B["🖥️ PROBE: Mark remaining patches<br/>as download failure"]
    B --> C["🖥️ PROBE: Collection → DOWNLOAD_FAILED<br/>Reason: SS timed out after {N} min"]
    C --> D["🖥️ PROBE: Cancel Polling Scheduler"]
    D --> E["🖥️ PROBE: No vendor fallback<br/>Admin must retry via<br/>RETRY_ON_DEMAND API"]

    style A fill:#f44336,color:#fff
    style E fill:#FF9800,color:#fff
```

---

## UC-2: Download Status Visibility on Probe

```mermaid
flowchart TD
    A(["🖥️ PROBE: Admin opens<br/>Centralized Download View"]) --> B["🖥️ PROBE: Query local Probe DB"]
    B --> C["🖥️ PROBE: COLLECTIONPENDINGPATCHES<br/>+ collection status tables"]
    C --> D{"🖥️ PROBE: Display"}
    D --> E["🟡 WAITING_FOR_SS_DOWNLOAD<br/>— waiting since, patches required vs available"]
    D --> F["🔴 DOWNLOAD_FAILED<br/>— failure reasons"]
    D --> G["🟢 Completed<br/>— deployment status"]
    E --> H["Actions: RETRY_ON_DEMAND<br/>CANCEL_DEPLOYMENT"]

    style A fill:#4CAF50,color:#fff
```

> No real-time query to SS — all state derived from local Probe DB via polling.

---

## UC-3: RedHat Certificate Forwarding

```mermaid
flowchart TD
    A(["🖥️ PROBE: Agent uploads RedHat cert"]) --> B["🖥️ PROBE: Validate cert<br/>CDN reachability test"]
    B --> C{"🖥️ PROBE: Centralized download<br/>enabled?"}
    C -->|Yes| D["🖥️ PROBE: Forward cert zip + metadata<br/>(edition, expiry) to SS<br/>via MultiPartUtilImpl"]
    C -->|No| Z(["Legacy flow"])
    D --> E{"🌐 SS: Already has cert<br/>with later expiry?"}
    E -->|Yes| F["🌐 SS: Skip — keep existing"]
    E -->|No| G["🌐 SS: Store zip → Extract PEM →<br/>Load PKCS12 keystore →<br/>Upsert SSREDHATCERTDETAILS"]
    G --> H["🌐 SS: At download time:<br/>Uses cert for mTLS<br/>against cdn.redhat.com"]

    style A fill:#4CAF50,color:#fff
    style H fill:#2196F3,color:#fff
```

---

## UC-4: SUSE Token Forwarding

```mermaid
flowchart TD
    A(["🖥️ PROBE: Admin registers<br/>SUSE token"]) --> B["🖥️ PROBE: Validate token<br/>SCC reachability check"]
    B --> C{"🖥️ PROBE: Centralized download<br/>enabled?"}
    C -->|Yes| D["🖥️ PROBE: Forward token + metadata<br/>(edition, productVersion) to SS<br/>via push-to-summary"]
    C -->|No| Z(["Legacy flow"])
    D --> E["🌐 SS: Upsert SSSUSEPRODUCTKEYS<br/>on (EDITION, PRODUCT_VERSION)"]
    E --> F["🌐 SS: SuseAuthtokenTask refreshes<br/>SS-side access token"]
    F --> G["🌐 SS: At download time:<br/>Appends SUSE token<br/>to vendor URLs"]

    style A fill:#4CAF50,color:#fff
    style G fill:#2196F3,color:#fff
```

---

## UC-5.1: Enable Centralized Download

```mermaid
flowchart TD
    A(["🌐 SS: Admin calls PUT<br/>centralizedDownload enabled=true"]) --> B["🌐 SS: Validate store path<br/>writable + disk space"]
    B --> C["🌐 SS: Write sentinel file<br/>.ss-store-sentinel"]
    C --> D["🌐 SS: Admin validates Probe access<br/>POST validateProbeAccess"]
    D --> E{"🌐 SS: All Probes pass?"}
    E -->|No| F(["Fix Probe access<br/>and retry"])
    E -->|Yes| G["🌐 SS: Persist ENABLED = true"]
    G --> H["⚠️ SS Restart Required"]
    H --> I["🌐 SS: Nginx config regenerated<br/>from templates"]
    I --> J["🌐 SS: SSDownloadMissingPatchTask<br/>scheduled"]
    J --> K["🖥️ PROBE: Picks up ENABLED=true<br/>on restart / DB sync"]
    K --> L["🖥️ PROBE: Vendor download blocked<br/>Missing-patch scheduler repurposed"]

    style A fill:#4CAF50,color:#fff
    style H fill:#FF9800,color:#fff
    style L fill:#2196F3,color:#fff
```

---

## UC-5.2: Disable Centralized Download

```mermaid
flowchart TD
    A(["🌐 SS: Admin calls PUT<br/>enabled=false"]) --> B["🌐 SS: Persist ENABLED = false"]
    B --> C["⚠️ SS Restart Required"]
    C --> D["🌐 SS: Nginx config regenerated<br/>without centralized download"]
    D --> E["🖥️ PROBE: Picks up ENABLED=false<br/>on restart / DB sync"]
    E --> F["🖥️ PROBE: Resumes legacy<br/>per-Probe vendor download"]
    F --> G["🌐 SS: Common store remains on disk<br/>Admin cleans up manually"]

    style A fill:#f44336,color:#fff
    style C fill:#FF9800,color:#fff
```

---

## UC-6: Probe-Side Nginx Caching — DS/Agent Binary Serving

```mermaid
flowchart TD
    A(["🖥️ PROBE: DS/Agent requests patch<br/>GET /store/fileName"]) --> B{"🖥️ PROBE: Nginx proxy_cache<br/>check"}
    B -->|"HIT ✓"| C["🖥️ PROBE: Serve from local cache<br/>X-Cache-Status: HIT"]
    B -->|MISS| D["🖥️ PROBE: proxy_pass → 127.0.0.1:9090"]
    D --> E["🖥️ PROBE: Internal server reads file<br/>from common store via alias"]
    E --> F["🖥️ PROBE: Serve file + cache locally<br/>X-Cache-Status: MISS"]
    F --> G["🖥️ PROBE: Next request for same file<br/>→ served as HIT"]

    H["⚡ 🖥️ PROBE: Thundering Herd:<br/>50 agents request same file<br/>proxy_cache_lock = on"] --> I["🖥️ PROBE: Only 1 upstream read<br/>49 wait → served as HIT"]

    style C fill:#2196F3,color:#fff
    style F fill:#FF9800,color:#fff
    style I fill:#9C27B0,color:#fff
```

---

## UC-7.1: Patch Upload from Summary Server

```mermaid
flowchart TD
    A(["🌐 SS: Admin uploads patch<br/>POST /api/v1/ss/patch/upload"]) --> B["🌐 SS: Store binary<br/>in common store"]
    B --> C["🌐 SS: Update SSPATCHSTORELOCATION<br/>STATUS = AVAILABLE"]
    C --> D["🖥️ PROBE: Discovers on next<br/>missing-patch scheduler<br/>or at deployment time"]

    style A fill:#4CAF50,color:#fff
    style D fill:#2196F3,color:#fff
```

---

## UC-7.2: Patch Upload from Probe Server

```mermaid
flowchart TD
    A(["🖥️ PROBE: Admin uploads patch"]) --> B["🖥️ PROBE: Forward to SS via<br/>MultiPartUtilImpl"]
    B --> C["🌐 SS: Store in common store<br/>Update SSPATCHSTORELOCATION"]
    C --> D["🖥️ PROBE: Result returned"]

    style A fill:#4CAF50,color:#fff
    style D fill:#2196F3,color:#fff
```

---

## UC-8: Linux Dependency Package Forwarding

```mermaid
flowchart TD
    A(["🖥️ PROBE: Linux agent posts<br/>PackageInfoFromAgent"]) --> B["🖥️ PROBE: Process locally<br/>PACKAGEINFO, PACKAGEDOWNLOADSTATUS"]
    B --> C{"🖥️ PROBE: Centralized download<br/>enabled?"}
    C -->|Yes| D["🖥️ PROBE: Forward metadata to SS via<br/>push-to-summary queue"]
    C -->|No| Z(["Legacy: Probe downloads"])
    D --> E["🖥️ PROBE: DependencyDownloadTask<br/>SKIPPED — SS handles it"]
    E --> F["🌐 SS: Deduplicates on<br/>(pkg_name, product_id, checksum)"]
    F --> G["🌐 SS: Download binaries<br/>Generate meta XMLs<br/>in common store"]
    G --> H["🖥️ PROBE: Discovers via<br/>file-level access on<br/>next scheduler run"]

    style A fill:#4CAF50,color:#fff
    style H fill:#2196F3,color:#fff
```

---

## UC-9: Common Store Path Change

```mermaid
flowchart TD
    A(["🌐 SS: Admin changes path<br/>PUT .../changePath"]) --> B["🌐 SS: Validate new path<br/>writable + space"]
    B --> C["🌐 SS: Probe access validation<br/>sentinel file check"]
    C --> D{"🌐 SS: Migrate files?"}
    D -->|Yes| E["🌐 SS: Copy files from<br/>old → new path"]
    D -->|No| F["🌐 SS: Skip migration"]
    E --> G["🌐 SS: Update DB + websettings.conf"]
    F --> G
    G --> H["⚠️ SS Restart Required"]
    H --> I["🌐 SS: Nginx reloads with<br/>new alias path"]
    I --> J["🌐 SS: SSDownloadMissingPatchTask<br/>re-downloads missing patches"]
    J --> K["🖥️ PROBE: Picks up new path<br/>on restart / DB sync"]

    style A fill:#4CAF50,color:#fff
    style H fill:#FF9800,color:#fff
```

---

## UC-10: Probe Cache Size Reduced

```mermaid
flowchart TD
    A(["🖥️ PROBE: Admin reduces<br/>cache max_size"]) --> B["⚠️ Probe Restart required"]
    B --> C["🖥️ PROBE: Nginx detects cache<br/>exceeds new max_size"]
    C --> D["🖥️ PROBE: LRU eviction runs<br/>(non-blocking)"]
    D --> E["🖥️ PROBE: Cache shrinks to<br/>fit new limit"]

    style A fill:#FF9800,color:#fff
```

---

## UC-11: Probe Cache Size Increased

```mermaid
flowchart TD
    A(["🖥️ PROBE: Admin increases<br/>cache max_size"]) --> B["⚠️ Probe Restart required"]
    B --> C["🖥️ PROBE: More disk budget available<br/>No files deleted"]
    C --> D["🖥️ PROBE: More files can be cached<br/>before eviction triggers"]

    style A fill:#4CAF50,color:#fff
```

---

## UC-12: Cache Disabled (ON → OFF)

```mermaid
flowchart TD
    A(["🖥️ PROBE: Admin disables<br/>Probe cache"]) --> B["⚠️ Probe Restart required"]
    B --> C["🖥️ PROBE: Nginx: proxy_cache off"]
    C --> D["🖥️ PROBE: All new requests →<br/>common store via network share"]
    D --> E["🖥️ PROBE: Old cached files remain on disk<br/>Cleaned up by: inactive expiry,<br/>max_size, or manual delete"]

    style A fill:#f44336,color:#fff
```

---

## UC-13: Patch Cleanup

```mermaid
flowchart TD
    A(["🌐 SS: CleanupUtil identifies<br/>patch for cleanup"]) --> B["🌐 SS: Mark PENDING_DELETE<br/>PENDING_DELETE_AT = now()"]
    B --> C["⏳ 🌐 SS: File NOT deleted yet"]
    C --> D["🌐 SS: DeferredCleanupTask<br/>(every 15 min)"]
    D --> E{"🌐 SS: Grace period elapsed?<br/>(30 min default)"}
    E -->|No| D
    E -->|Yes| F["🌐 SS: Physically delete file<br/>STATUS → DELETED"]
    F --> G["🌐 SS: Regenerate deleted-patches.xml<br/>in common store"]
    G --> H["🌐 SS: Clean up old .failed markers<br/>older than 2× timeout"]

    I["🖥️ PROBE: Probe-side cleanup:<br/>DISABLED when centralized<br/>download is enabled"] -.-> J["🌐 SS owns entire<br/>binary lifecycle"]

    style A fill:#FF9800,color:#fff
    style F fill:#f44336,color:#fff
    style I fill:#9E9E9E,color:#fff
```

---

## UC-14: Re-Enable After Disable

```mermaid
flowchart TD
    A(["🌐 SS: Admin re-enables<br/>centralized download"]) --> B["🌐 SS: Same flow as UC-5.1"]
    B --> C["🌐 SS: SSDownloadMissingPatchTask<br/>rescans & re-downloads"]
    C --> D["🖥️ PROBE: Resumes centralized mode<br/>on restart / DB sync"]

    style A fill:#4CAF50,color:#fff
    style D fill:#2196F3,color:#fff
```

---

## UC-15: New Probe Joins After Enable

```mermaid
flowchart TD
    A(["🖥️ PROBE: New Probe installed<br/>connects to SS"]) --> B["🖥️ PROBE: DB sync picks up<br/>ENABLED=true + store path"]
    B --> C["🖥️ PROBE: Admin configures network<br/>share / mount on Probe"]
    C --> D["🌐 SS: Admin runs<br/>validateProbeAccess"]
    D --> E["⚠️ Probe restart"]
    E --> F["🖥️ PROBE: Nginx config generated<br/>with centralized download"]
    F --> G["🖥️ PROBE: Missing-patch scheduler<br/>begins scanning common store"]

    style A fill:#4CAF50,color:#fff
    style G fill:#2196F3,color:#fff
```

---

## UC-16: SS Restart During Active Download

```mermaid
flowchart TD
    A(["🌐 SS: Restarts"]) --> B["🌐 SS: DB-backed queue survives<br/>(ss-patch-download-data)"]
    B --> C["🌐 SS: Resumes downloads<br/>from vendor on startup"]
    C --> D["🖥️ PROBE: Polling unaffected<br/>continues every 5 min"]
    D --> E{"🌐 SS: Finishes<br/>before Probe timeout?"}
    E -->|Yes| F["🖥️ PROBE: Discovers file<br/>on next poll tick ✓"]
    E -->|No| G["🖥️ PROBE: → DOWNLOAD_FAILED<br/>Admin retries"]

    style A fill:#FF9800,color:#fff
    style F fill:#2196F3,color:#fff
    style G fill:#f44336,color:#fff
```

---

## UC-17: Probe Restart During Active Download

```mermaid
flowchart TD
    A(["🖥️ PROBE: Restarts"]) --> B["🖥️ PROBE: COLLECTIONPENDINGPATCHES<br/>survives (DB-backed)"]
    B --> C["🖥️ PROBE: Startup recovery scans<br/>WAITING_FOR_SS_DOWNLOAD collections"]
    C --> D{"🖥️ PROBE: All patches available<br/>in common store?"}
    D -->|Yes| E["🖥️ PROBE: Resume deployment<br/>Add to PSL → Generate zip → Deploy"]
    D -->|No| F["🖥️ PROBE: Re-send on-demand to SS<br/>Restart polling scheduler"]

    style A fill:#FF9800,color:#fff
    style E fill:#2196F3,color:#fff
```

---

## UC-18: Common Store Temporarily Inaccessible

```mermaid
flowchart TD
    A(["🖥️ PROBE: Network share / mount<br/>becomes inaccessible"]) --> B["🖥️ PROBE: File.exists() returns false<br/>or throws IOException"]
    B --> C["🖥️ PROBE: Neither file nor marker readable<br/>Treated as 'still downloading'"]
    C --> D["🖥️ PROBE: Continue polling"]
    D --> E{"Share recovers<br/>before timeout?"}
    E -->|Yes| F["🖥️ PROBE: Next poll discovers<br/>files normally ✓"]
    E -->|No| G["🖥️ PROBE: Collection → DOWNLOAD_FAILED<br/>Admin retries after fixing share"]

    H["🖥️ PROBE: If proxy_cache ON:<br/>Previously cached files<br/>still served as HIT"] -.-> I["🖥️ PROBE: DS/Agent downloads<br/>unaffected for cached patches"]

    style A fill:#f44336,color:#fff
    style F fill:#2196F3,color:#fff
    style G fill:#f44336,color:#fff
    style H fill:#9C27B0,color:#fff
```

---

## Master Overview — All Use Cases

```mermaid
flowchart LR
    subgraph Setup["⚙️ Setup & Config (🌐 SS)"]
        UC5_1[UC-5.1: Enable]
        UC5_2[UC-5.2: Disable]
        UC9[UC-9: Change Store Path]
        UC14[UC-14: Re-Enable]
        UC15[UC-15: New Probe Joins]
    end

    subgraph Credentials["🔑 Credential Forwarding (🖥️ PROBE → 🌐 SS)"]
        UC3[UC-3: RedHat Cert]
        UC4[UC-4: SUSE Token]
    end

    subgraph ProbeDownload["📥 Probe-Side Download (🖥️ PROBE polls common store)"]
        UC1_1[UC-1.1: Deployment Trigger]
        UC1_2[UC-1.2: Agent Request]
        UC1_3[UC-1.3: Missing Patch Scheduler]
        UC1_4[UC-1.4: Redownload]
        UC1_5[UC-1.5: SS Failure]
        UC1_6[UC-1.6: Existing Failure Marker]
        UC1_7[UC-1.7: Already Available]
        UC1_8[UC-1.8: Checksum Invalid]
        UC1_9[UC-1.9: Timeout]
    end

    subgraph Serving["🚀 Serving & Caching (🖥️ PROBE Nginx)"]
        UC6[UC-6: Nginx Cache Serving]
        UC10[UC-10: Cache Size Reduced]
        UC11[UC-11: Cache Size Increased]
        UC12[UC-12: Cache Disabled]
    end

    subgraph Upload["📤 Upload & Forwarding (🖥️ PROBE → 🌐 SS)"]
        UC7_1[UC-7.1: Upload from SS]
        UC7_2[UC-7.2: Upload from Probe]
        UC8[UC-8: Linux Dep Forwarding]
    end

    subgraph Ops["🔧 Operations"]
        UC2["UC-2: Status Visibility (🖥️ PROBE)"]
        UC13["UC-13: Patch Cleanup (🌐 SS)"]
        UC16["UC-16: SS Restart (🌐 SS)"]
        UC17["UC-17: Probe Restart (🖥️ PROBE)"]
        UC18["UC-18: Store Inaccessible (🖥️ PROBE)"]
    end

    Setup --> ProbeDownload
    Credentials --> ProbeDownload
    ProbeDownload --> Serving
    Upload --> ProbeDownload
```
