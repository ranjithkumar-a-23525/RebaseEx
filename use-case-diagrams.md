# Centralized Patch Download — Use Case Flow Diagrams

---

## UC-1.1: Deployment-Triggered Download — Patch Not in Common Store

```mermaid
flowchart TD
    A([Deployment Initiated]) --> B{Check Common Store<br/>for all patches}
    B -->|All found| Z1([Skip to UC-1.7:<br/>Direct PSL add])
    B -->|Some missing| C[Send On-Demand request<br/>to SS via push-to-summary]
    C --> D[Update status →<br/>WAITING_FOR_SS_DOWNLOAD 502]
    D --> E[Persist to<br/>COLLECTIONPENDINGPATCHES]
    E --> F[Start 5-min<br/>Polling Scheduler]
    F --> G{Poll Common Store}
    G -->|File exists| H[Mark download success<br/>Add to PSL<br/>Remove from pending]
    G -->|.failed marker exists| I[Read failure reason<br/>Mark DOWNLOAD_FAILED]
    G -->|Neither exists| J{Timeout reached?<br/>30 min / 6 ticks}
    J -->|No| G
    J -->|Yes| K[Mark DOWNLOAD_FAILED<br/>Reason: SS timed out]
    H --> L{All patches ready?}
    L -->|No| G
    L -->|Yes| M[Generate patch-products.zip<br/>Deploy to DS/Agents]
    M --> N([Cancel Polling ✓])
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
    A([Agent requests patch<br/>not in PSL]) --> B[Forward request to SS<br/>via push-to-summary]
    B --> C[Add to collection-to-patch<br/>status map]
    C --> D[Polling Scheduler monitors<br/>common store]
    D --> E{Check Common Store}
    E -->|File exists| F[Mark success → Add to PSL<br/>Fire post-download callbacks]
    E -->|.failed marker| G[Mark failure<br/>Post-download activity]
    E -->|Timeout| H[Mark failed<br/>Post-download activity]
    E -->|Neither| D

    I[Missing Patch Scheduler<br/>every 30 min] -.->|May discover<br/>file first| F

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
    A([Scheduler fires<br/>every 30 min]) --> B[Query SCOPEAFFPATCHSTATUS<br/>LEFT JOIN PATCHSTORELOCATION]
    B --> C[Find patches needed<br/>but not in PSL]
    C --> D{For each missing patch}
    D --> E{File exists in<br/>common store?}
    E -->|Yes & size > 0| F[Add to PSL directly<br/>Fire post-download callbacks<br/>Log: found in common store]
    E -->|No| G[Skip — SS will handle<br/>via SSDownloadMissingPatchTask<br/>or on-demand]
    F --> D
    G --> D
    D -->|All processed| H([Done ✓])

    style A fill:#FF9800,color:#fff
    style H fill:#2196F3,color:#fff
```

> **Note:** This scheduler is a passive scanner — it never sends download requests to SS.

---

## UC-1.4: Redownload Triggered for Patch

```mermaid
flowchart TD
    A([Admin triggers<br/>Redownload]) --> B[Forward to SS via<br/>push-to-summary<br/>highest priority]
    B --> C[Add to download listener]
    C --> D[Polling Scheduler monitors]
    D --> E{Check Common Store}
    E -->|File exists| F([Success ✓])
    E -->|.failed marker| G([Failure ✗])
    E -->|Timeout| H([Failure — timed out ✗])
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
    A([SS download from<br/>vendor fails]) --> B{Retry exhausted?<br/>3 retries for connection<br/>Immediate for 404/checksum}
    B -->|No| C[Retry download]
    C --> B
    B -->|Yes| D[Update SSPATCHSTORELOCATION<br/>STATUS = FAILED]
    D --> E["Write failure marker:<br/>.patch-status/{patchId}_{langId}.failed"]
    E --> F[No push event to Probe]
    F --> G[Probe polling detects<br/>.failed marker ≤5 min]
    G --> H[Probe reads failure reason]
    H --> I[Mark patch as failed<br/>Collection → DOWNLOAD_FAILED]

    style A fill:#f44336,color:#fff
    style I fill:#f44336,color:#fff
```

---

## UC-1.6: Failure Marker Already Exists

