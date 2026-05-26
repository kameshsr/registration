# MOSIP Identity Draft Lifecycle — End-to-End Flow

> Covers two projects working together:
> - **registration-processor** (`registrationNew1`) — the caller that orchestrates the draft workflow
> - **id-repository** (`id-repositoryNew`) — the server that owns the draft data and exposes the REST API

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [API Endpoints — IdRepoDraftController](#api-endpoints)
4. [Registration Processor Side — IdrepoDraftService](#registration-processor-side)
5. [ID Repository Side — IdRepoDraftServiceImpl](#id-repository-side)
   - [hasDraft](#1-hasdraft)
   - [createDraft](#2-createdraft)
   - [getDraft](#3-getdraft)
   - [updateDraft](#4-updatedraft)
   - [publishDraft](#5-publishdraft)
   - [discardDraft](#6-discarddraft)
   - [extractBiometrics](#7-extractbiometrics)
6. [Stage-Level Flows](#stage-level-flows)
   - [UIN Generator Stage](#uin-generator-stage)
   - [Biometric Extraction Stage](#biometric-extraction-stage)
   - [Finalization Stage](#finalization-stage)
7. [Internal Service Dependencies](#internal-service-dependencies)
8. [Error Handling & Exception Types](#error-handling)
9. [Interceptors & Filters — Encrypt/Decrypt Layer](#interceptors--filters)
10. [Sequence Diagram](#sequence-diagram)

---

## Overview

MOSIP uses a **draft pattern** to build an identity record incrementally across multiple processing stages before permanently committing it to the ID Repository. The draft acts as a staging area: demographic data, documents, and biometrics are added piece by piece as the registration packet moves through the pipeline. Only when all stages succeed is the draft **published**, making the identity live.

The key API at the centre of this is:

```
HEAD /idrepository/v1/identity/draft/{registrationId}
```

This is `IDREPOHASDRAFT` — a lightweight probe that returns HTTP 200 (draft exists) or HTTP 204 (no draft), and is used as a guard before every draft operation.

---

## Complete End-to-End Flow (with Interceptors & Filters)

This is the full pipeline from registration-processor stages through id-repository, credential pipeline, and IDA — with all encrypt/decrypt layers annotated inline.

```
BiometricExtractionStage / FinalizationStage / UinGeneratorStage
          │
          ▼
   HEAD /idrepository/v1/identity/draft/{id}          ← IDREPOHASDRAFT
          │
          │   ┌─ id-repository HTTP boundary ─────────────────────────────┐
          │   │  IdRepoFilter (BaseIdRepoFilter)                          │
          │   │  → logs request URL + timing only; NO crypto here         │
          │   └───────────────────────────────────────────────────────────┘
          │
    200? ─┼─ 204?
          │         │
          │         └─► FinalizationStage        → FAIL (no draft)
          │         └─► BiometricExtractionStage → FAIL (no draft)
          │         └─► UinGeneratorStage
          │               └─► POST /draft/create/{id}     ← IDREPOCREATEDRAFT
          │               │       │
          │               │       ▼ [id-repository: IdRepoDraftServiceImpl.createDraft()]
          │               │   uinDraftRepo.save(newDraft)
          │               │       │
          │               │       │  ◄── IdRepoEntityInterceptor.onSave()
          │               │       │       encrypt uinData  (AES via securityManager.encrypt, uinDataRefId)
          │               │       │       encrypt uin      (AES+Salt via securityManager.encryptWithSalt)
          │               │       ▼
          │               │   [uin_draft table — stored encrypted at rest]
          │               │
          │               └─► PATCH /draft/update/{id}    ← IDREPOUPDATEDRAFT
          │                       │
          │                       ▼ [updateDraft()]
          │                   uinDraftRepo.findByRegId(regId)
          │                       │
          │                       │  ◄── IdRepoEntityInterceptor.onLoad()
          │                       │       decrypt uinData (AES)
          │                       │       verify SHA-256(decrypted) == uinDataHash
          │                       │       → throws IDENTITY_HASH_MISMATCH if tampered
          │                       ▼
          │                   [UinDraft entity — plain text in service layer]
          │                   JSONCompare diff → merge demographics
          │                   S3 upload new docs
          │                   uinDraftRepo.save(draft)
          │                       │
          │                       │  ◄── IdRepoEntityInterceptor.onFlushDirty()
          │                       │       re-encrypt uinData + uin before DB flush
          │                       ▼
          │                   [uin_draft table — re-encrypted on UPDATE]
          │
          └─► BiometricExtractionStage
          │     └─► PUT /draft/extractbiometrics/{id}
          │             │
          │             ▼ [extractBiometrics() — @Transactional(NOT_SUPPORTED)]
          │         uinDraftRepo.findByRegId(regId)
          │             │  ◄── IdRepoEntityInterceptor.onLoad() (decrypt, hash-verify)
          │             ▼
          │         S3.getBiometricObject() → raw CBEFF bytes
          │         proxyService.getBiometricsForRequestedFormats()
          │         S3.store(extracted result)
          │
          └─► FinalizationStage
                └─► GET /draft/publish/{id}               ← IDREPOPUBLISHDRAFT
                      │
                      ▼ [publishDraft()]
                  uinDraftRepo.findByRegId(regId)
                      │  ◄── IdRepoEntityInterceptor.onLoad() (decrypt, hash-verify)
                      ▼
                  securityManager.decryptWithSalt(encryptedUin, salt)
                  → plain UIN string for addIdentity/updateIdentity

                  ┌── New Identity ────────────────────────────────────────┐
                  │  vidDraftHelper.generateDraftVid(uin)                  │
                  │  super.addIdentity()                                   │
                  │  uinRepo.save(newUin)                                  │
                  │    │  ◄── IdRepoEntityInterceptor.onSave()             │
                  │    │       encrypt uinData + uin before INSERT         │
                  │    ▼                                                   │
                  │  [uin table — stored encrypted]                        │
                  │  vidDraftHelper.activateDraftVid(draftVid)             │
                  └────────────────────────────────────────────────────────┘
                  ┌── Existing Identity ───────────────────────────────────┐
                  │  super.updateIdentity()                                │
                  │  uinRepo.save(updatedUin)                              │
                  │    │  ◄── IdRepoEntityInterceptor.onFlushDirty()       │
                  │    │       re-encrypt uinData + uin on UPDATE          │
                  │    ▼                                                   │
                  │  [uin table — re-encrypted]                            │
                  └────────────────────────────────────────────────────────┘

                  notify() → WebSub
                  └─► publish IDENTITY_CREATED / IDENTITY_UPDATED topic

                  issueCredential()
                  └─► writes row → credential_request_status (status = NEW)
                        │
                        │  ◄── CredentialTransactionInterceptor.onSave()
                        │       Base64URLSafe(requestBytes) → cryptoUtil.encryptData()
                        │       stores encrypted `request` field in DB
                        ▼
                  [credential_request_status — request column encrypted at rest]

                        │
                        ▼  @Scheduled job (identity-service CredentialStatusManager)
              reads credential_request_status WHERE status = NEW
                        │
                        │  ◄── CredentialTransactionInterceptor.onLoad()
                        │       cryptoUtil.decryptData() → Base64URLSafe decode
                        │       (falls back to raw if decrypt fails — backward compat)
                        ▼
              POST /credentialrequest/{partnerId}   → Credential Request Generator

                        │
                        ▼ [credential-request-generator]
              saves credential_transaction (status = NEW)
                        │  ◄── CredentialTransactionInterceptor.onSave() (same pattern)
                        ▼
              [credential_transaction — request column encrypted]

                        │
                        ▼  @Scheduled Spring Batch — CredentialProcessJob
              POST /credentialservice/issue         → Credential Service

                        │
                        ▼ [credential-service]
              POST /datashare/create                → Data Share Service
              └─► returns dataShareUri

              publish CREDENTIAL_ISSUED event       → WebSub Hub
              topic: {partnerId}/CREDENTIAL_ISSUED

  ══════════════════════════════════════════════════════════════════════
  IDA (id-authentication) — SUBSCRIBED to {partnerId}/CREDENTIAL_ISSUED
  ══════════════════════════════════════════════════════════════════════

  IdChangeEventHandlerServiceImpl.handleCredentialIssued()
  └─► credentialStoreService.storeEventModel()
        └─► saves → credential_event_store (status = NEW)

                        │
                        ▼  @Scheduled Spring Batch — CredentialStoreJob
              CredentialStoreTasklet.execute()
              credentialEventRepo.findNewOrFailedEvents(pageSize=100)
              ForkJoinPool (threads: ida.batch.credential.store.thread.count)

                        │ per event:
                        ▼ CredentialStoreServiceImpl.doProcessCredentialStoreEvent()
              parse EventModel
              extract dataShareUri + demoEncryptedRandomKey + bioEncryptedRandomKey
              saveSalt() → ida_uin_hash_salt table
              securityManager.reEncryptAndStoreRandomKey() → re-encrypt under IDA KMS
              dataShareManager.downloadObject(dataShareUri) → fetch credential JSON

              createIdentityEntity()
              ├─► splitDemoBioData() — separate demographic vs biometric keys
              ├─► identityCacheRepo.findById(idHash) — INSERT or UPDATE
              │     ├─► demographicData (stored in identity_cache)
              │     └─► biometricData  (stored in identity_cache)
              └─► storeIdentityEntity()
                    └─► identityCacheRepo.save() → DB

              updateEventProcessingStatus()
              ├─► credential_event_store → status = STORED
              └─► CredentialStoreStatusEventPublisher.publishEvent()
                        │
                        ▼
              publish CREDENTIAL_STATUS_UPDATE event → WebSub Hub
              topic: ida-topic-credential-status-update
              payload: { requestId, status: "STORED", timestamp }

  ══════════════════════════════════════════════════════════════════════
  IDA Auth Request — HTTP Filter Chain (how auth is decrypted)
  ══════════════════════════════════════════════════════════════════════

  Partner sends: POST /auth/{mispLK}/{partnerId}/{apiKey}
  Headers: signature (JWS of full body)
  Body:    { id, version, requestSessionKey(RSA-encrypted), requestHMAC, thumbprint,
             request(AES-encrypted+base64), biometrics[](AES-GCM per BDB) }
                        │
  [BaseIDAFilter]       ▼
              validate `id` field + `version` format
                        │
  [BaseAuthFilter]      ▼
              authenticateRequest()
              └─► keyManager.verifySignature(signature header)
                  validates RSA cert trust chain of partner

  [IdAuthFilter]        ▼
              decipherRequest()
              ├─► decode(requestSessionKey) → encryptedSessionKey bytes
              ├─► keyManager.kernelDecryptAndDecode(
              │     thumbprint, encryptedSessionKey, encryptedHMAC, IDA_refId)
              │   → RSA-decrypts session key using IDA private key
              │   → AES-decrypts HMAC, validates payload integrity
              │
              ├─► keyManager.requestData()
              │   → AES-decrypts `request` field using session key
              │   → verifies HMAC of decrypted payload
              │
              └─► decipherBioData() [per biometric segment]
                    decode JWS data → verify digitalId signature
                    salt = Base64(XOR(timestamp, txnId)[-2 bytes])
                    aad  = Base64(XOR(timestamp, txnId)[-3 bytes])
                    keyManager.kernelDecrypt(thumbprint, encBioSessionKey,
                      encBioValue, BIO_refId, aad, salt)
                    → AES-GCM decrypts each Biometric Data Block (BDB)
                        │
  [IdAuthFilter]        ▼
              validateDecipheredRequest()
              └─► partnerService.validateAndGetPolicy(partnerId)
                  checkMispPolicyAllowed()
                  checkAllowedAuthTypeBasedOnPolicy()  (demo/bio/otp/pin)
                  addMetadata(partnerId, partnerPolicy) to request
                        │
                        ▼
              Spring AuthController (receives plain-text decrypted request)
              └─► fetch identity from identity_cache by idHash
              └─► run auth match (demographic / biometric / OTP)
                        │
  [BaseIDAFilter]       ▼
              consumeResponse()
              └─► keyManager.signResponse(responseAsString)
                  → signs response with IDA private key (JWS)
                  → sets `response-signature` HTTP header
                        │
                        ▼
  Partner receives: JSON body (plain) + response-signature header (JWS)
```

---

### Key WebSub Topics Summary

| Direction | Topic | Publisher | Subscriber | Purpose |
|---|---|---|---|---|
| id-repo → WebSub | `{partnerId}/IDENTITY_CREATED` | id-repository | (informational) | New identity published |
| id-repo → WebSub | `{partnerId}/IDENTITY_UPDATED` | id-repository | (informational) | Existing identity updated |
| cred-service → WebSub | `{partnerId}/CREDENTIAL_ISSUED` | credential-service | **IDA** | Triggers credential store in IDA |
| IDA → WebSub | `ida-topic-credential-status-update` | **IDA** | **credential-request-generator** | IDA acknowledges STORED / FAILED |
| id-repo → WebSub | `{partnerId}/REMOVE_ID` | id-repository | IDA | Tells IDA to remove identity from cache |
| IDA → WebSub | `REMOVE_ID_STATUS` | IDA | id-repository | IDA confirms identity removed |
| id-repo → WebSub | `{partnerId}/DEACTIVATE_ID` | id-repository | IDA | Update expiry/transaction limit in cache |
| id-repo → WebSub | `{partnerId}/ACTIVATE_ID` | id-repository | IDA | Restore identity metadata in cache |
| id-repo → WebSub | `VID_CRED_STATUS_UPDATE` | credential-service | id-repository | VID lifecycle events (subscribed on startup) |

---

### Encrypt/Decrypt Layer Summary

| Where | Class | Type | What is encrypted |
|---|---|---|---|
| id-repository HTTP in | `IdRepoFilter` | Servlet Filter | **Nothing** — logging/timing only |
| id-repository DB save | `IdRepoEntityInterceptor` | Hibernate Interceptor | `uinData` (AES), `uin` (AES+Salt), `handle` (AES+Salt) |
| id-repository DB load | `IdRepoEntityInterceptor` | Hibernate Interceptor | Decrypts `uinData`; verifies SHA-256 hash |
| credential-request-gen DB | `CredentialTransactionInterceptor` | Hibernate Interceptor | `request` column in `credential_request_status` / `credential_transaction` |
| vid-service DB | `IdRepoVidEntityInterceptor` | Hibernate Interceptor | `vid` + `vidData` columns |
| IDA HTTP in (request) | `BaseIDAFilter→IdAuthFilter` | Servlet Filter | `request` field (AES session key), biometric BDBs (AES-GCM) |
| IDA HTTP in (signature) | `BaseAuthFilter` | Servlet Filter | Verifies JWS signature header (RSA) |
| IDA HTTP out (response) | `BaseIDAFilter` | Servlet Filter | Signs response with JWS (RSA) |
| IDA KMS re-encrypt | `CredentialStoreServiceImpl` | Service logic | `randomKey` re-encrypted under IDA's KMS |

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                  Registration Processor (caller)                    │
│                                                                    │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │
│  │ UinGeneratorStage│  │BiometricExtraction│  │FinalizationStage│  │
│  └────────┬─────────┘  └────────┬──────────┘  └───────┬─────────┘  │
│           │                     │                      │           │
│           └─────────────────────┼──────────────────────┘           │
│                                 │                                  │
│                    ┌────────────▼───────────┐                      │
│                    │   IdrepoDraftService   │                      │
│                    │  (packet-manager)      │                      │
│                    └────────────┬───────────┘                      │
│                                 │  REST calls via                  │
│                                 │  RegistrationProcessorRestClient │
└─────────────────────────────────┼────────────────────────────────┘
                                  │ HTTP
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│              ID Repository — Identity Service (server)              │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │               IdRepoDraftController                          │   │
│  │   HEAD /draft/{id}       → hasDraft()                        │   │
│  │   GET  /draft/{id}       → getDraft()                        │   │
│  │   POST /draft/create/{id}→ createDraft()                     │   │
│  │   PATCH /draft/update/{id}→ updateDraft()                    │   │
│  │   GET  /draft/publish/{id}→ publishDraft()                   │   │
│  │   DELETE /draft/discard/{id}→ discardDraft()                 │   │
│  │   PUT  /draft/extractbiometrics/{id}→ extractBiometrics()    │   │
│  └──────────────────────────┬───────────────────────────────────┘   │
│                             │                                       │
│                  ┌──────────▼────────────┐                         │
│                  │ IdRepoDraftServiceImpl │                         │
│                  └──────────┬────────────┘                         │
│          ┌──────────────────┼─────────────────────┐                │
│          │                  │                     │                │
│   ┌──────▼──────┐  ┌────────▼──────┐  ┌──────────▼──────────┐    │
│   │ UinDraftRepo│  │ObjectStoreHelp│  │IdRepoProxyServiceImpl│    │
│   │  (DB/JPA)   │  │  (S3/MinIO)   │  │  (bio extraction)   │    │
│   └─────────────┘  └───────────────┘  └────────────────────-┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## API Endpoints

All endpoints are under `/idrepository/v1/identity` (the base path) + `/draft`.

| Method   | Path                               | ApiName (reg-proc)     | Purpose                                   |
|----------|------------------------------------|------------------------|-------------------------------------------|
| `HEAD`   | `/draft/{registrationId}`          | `IDREPOHASDRAFT`       | Check if a draft exists (200/204)         |
| `GET`    | `/draft/{registrationId}`          | `IDREPOGETDRAFT`       | Fetch draft identity + docs + biometrics  |
| `POST`   | `/draft/create/{registrationId}`   | `IDREPOCREATEDRAFT`    | Create a new draft (optionally clone UIN) |
| `PATCH`  | `/draft/update/{registrationId}`   | `IDREPOUPDATEDRAFT`    | Update demographics/documents in draft    |
| `GET`    | `/draft/publish/{registrationId}`  | `IDREPOPUBLISHDRAFT`   | Publish draft → make identity live        |
| `DELETE` | `/draft/discard/{registrationId}`  | `IDREPODISCARDDRAFT`   | Discard and delete draft                  |
| `PUT`    | `/draft/extractbiometrics/{id}`    | *(direct from stage)*  | Pre-extract biometrics into object store  |
| `GET`    | `/draft/uin/{UIN}`                 | *(admin use)*          | Lookup draft by UIN                       |

---

## Registration Processor Side

**File:** `registration-processor-packet-manager/src/main/java/.../idreposervice/IdrepoDraftService.java`

This service is the client-side facade used by all three processing stages. It wraps every HTTP call through `RegistrationProcessorRestClientService`, which resolves the URL from `env.getProperty(ApiName.XXX.name())`.

### Method: `idrepoHasDraft(id)`
```
HEAD /draft/{id}
```
- Calls `registrationProcessorRestClientService.headApi(IDREPOHASDRAFT, [id], null, null)`
- Returns `true` if HTTP 200, `false` if HTTP 204
- Throws `IdrepoDraftException` if the response is anything else

### Method: `idrepoGetDraft(id)`
```
GET /draft/{id}
```
- Called internally by `idrepoUpdateDraft` when a draft already exists
- Returns `ResponseDTO` containing identity, documents, biometricReferenceId, status, UIN, registrationId

### Method: `idrepoCreateDraft(id, uin)`
```
POST /draft/create/{id}?UIN={uin}
```
- Creates a brand-new draft
- If `uin` is non-null, the draft is cloned from that UIN's existing identity record

### Method: `idrepoUpdateDraft(id, uin, idRequestDto)` ← Main orchestrator
This is the most important method. It internally decides whether to create or merge:

```
1. HEAD /draft/{id}              ← idrepoHasDraft()
   ├── 204 (no draft found)
   │     └── POST /draft/create/{id}?UIN={uin}   ← idrepoCreateDraft()
   └── 200 (draft exists)
         ├── GET /draft/{id}                      ← idrepoGetDraft()
         │   (preserves UIN + merges identity)
         └── PATCH /draft/update/{id}             ← IDREPOUPDATEDRAFT
               ├── success → returns IdResponseDTO
               └── error   → DELETE /draft/discard/{id}  ← idrepoDiscardDraft()
                             throws IdrepoDraftException or
                             IdrepoDraftReprocessableException
```

### Method: `idrepoPublishDraft(id)`
```
GET /draft/publish/{id}
```
- Called by FinalizationStage after confirming draft exists
- On error with code `IDR-IDS-003` (key manager error) → throws `IdrepoDraftReprocessableException` (retry)
- On any other error → calls `idrepoDiscardDraft(id)` + throws `IdrepoDraftException`

### Method: `idrepoDiscardDraft(id)`
```
DELETE /draft/discard/{id}
```
- Called on error paths to clean up a corrupted or failed draft
- Key manager errors → `IdrepoDraftReprocessableException`
- Other errors → `IdrepoDraftException`

---

## ID Repository Side

**File:** `id-repository-identity-service/src/main/java/.../service/impl/IdRepoDraftServiceImpl.java`

Extends `IdRepoServiceImpl` and implements `IdRepoDraftService<IdRequestDTO, IdResponseDTO>`.

### 1. `hasDraft`

```
HEAD /draft/{registrationId}
```

**What it does:**
```java
return uinDraftRepo.existsByRegId(regId);
```
- Single DB query: `SELECT COUNT(*) FROM uin_draft WHERE reg_id = ?`
- Returns `true` → controller sends HTTP 200
- Returns `false` → controller sends HTTP 204
- DB failure → `IdRepoAppException(DATABASE_ACCESS_ERROR)`

---

### 2. `createDraft`

```
POST /draft/create/{registrationId}?UIN={uin}
```

**What it does:**

```
1. Check for duplicate: uinHistoryRepo.existsByRegId() OR uinDraftRepo.existsByRegId()
   └── if exists & forceMerge disabled → throw RECORD_EXISTS

2. If forceMerge enabled:
   └── proxyService.retrieveIdentityByRid() → fetch existing identity → extract UIN

3a. If UIN provided (update/correction flow):
    ├── uinRepo.findByUinHash()              ← DB: load existing identity
    ├── mapper.convertValue(uinObject, UinDraft.class)
    └── updateBiometricAndDocumentDrafts()   ← O(N) merge of bio + doc lists

3b. If UIN not provided (new registration flow):
    ├── idRepoServiceHelper.generateUin()    ← HTTP call to kernel UIN generator
    │     (Propagation.NOT_SUPPORTED — releases DB connection during REST call)
    ├── generateIdentityObject(uin)          ← builds {"identity": {"UIN": "..."}}
    └── converts to bytes + hashes

4. Set statusCode = "DRAFT", save to uinDraftRepo (DB)
5. Return constructIdResponse("DRAFTED")
```

**Internal service calls:**
- `uinRepo.findByUinHash()` — DB lookup of existing UIN
- `idRepoServiceHelper.generateUin()` — REST call to `mosip-kernel-idgenerator-uin` service
- `uinDraftRepo.save()` — persist new draft row

---

### 3. `getDraft`

```
GET /draft/{registrationId}
```

**What it does:**

```
1. uinDraftRepo.findByRegId(regId)         ← DB: load UinDraft entity

2. For each UinBiometricDraft in draft:
   └── extractAndGetCombinedCbeff()
       ├── objectStoreHelper.getBiometricObject(uinHash, bioFileId)  ← S3/MinIO: fetch CBEFF
       └── proxyService.getBiometricsForRequestedFormats()           ← extract if formats requested

3. For each UinDocumentDraft in draft:
   └── objectStoreHelper.getDemographicObject(uinHash, docId)        ← S3/MinIO: fetch document

4. constructIdResponse(draft.getUinData(), statusCode, documents)
   └── parses UIN data bytes → ObjectNode
       moves verifiedAttributes to top level
       returns identity + documents in response
```

**Internal service calls:**
- `uinDraftRepo.findByRegId()` — DB
- `objectStoreHelper.getBiometricObject()` — Object Store (S3/MinIO)
- `objectStoreHelper.getDemographicObject()` — Object Store (S3/MinIO)
- `proxyService.getBiometricsForRequestedFormats()` — ABIS/Extraction service (if extraction formats requested)

---

### 4. `updateDraft`

```
PATCH /draft/update/{registrationId}
```

**What it does:**

```
1. uinDraftRepo.findByRegId(registrationId)      ← DB: load draft

2a. If uinData is null (first update, new registration):
    ├── convert request.identity → bytes
    ├── set uinData + uinDataHash
    └── updateDocuments() → upload docs to object store

2b. If uinData exists (subsequent update):
    ├── updateDemographicData()
    │   ├── parse existing uinData (JSON)
    │   ├── parse incoming identity (JSON)
    │   ├── updateVerifiedAttributes()           ← merge verified attr list
    │   ├── JSONCompare.compareJSON()            ← detect changed fields
    │   └── updateJsonObject()                  ← apply diff to db data
    └── updateDocuments()
        ├── mapper.convertValue(draft → Uin)
        ├── super.updateDocuments()             ← upload new doc files to S3
        └── updateBiometricAndDocumentDrafts()  ← sync bio/doc lists (O(N))

3. uinDraftRepo.save(draftToUpdate)              ← DB: persist changes
4. Return constructIdResponse("DRAFTED")
```

**Internal service calls:**
- `uinDraftRepo.findByRegId()` — DB
- `objectStoreHelper` (via `super.updateDocuments()`) — S3/MinIO document upload
- `uinDraftRepo.save()` — DB

---

### 5. `publishDraft`

```
GET /draft/publish/{registrationId}
```

**What it does — this is the most complex operation:**

```
1. uinDraftRepo.findByRegId(regId)              ← DB: load draft

2. anonymousProfileHelper.setNewCbeff()         ← set biometric reference for profile

3. buildRequest(regId, draft)
   └── parse UinDraft.uinData → IdRequestDTO
       (removes verifiedAttributes from identity map)

4. validateRequest(idRequest.getRequest())       ← schema validation

5. decryptUin(draft.getUin(), draft.getUinHash())
   └── uinEncryptSaltRepo.getOne()              ← DB: fetch salt
       securityManager.decryptWithSalt()        ← decrypt UIN

6. uinRepo.existsByUinHash(draft.getUinHash())  ← DB: is this a new or existing identity?

   Case A — Existing UIN (update / correction / reactivation):
   └── super.updateIdentity(idRequest, uin)
       ├── update uin table row                 ← DB
       ├── update uin_history                   ← DB
       └── (document/bio updates handled by publishDocuments step)

   Case B — New UIN (new registration):
   ├── vidDraftHelper.generateDraftVid(uin)     ← REST call to VID service
   ├── super.addIdentity(idRequest, uin)
   │   ├── insert into uin table               ← DB
   │   ├── insert into uin_history             ← DB
   │   └── store documents/biometrics in S3   ← Object Store
   └── vidDraftHelper.activateDraftVid(draftVid) ← REST call to VID service

7. anonymousProfileHelper.buildAndsaveProfile(true)  ← save anonymous profile

8. publishDocuments(draft, uinObject)
   ├── uinBiometricRepo.saveAll()              ← DB: copy bio records from draft
   └── uinDocumentRepo.saveAll()              ← DB: copy doc records from draft

9. uinDraftRepo.deleteByRegId(regId)           ← DB: delete draft (cleanup)

10. Return constructIdResponse(uinObject.getStatusCode(), draftVid)
```

**Internal service calls:**
- `uinDraftRepo.findByRegId()` — DB
- `uinEncryptSaltRepo.getOne()` — DB (salt fetch for UIN decryption)
- `uinRepo.existsByUinHash()` — DB
- `vidDraftHelper.generateDraftVid()` — REST call to VID Generator service
- `super.addIdentity()` / `super.updateIdentity()` — DB + S3
- `vidDraftHelper.activateDraftVid()` — REST call to VID service
- `uinBiometricRepo.saveAll()` — DB
- `uinDocumentRepo.saveAll()` — DB
- `uinDraftRepo.deleteByRegId()` — DB (delete draft)

---

### 6. `discardDraft`

```
DELETE /draft/discard/{registrationId}
```

**What it does:**

```
1. uinDraftRepo.existsByRegId(regId)     ← DB: check exists
   └── if not found → throw NO_RECORD_FOUND

2. uinDraftRepo.deleteByRegId(regId)     ← DB: delete draft row + cascade

3. Return constructIdResponse("DISCARDED")
```

---

### 7. `extractBiometrics`

```
PUT /draft/extractbiometrics/{registrationId}
```

**What it does:**

```
1. If extractionFormats is empty → return immediately (no-op)

2. uinDraftRepo.findByRegId(registrationId)    ← DB: load draft

3. For each UinBiometricDraft:
   a. deleteExistingExtractedBioData()
      └── for each format:
          objectStoreHelper.deleteBiometricObject()  ← S3: delete stale file (non-fatal)

   b. extractAndGetCombinedCbeff()
      ├── objectStoreHelper.getBiometricObject()     ← S3: fetch raw CBEFF
      └── proxyService.getBiometricsForRequestedFormats()
          └── calls ABIS/extraction service → stores result in S3

4. Return constructIdResponse("DRAFTED")
```

Runs with `@Transactional(propagation = Propagation.NOT_SUPPORTED)` — DB connection is released before any S3 or extraction I/O begins.

---

## Stage-Level Flows

### UIN Generator Stage

Called for: **New Registration, Update, Correction, Re-activation, De-activation, Lost UIN**

```
UinGeneratorStage.process()
    │
    └── idrepoDraftService.idrepoUpdateDraft(id, uin, idRequestDTO)
        │
        ├── HEAD /draft/{id}                           (idrepoHasDraft)
        │   ├── 204 → POST /draft/create/{id}?UIN=uin  (idrepoCreateDraft)
        │   └── 200 → GET /draft/{id}                  (idrepoGetDraft)
        │              └── merge UIN into new identity payload
        │
        └── PATCH /draft/update/{id}                  (IDREPOUPDATEDRAFT)
            ├── success → IdResponseDTO returned
            └── error   → DELETE /draft/discard/{id}   (idrepoDiscardDraft)
```

`idrepoUpdateDraft` is called in 5 different contexts within UinGeneratorStage:
- `addIdentityToIdRepo()` — new registration (line 618)
- `idRepoRequestBuilder()` — update/correction (line 812, 1008)
- `reActivateUin()` — re-activation (line 879)
- Lost UIN flow (line 1182)

### Biometric Extraction Stage

```
BiometricExtractionStage.process()
    │
    ├── HEAD /draft/{id}                               (idrepoHasDraft)
    │   └── 204 → FAIL: mark DRAFT_REQUEST_UNAVAILABLE
    │
    └── 200 → getExtractors(registrationId)            (PMS API call)
              └── addBiometricExtractiontoIdRepository()
                  └── PUT /draft/extractbiometrics/{id} (for each extractor)
                      ├── success → PROCESSING
                      └── error   → DELETE /draft/discard/{id}
```

### Finalization Stage

```
FinalizationStage.process()
    │
    ├── HEAD /draft/{id}                               (idrepoHasDraft)
    │   └── 204 → FAIL: mark DRAFT_REQUEST_UNAVAILABLE
    │
    └── 200 → GET /draft/publish/{id}                  (idrepoPublishDraft)
              ├── success → mark PROCESSING + pass to next stage
              ├── IDR-IDS-003 error → IdrepoDraftReprocessableException (retry)
              └── other error → DELETE /draft/discard/{id} + FAILED
```

---

## Internal Service Dependencies

| Service / Component                  | Used by                    | Purpose                                                |
|--------------------------------------|----------------------------|--------------------------------------------------------|
| `UinDraftRepo` (JPA)                 | All draft methods          | CRUD on `uin_draft` table                              |
| `UinRepo` (JPA)                      | createDraft, publishDraft  | Lookup/check existing live identity                    |
| `UinBiometricRepo` (JPA)             | publishDraft               | Persist bio records from draft to live table           |
| `UinDocumentRepo` (JPA)              | publishDraft               | Persist doc records from draft to live table           |
| `UinHistoryRepo` (JPA)               | createDraft (duplicate check) | Check if reg ID already processed                   |
| `UinEncryptSaltRepo` (JPA)           | publishDraft               | Fetch salt for UIN decryption                          |
| `ObjectStoreHelper` (S3/MinIO)       | getDraft, updateDraft, extractBiometrics | Read/write CBEFF and document files   |
| `IdRepoServiceHelper`                | createDraft                | `generateUin()` — REST call to UIN generator           |
| `IdRepoProxyServiceImpl`             | getDraft, extractBiometrics, createDraft (forceMerge) | Bio extraction + identity retrieval |
| `VidDraftHelper`                     | publishDraft (new identity)| Generate + activate VID for new registrant             |
| `AnonymousProfileHelper`             | publishDraft               | Build and save anonymous profile data                  |
| `IdRequestValidator`                 | updateDraft, publishDraft  | Validate incoming request schema                       |
| `AuditHelper`                        | All controller methods     | Record audit trail for every operation                 |
| `IdRepoSecurityManager`              | All service methods        | Cryptographic operations + user context                |

---

## Error Handling

### Registration Processor Exceptions

| Exception                          | When thrown                              | Effect on stage                  |
|------------------------------------|------------------------------------------|----------------------------------|
| `IdrepoDraftException`             | Draft API returns error (non-retryable)  | Stage marks packet as FAILED     |
| `IdrepoDraftReprocessableException`| Key manager error (code `IDR-IDS-003`)   | Stage marks packet as PROCESSING (retry) |
| `ApisResourceAccessException`      | Network/HTTP error to id-repo            | Stage marks packet as PROCESSING (retry) |

### ID Repository Exceptions

| Exception                  | Error Code              | Cause                                         |
|----------------------------|-------------------------|-----------------------------------------------|
| `IdRepoAppException`       | `NO_RECORD_FOUND`       | Draft row not in DB                           |
| `IdRepoAppException`       | `RECORD_EXISTS`         | Duplicate reg ID, draft already created       |
| `IdRepoAppException`       | `DATABASE_ACCESS_ERROR` | JPA / JDBC / transaction failure              |
| `IdRepoAppException`       | `UIN_GENERATION_FAILED` | Kernel UIN generator returned error           |
| `IdRepoAppException`       | `BIO_EXTRACTION_ERROR`  | Biometric extraction failed                   |
| `IdRepoAppException`       | `UNKNOWN_ERROR`         | JSON parse error or unexpected failure        |
| `IdRepoAppUncheckedException` | `UIN_HASH_MISMATCH`  | Decrypted UIN hash doesn't match stored hash  |

---

## Interceptors & Filters

Every layer in this pipeline applies encryption/decryption either at the **HTTP request/response level** (Servlet Filters) or at the **database read/write level** (Hibernate Interceptors). Neither layer is visible in business logic — both operate transparently.

---

### Hibernate Interceptors (DB-level transparent encrypt/decrypt)

#### 1. `IdRepoEntityInterceptor` — id-repository-identity-service

File: `id-repository-identity-service/.../interceptor/IdRepoEntityInterceptor.java`

Registered as a Hibernate `Interceptor` on the `SessionFactory`. Fires on every JPA save/load/update touching identity entities.

| Hibernate hook | Entities affected | What it does |
|---|---|---|
| `onSave()` | `Uin`, `UinDraft`, `UinHistory`, `HandleInfo` | Encrypts `uinData` field via `securityManager.encrypt(uinData, uinDataRefId)` (AES-based). Encrypts `uin` field via `securityManager.encryptWithSalt(uin, salt)` where salt is derived from the UIN string. Same treatment for `handle` in `HandleInfo`. |
| `onLoad()` | `Uin`, `UinDraft`, `UinHistory` | Decrypts `uinData` via `securityManager.decrypt(encryptedData, uinDataRefId)`. Then verifies `SHA-256(decryptedData) == uinDataHash`; throws `IDENTITY_HASH_MISMATCH` error if tampered. |
| `onFlushDirty()` | same | Re-encrypts `uinData` and `uin` on any UPDATE before Hibernate flushes to DB. |

This means that **all reads from and writes to `uin_draft`, `uin`, `uin_history` tables are always encrypted at rest**. The service layer always works with plain-text objects — it never calls encrypt/decrypt explicitly.

#### 2. `CredentialTransactionInterceptor` — credential-request-generator

File: `credential-request-generator/.../interceptor/CredentialTransactionInterceptor.java`

Registered as a Hibernate Interceptor on the `credential_request_status` table's `CredentialEntity`.

| Hibernate hook | What it does |
|---|---|
| `onSave()` | Encodes `request` bytes as Base64URLSafe, then encrypts via `cryptoUtil.encryptData()`. Stores encrypted string. |
| `onLoad()` | Decrypts `request` field via `cryptoUtil.decryptData()`, then Base64URLSafe-decodes to plain text. Falls back to raw (un-encrypted) value if decryption fails — backward compatibility with MOSIP 1.1.5.5. Respects a thread-local `CryptoContext.isSkipDecryption()` flag for batch processing scenarios. |
| `onFlushDirty()` | Re-encrypts `request` on UPDATE. |

This means the credential request JSON stored in `credential_request_status` is always encrypted at rest.

#### 3. `IdRepoVidEntityInterceptor` — id-repository-vid-service

File: `id-repository-vid-service/.../interceptor/IdRepoVidEntityInterceptor.java`

Same pattern as `IdRepoEntityInterceptor` but for VID entities (`Vid`, `VidHistory`). Encrypts/decrypts `vid` and `vidData` fields transparently on all VID table operations.

---

### Servlet Filters (HTTP-level)

#### id-repository: `BaseIdRepoFilter` / `IdRepoFilter`

File: `id-repository-core/.../filter/BaseIdRepoFilter.java`
File: `id-repository-identity-service/.../filter/IdRepoFilter.java`

These are **request timing and logging filters only** — they do NOT perform encryption or decryption.

- Logs the request URL and timestamps (request time, response time, duration in ms)
- Skips static asset paths (swagger, webjars, icons, css, js)
- `buildResponse()` returns `null` in the concrete `IdRepoFilter` — it is a pure pass-through

All actual encryption in id-repository happens at the Hibernate interceptor layer (above) and inside `IdRepoSecurityManager` which is called explicitly by service methods.

---

#### id-authentication: `BaseIDAFilter` → `BaseAuthFilter` → `IdAuthFilter`

File: `authentication-common/.../filter/BaseIDAFilter.java`  
File: `authentication-common/.../filter/BaseAuthFilter.java`  
File: `authentication-common/.../filter/IdAuthFilter.java`

These filters form a **three-tier abstract hierarchy**. Every external authentication endpoint (`/auth`, `/kyc`, etc.) passes through the full chain before reaching the Spring controller.

**Concrete filter classes that extend `IdAuthFilter`:**
- `ExternalAuthFilter` — external partner auth
- `KycAuthenticationFilter` / `KycAuthFilter` / `KycExchangeFilter` — KYC flows
- `OTPFilter` / `InternalOtpFilter` — OTP
- `InternalAuthFilter` — internal service auth
- `IdentityKeyBindingFilter`, `VciExchangeFilter` — advanced flows

**Full per-request filter execution order:**

```
HTTP Request (encrypted + signed)
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│ BaseIDAFilter.doFilter()                                 │
│  1. Skip swagger/actuator/callback URLs                  │
│  2. Wrap request in ResettableStreamHttpServletRequest   │
│  3. Read raw JSON body                                   │
│  4. Validate `id` field matches configured API ID        │
│  5. Validate `version` field format                      │
│  6. → consumeRequest()                                   │
└──────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│ BaseAuthFilter.consumeRequest()                          │
│  A. authenticateRequest()                                │
│     └─ verifies JWS `signature` HTTP header via         │
│        keyManager (RSA cert from Key Manager service)    │
│        Validates certificate trust chain if required     │
│                                                          │
│  B. decipherAndValidateRequest()                         │
│     └─ calls decipherRequest() + validateDecipheredRequest│
└──────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│ IdAuthFilter.decipherRequest()  ← THE DECRYPT STEP       │
│                                                          │
│  request field (base64url) ──────────────────────────►  │
│  requestSessionKey (base64url encrypted sym key)         │
│  requestHMAC (base64url encrypted HMAC)                  │
│  thumbprint (cert thumbprint of partner)                 │
│                                                          │
│  Step 1: decode(requestSessionKey) → encryptedSessionKey │
│  Step 2: keyManager.kernelDecryptAndDecode(              │
│            thumbprint, encryptedSessionKey,              │
│            encryptedHMAC, IDA_refId)                     │
│          → RSA-decrypts session key using IDA private key│
│          → AES-decrypts HMAC using session key           │
│          → returns plain HMAC string                     │
│                                                          │
│  Step 3: keyManager.requestData(requestBody, ...)        │
│          → AES-decrypts the `request` field using session│
│            key; validates HMAC of decrypted payload      │
│          → returns plain Map<String,Object>              │
│                                                          │
│  Step 4: if biometrics present → decipherBioData()       │
│    per segment:                                          │
│      - decode JWS data field (base64url payload)         │
│      - verify biometric device digitalId JWS signature   │
│      - compute salt = Base64(XOR(timestamp,txnId)[-2:])  │
│        and aad  = Base64(XOR(timestamp,txnId)[-3:])      │
│      - keyManager.kernelDecrypt(thumbprint,              │
│            encryptedBioSessionKey, encryptedBioValue,    │
│            BIO_refId, aad, salt)                         │
│          → AES-GCM decrypts each BDB (biometric data block)│
│                                                          │
│  Step 5: Replace request field in body with decrypted map│
└──────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│ IdAuthFilter.validateDecipheredRequest()                 │
│  - Parse partner cert thumbprint from signature header   │
│  - partnerService.validateAndGetPolicy(partnerId, ...)   │
│  - checkMispPolicyAllowed()                              │
│  - checkAllowedAuthTypeBasedOnPolicy() (demo/bio/otp/pin)│
│  - checkMandatoryAuthTypeBasedOnPolicy()                 │
│  - addMetadata() → attaches partner info to request map  │
└──────────────────────────────────────────────────────────┘
        │
        ▼
  Spring Controller (receives plain-text decrypted request)
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│ BaseIDAFilter.consumeResponse()  ← THE SIGN STEP         │
│  - keyManager.signResponse(responseAsString)             │
│    → Signs response JSON with IDA private key (JWS)      │
│  - Adds `response-signature` HTTP response header        │
│  - Stores auth transaction (if needStoreAuthTransaction) │
│  - Stores anonymous profile (if needStoreAnonymousProfile│
└──────────────────────────────────────────────────────────┘
        │
        ▼
HTTP Response (plain JSON body + JWS signature header)
```

**Key manager reference IDs used:**
- `IDA_refId` = `${mosip.ida.auth.partner.id}` (env config) — used to decrypt the request session key
- `BIO_refId` = `${mosip.ida.auth.partner.bio.id}` (env config) — used to decrypt biometric BDB values

---

### Encrypt/Decrypt — Where Each Layer Operates

```
LAYER                   SERVICE                  WHAT IS ENCRYPTED
─────────────────────── ──────────────────────── ────────────────────────────────────
HTTP Filter (request)   id-authentication        - request field (AES, session key)
                                                 - biometric BDB values (AES-GCM)
                                                 - HMAC of request (AES)
HTTP Filter (response)  id-authentication        - response signature (JWS header)
HTTP Filter             id-repository            No crypto — timing/logging only
Hibernate Interceptor   id-repository            - uinData column (AES, uinDataRefId)
                                                 - uin column (AES+Salt)
                                                 - handle column (AES+Salt)
Hibernate Interceptor   credential-request-gen   - request column in credential_request_status
Hibernate Interceptor   id-repository vid-svc    - vid + vidData columns
Explicit service call   id-repository            - UIN decrypt before publishDraft
                        (securityManager.         (securityManager.decryptWithSalt)
                         decryptWithSalt)
Data share download     id-authentication        - reEncryptAndStoreRandomKey (KMS)
(CredentialStoreService)                           re-encrypts random key under IDA's KMS
                                                 - demographicData / biometricData
                                                   stored in identity_cache as-is
                                                   (already decrypted from data share)
```

---

## Sequence Diagram

See `DRAFT_LIFECYCLE_SEQUENCE.mermaid` in this repository for the full visual sequence diagram.

The diagram covers all three stage flows:
1. **UIN Generator Stage** — `idrepoUpdateDraft` chain (hasDraft → create or merge → update)
2. **Biometric Extraction Stage** — `idrepoHasDraft` guard + `extractBiometrics`
3. **Finalization Stage** — `idrepoHasDraft` guard + `publishDraft`