```mermaid
flowchart TD
    A([Probe encounters patch<br/>with .failed marker]) --> B{Already in<br/>collectionToPatchStatusMap?}
    B -->|No — Deployment/Agent| C[Send on-demand request to SS]
    C --> D["SS detects FAILED status →<br/>Clear .failed marker →<br/>Reset to QUEUED →<br/>Re-queue highest priority"]
    D --> E[Probe polls normally]
    B -->|Yes| F[Skip duplicate request<br/>Continue polling]
    
    G([Redownload trigger]) --> H[Delete failure marker<br/>unconditionally]
    H --> I[Send redownload to SS<br/>Normal flow resumes]

    style A fill:#FF9800,color:#fff
    style G fill:#FF9800,color:#fff
```

---

## UC-1.7: Patch Already Available — Checksum Valid

```mermaid
flowchart TD
    A([Patch found in<br/>Common Store]) --> B{Validate checksum}
    B -->|Regular patch| C[SHA256 from BINARYCHECKSUMVALUES<br/>MD5 fallback for >4GB]
    B -->|Linux dep pkg| D[Checksum from PACKAGEINFO<br/>or meta XML]
    C --> E{Valid?}
    D --> E
    E -->|Yes ✓| F[Skip download entirely<br/>Add to PSL<br/>Post-download activity]
    E -->|No ✗| G([Go to UC-1.8:<br/>Checksum Invalid])

    style A fill:#4CAF50,color:#fff
    style F fill:#2196F3,color:#fff
    style G fill:#f44336,color:#fff
```

---

## UC-1.8: Patch Available but Checksum Invalid

```mermaid
flowchart TD
    A([Checksum mismatch<br/>detected]) --> B[Probe sends corruption<br/>report to SS]
    B --> C[SS deletes corrupted file<br/>+ failure markers]
    C --> D[SS re-queues download<br/>from vendor — highest priority]
    D --> E[Probe polling resumes<br/>Normal flow]

    style A fill:#f44336,color:#fff
    style E fill:#2196F3,color:#fff
```

---

## UC-1.9: Download Timeout

```mermaid
flowchart TD
    A([Polling reaches maxPolls<br/>6 ticks × 5 min = 30 min]) --> B[Mark remaining patches<br/>as download failure]
    B --> C["Collection → DOWNLOAD_FAILED<br/>Reason: SS timed out after {N} min"]
    C --> D[Cancel Polling Scheduler]
    D --> E[No vendor fallback<br/>Admin must retry via<br/>RETRY_ON_DEMAND API]

    style A fill:#f44336,color:#fff
    style E fill:#FF9800,color:#fff
```

---

## UC-2: Download Status Visibility on Probe

```mermaid
flowchart TD
    A([Admin opens<br/>Centralized Download View]) --> B[Query local Probe DB]
    B --> C[COLLECTIONPENDINGPATCHES<br/>+ collection status tables]
    C --> D{Display}
    D --> E["🟡 WAITING_FOR_SS_DOWNLOAD<br/>— waiting since, patches required vs available"]
    D --> F["🔴 DOWNLOAD_FAILED<br/>— failure reasons"]
    D --> G["🟢 Completed<br/>— deployment status"]
    E --> H[Actions: RETRY_ON_DEMAND<br/>CANCEL_DEPLOYMENT]

    style A fill:#4CAF50,color:#fff
```

> No real-time query to SS — all state derived from local Probe DB via polling.

---

## UC-3: RedHat Certificate Forwarding

```mermaid
flowchart TD
    A([Agent uploads RedHat cert<br/>to Probe]) --> B[Probe validates cert<br/>CDN reachability test]
    B --> C{Centralized download<br/>enabled?}
    C -->|Yes| D["Forward cert zip + metadata<br/>(edition, expiry) to SS<br/>via MultiPartUtilImpl"]
    C -->|No| Z([Legacy flow])
    D --> E{SS already has cert<br/>with later expiry?}
    E -->|Yes| F[Skip — keep existing]
    E -->|No| G["Store zip → Extract PEM →<br/>Load PKCS12 keystore →<br/>Upsert SSREDHATCERTDETAILS"]
    G --> H["At download time:<br/>SS uses cert for mTLS<br/>against cdn.redhat.com"]

    style A fill:#4CAF50,color:#fff
    style H fill:#2196F3,color:#fff
```

---

## UC-4: SUSE Token Forwarding

```mermaid
flowchart TD
    A([Probe admin registers<br/>SUSE token]) --> B[Probe validates token<br/>SCC reachability check]
    B --> C{Centralized download<br/>enabled?}
    C -->|Yes| D["Forward token + metadata<br/>(edition, productVersion) to SS<br/>via push-to-summary"]
    C -->|No| Z([Legacy flow])
    D --> E["SS upserts SSSUSEPRODUCTKEYS<br/>on (EDITION, PRODUCT_VERSION)"]
    E --> F[SuseAuthtokenTask refreshes<br/>SS-side access token]
    F --> G["At download time:<br/>SS appends SUSE token<br/>to vendor URLs"]

    style A fill:#4CAF50,color:#fff
    style G fill:#2196F3,color:#fff
```

---

## UC-5.1: Enable Centralized Download

```mermaid
flowchart TD
    A([Admin calls PUT<br/>centralizedDownload enabled=true]) --> B[Validate store path<br/>writable + disk space]
    B --> C[Write sentinel file<br/>.ss-store-sentinel]
    C --> D[Admin validates Probe access<br/>POST validateProbeAccess]
    D --> E{All Probes pass?}
    E -->|No| F([Fix Probe access<br/>and retry])
    E -->|Yes| G[Persist ENABLED = true]
    G --> H["⚠️ SS Restart Required"]
    H --> I[Nginx config regenerated<br/>from templates]
    I --> J[SSDownloadMissingPatchTask<br/>scheduled]
    J --> K[Probes pick up ENABLED=true<br/>on restart / DB sync]
    K --> L[Probe vendor download blocked<br/>Missing-patch scheduler repurposed]

    style A fill:#4CAF50,color:#fff
    style H fill:#FF9800,color:#fff
    style L fill:#2196F3,color:#fff
```

---

## UC-5.2: Disable Centralized Download

```mermaid
flowchart TD
    A([Admin calls PUT<br/>enabled=false]) --> B[Persist ENABLED = false]
    B --> C["⚠️ SS Restart Required"]
    C --> D[Nginx config regenerated<br/>without centralized download]
    D --> E[Probes pick up ENABLED=false<br/>on restart / DB sync]
    E --> F[Probes resume legacy<br/>per-Probe vendor download]
    F --> G[Common store remains on disk<br/>Admin cleans up manually]

    style A fill:#f44336,color:#fff
    style C fill:#FF9800,color:#fff
```

---

## UC-6: Probe-Side Nginx Caching — DS/Agent Binary Serving

```mermaid
flowchart TD
    A([DS/Agent requests patch<br/>GET /store/fileName]) --> B{Nginx proxy_cache<br/>check}
    B -->|HIT ✓| C["Serve from local cache<br/>X-Cache-Status: HIT"]
    B -->|MISS| D["proxy_pass → 127.0.0.1:9090"]
    D --> E[Internal server reads file<br/>from common store via alias]
    E --> F["Serve file + cache locally<br/>X-Cache-Status: MISS"]
    F --> G[Next request for same file<br/>→ served as HIT]

    H["⚡ Thundering Herd:<br/>50 agents request same file<br/>proxy_cache_lock = on"] --> I["Only 1 upstream read<br/>49 wait → served as HIT"]

    style C fill:#2196F3,color:#fff
    style F fill:#FF9800,color:#fff
    style I fill:#9C27B0,color:#fff
```

---

## UC-7.1: Patch Upload from Summary Server

```mermaid
flowchart TD
    A([Admin uploads patch<br/>POST /api/v1/ss/patch/upload]) --> B[SS stores binary<br/>in common store]
    B --> C[Update SSPATCHSTORELOCATION<br/>STATUS = AVAILABLE]
    C --> D[Probes discover on next<br/>missing-patch scheduler<br/>or at deployment time]

    style A fill:#4CAF50,color:#fff
    style D fill:#2196F3,color:#fff
```

---

## UC-7.2: Patch Upload from Probe Server

```mermaid
flowchart TD
    A([Probe admin uploads patch]) --> B[Probe forwards to SS via<br/>MultiPartUtilImpl]
    B --> C[SS stores in common store<br/>Updates SSPATCHSTORELOCATION]
    C --> D[Result returned to Probe]

    style A fill:#4CAF50,color:#fff
    style D fill:#2196F3,color:#fff
```

---

## UC-8: Linux Dependency Package Forwarding

```mermaid
flowchart TD
    A([Linux agent posts<br/>PackageInfoFromAgent]) --> B[Probe processes locally<br/>PACKAGEINFO, PACKAGEDOWNLOADSTATUS]
    B --> C{Centralized download<br/>enabled?}
    C -->|Yes| D["Forward metadata to SS via<br/>push-to-summary queue"]
    C -->|No| Z([Legacy: Probe downloads])
    D --> E[Probe's DependencyDownloadTask<br/>SKIPPED — SS handles it]
    E --> F["SS deduplicates on<br/>(pkg_name, product_id, checksum)"]
    F --> G[SS downloads binaries<br/>Generates meta XMLs<br/>in common store]
    G --> H[Probes discover via<br/>file-level access on<br/>next scheduler run]

    style A fill:#4CAF50,color:#fff
    style H fill:#2196F3,color:#fff
```

---

## UC-9: Common Store Path Change

```mermaid
flowchart TD
    A([Admin changes path<br/>PUT .../changePath]) --> B[Validate new path<br/>writable + space]
    B --> C[Probe access validation<br/>sentinel file check]
    C --> D{Migrate files?}
    D -->|Yes| E[Copy files from<br/>old → new path]
    D -->|No| F[Skip migration]
    E --> G[Update DB + websettings.conf]
    F --> G
    G --> H["⚠️ SS Restart Required"]
    H --> I[Nginx reloads with<br/>new alias path]
    I --> J[SSDownloadMissingPatchTask<br/>re-downloads missing patches]
    J --> K[Probes pick up new path<br/>on restart / DB sync]

    style A fill:#4CAF50,color:#fff
    style H fill:#FF9800,color:#fff
```

---

## UC-10: Probe Cache Size Reduced

```mermaid
flowchart TD
    A([Admin reduces<br/>cache max_size]) --> B["⚠️ Restart required"]
    B --> C[Nginx detects cache<br/>exceeds new max_size]
    C --> D["LRU eviction runs<br/>(non-blocking)"]
    D --> E[Cache shrinks to<br/>fit new limit]

    style A fill:#FF9800,color:#fff
```

---

## UC-11: Probe Cache Size Increased

```mermaid
flowchart TD
    A([Admin increases<br/>cache max_size]) --> B["⚠️ Restart required"]
    B --> C[More disk budget available<br/>No files deleted]
    C --> D[More files can be cached<br/>before eviction triggers]

    style A fill:#4CAF50,color:#fff
```

---

## UC-12: Cache Disabled (ON → OFF)

```mermaid
flowchart TD
    A([Admin disables<br/>Probe cache]) --> B["⚠️ Restart required"]
    B --> C["Nginx: proxy_cache off"]
    C --> D[All new requests →<br/>common store via network share]
    D --> E["Old cached files remain on disk<br/>Cleaned up by: inactive expiry,<br/>max_size, or manual delete"]

    style A fill:#f44336,color:#fff
```

---

## UC-13: Patch Cleanup

```mermaid
flowchart TD
    A([CleanupUtil identifies<br/>patch for cleanup]) --> B["Mark PENDING_DELETE<br/>PENDING_DELETE_AT = now()"]
    B --> C["⏳ File NOT deleted yet"]
    C --> D["DeferredCleanupTask<br/>(every 15 min)"]
    D --> E{"Grace period elapsed?<br/>(30 min default)"}
    E -->|No| D
    E -->|Yes| F[Physically delete file<br/>STATUS → DELETED]
    F --> G[Regenerate deleted-patches.xml<br/>in common store]
    G --> H[Clean up old .failed markers<br/>older than 2× timeout]

    I["Probe-side cleanup:<br/>DISABLED when centralized<br/>download is enabled"] -.-> J["SS owns entire<br/>binary lifecycle"]

    style A fill:#FF9800,color:#fff
    style F fill:#f44336,color:#fff
    style I fill:#9E9E9E,color:#fff
```

---

## UC-15: New Probe Joins After Enable

```mermaid
flowchart TD
    A([New Probe installed<br/>connects to SS]) --> B[DB sync picks up<br/>ENABLED=true + store path]
    B --> C[Admin configures network<br/>share / mount on Probe]
    C --> D[Admin runs<br/>validateProbeAccess from SS]
    D --> E["⚠️ Probe restart"]
    E --> F[Nginx config generated<br/>with centralized download]
    F --> G[Missing-patch scheduler<br/>begins scanning common store]

    style A fill:#4CAF50,color:#fff
    style G fill:#2196F3,color:#fff
```

---

## UC-16: SS Restart During Active Download

```mermaid
flowchart TD
    A([SS restarts]) --> B["DB-backed queue survives<br/>(ss-patch-download-data)"]
    B --> C[SS resumes downloads<br/>from vendor on startup]
    C --> D[Probe polling unaffected<br/>continues every 5 min]
    D --> E{SS finishes<br/>before Probe timeout?}
    E -->|Yes| F[Probe discovers file<br/>on next poll tick ✓]
    E -->|No| G[Probe → DOWNLOAD_FAILED<br/>Admin retries]

    style A fill:#FF9800,color:#fff
    style F fill:#2196F3,color:#fff
    style G fill:#f44336,color:#fff
```

---

## UC-17: Probe Restart During Active Download

```mermaid
flowchart TD
    A([Probe restarts]) --> B["COLLECTIONPENDINGPATCHES<br/>survives (DB-backed)"]
    B --> C[Startup recovery scans<br/>WAITING_FOR_SS_DOWNLOAD collections]
    C --> D{All patches available<br/>in common store?}
    D -->|Yes| E[Resume deployment<br/>Add to PSL → Generate zip → Deploy]
    D -->|No| F[Re-send on-demand to SS<br/>Restart polling scheduler]

    style A fill:#FF9800,color:#fff
    style E fill:#2196F3,color:#fff
```

---

## UC-18: Common Store Temporarily Inaccessible

```mermaid
flowchart TD
    A(["Network share / mount<br/>becomes inaccessible"]) --> B["File.exists() returns false<br/>or throws IOException"]
    B --> C[Neither file nor marker readable<br/>Treated as 'still downloading']
    C --> D[Continue polling]
    D --> E{Share recovers<br/>before timeout?}
    E -->|Yes| F[Next poll discovers<br/>files normally ✓]
    E -->|No| G[Collection → DOWNLOAD_FAILED<br/>Admin retries after fixing share]

    H["If proxy_cache ON:<br/>Previously cached files<br/>still served as HIT"] -.-> I["DS/Agent downloads<br/>unaffected for cached patches"]

    style A fill:#f44336,color:#fff
    style F fill:#2196F3,color:#fff
    style G fill:#f44336,color:#fff
    style H fill:#9C27B0,color:#fff
```

---

## Master Overview — All Use Cases

```mermaid
flowchart LR
    subgraph Setup["⚙️ Setup & Config"]
        UC5_1[UC-5.1: Enable]
        UC5_2[UC-5.2: Disable]
        UC9[UC-9: Change Store Path]
        UC14[UC-14: Re-Enable]
        UC15[UC-15: New Probe Joins]
    end

    subgraph Credentials["🔑 Credential Forwarding"]
        UC3[UC-3: RedHat Cert]
        UC4[UC-4: SUSE Token]
    end

    subgraph ProbeDownload["📥 Probe-Side Download (UC-1)"]
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

    subgraph Serving["🚀 Serving & Caching"]
        UC6[UC-6: Nginx Cache Serving]
        UC10[UC-10: Cache Size Reduced]
        UC11[UC-11: Cache Size Increased]
        UC12[UC-12: Cache Disabled]
    end

    subgraph Upload["📤 Upload & Forwarding"]
        UC7_1[UC-7.1: Upload from SS]
        UC7_2[UC-7.2: Upload from Probe]
        UC8[UC-8: Linux Dep Forwarding]
    end

    subgraph Ops["🔧 Operations"]
        UC2[UC-2: Status Visibility]
        UC13[UC-13: Patch Cleanup]
        UC16[UC-16: SS Restart]
        UC17[UC-17: Probe Restart]
        UC18[UC-18: Store Inaccessible]
    end

    Setup --> ProbeDownload
    Credentials --> ProbeDownload
    ProbeDownload --> Serving
    Upload --> ProbeDownload
```

