---
title: "SSO Series Part 8: Certificate Rotation & Key Management | SSO 系列之八：證書輪換與金鑰管理"
date: 2026-04-28 15:15:00 +0800
categories: [Security, SSO Series]
tags: [sso, certificates, x509, saml, jwks, key-rotation, security, typescript, nodejs, assisted_by_ai]
mermaid: true
toc: true
---

## Introduction: The Ticking Time Bomb

Certificates expire. Keys get compromised. IdPs rotate their signing credentials on a regular schedule—typically every 1-2 years for SAML X.509 certificates, and much more frequently for OIDC JWKS (some providers rotate keys every 24 hours). If our application caches a certificate that suddenly becomes invalid, every SSO login attempt will fail. For a large enterprise, this means hundreds of users locked out simultaneously.

In Part 8, we will implement `SSO-008`: **Automated Certificate Rotation & Key Management**. We will build systems to automatically discover new keys, gracefully handle key transitions (supporting both old and new keys during a rollover window), and alert administrators before certificates expire.

---

## 1. The Certificate Lifecycle

Every certificate and signing key goes through a predictable lifecycle. Understanding this lifecycle is the first step to managing it.

### Mermaid Diagram: Certificate Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> Unknown: Not yet discovered

    Unknown --> Active: First seen via<br/>Auto-Discovery

    Active --> ExpiringSoon: Validity period<br/>approaching end
    Active --> Revoked: IdP revokes<br/>(compromise)

    ExpiringSoon --> Active: IdP publishes<br/>new certificate
    ExpiringSoon --> Expired: NotBefore/NotOnOrAfter<br/>passed

    Revoked --> [*]: Removed from cache

    Expired --> [*]: Removed from cache

    note right of Active
        In JWKS: key is in the key set
        In SAML: certificate in metadata
    end note

    note right of ExpiringSoon
        Alert threshold: 30 days
        for SAML, N/A for JWKS
        (auto-rotated by IdP)
    end note
```

---

## 2. OIDC JWKS Key Rotation

OIDC providers publish their public keys at the `jwks_uri` endpoint. These keys have a `kid` (Key ID) that uniquely identifies each key. When a provider rotates keys, they typically:

1. Add the new key to the JWKS endpoint (both old and new keys coexist)
2. Start signing new tokens with the new key
3. Remove the old key after a grace period

### Mermaid Diagram: OIDC JWKS Key Rotation Flow

```mermaid
sequenceDiagram
    autonumber
    participant SP as Service Provider
    participant Cache as JWKS Cache<br/>(jwks-rsa)
    participant IdP as Identity Provider

    Note over SP,IdP: Normal Operation
    SP->>Cache: Need public key for kid: "key-001"
    Cache->>Cache: Cache HIT ✅
    Cache->>SP: Return public key

    Note over SP,IdP: Key Rotation Begins (IdP adds new key)
    SP->>Cache: Need public key for kid: "key-002"
    Cache->>Cache: Cache MISS for kid: "key-002"
    Cache->>IdP: GET /jwks
    IdP->>Cache: {"keys": [<br/>  {"kid": "key-001", ...},<br/>  {"kid": "key-002", ...}<br/>]}
    Cache->>Cache: Cache new key set
    Cache->>SP: Return public key for "key-002"

    Note over SP,IdP: Old Key Removed (after grace period)
    SP->>Cache: Need public key for kid: "key-001"
    Cache->>Cache: Cache HIT but key not in new set
    Cache->>IdP: GET /jwks (refresh)
    IdP->>Cache: {"keys": [<br/>  {"kid": "key-002", ...}<br/>]}
    Cache->>SP: key-001 not found ❌
    Note over SP: Token with kid: "key-001" now invalid
```

### Code Implementation: Enhanced JWKS Service with Rotation Handling

```typescript
// src/sso/services/jwks.service.ts
import { Injectable, Logger, InternalServerErrorException } from '@nestjs/common';
import * as jwksClient from 'jwks-rsa';
import { IdpProvider } from '../entities/idp-provider.entity';

@Injectable()
export class JwksService {
  private readonly logger = new Logger(JwksService.name);
  private clients = new Map<string, jwksClient.JwksClient>();

  private getClient(provider: IdpProvider, jwksUri: string): jwksClient.JwksClient {
    if (!this.clients.has(provider.id)) {
      const client = jwksClient({
        jwksUri: jwksUri,
        cache: true,
        cacheMaxEntries: 10, // Allow more keys during rotation
        cacheMaxAge: 600000, // 10 minutes — shorter to pick up new keys faster
        rateLimit: true,
        jwksRequestsPerMinute: 10,
        // Handle key rotation gracefully
        getKeysInterceptor: (keys) => {
          this.logger.debug(`JWKS fetched: ${keys.length} keys available for ${provider.providerCode}`);
          return keys;
        },
      });
      this.clients.set(provider.id, client);
    }
    return this.clients.get(provider.id);
  }

  async getPublicKey(provider: IdpProvider, jwksUri: string, kid: string): Promise<string> {
    try {
      const client = this.getClient(provider, jwksUri);
      const key = await client.getSigningKey(kid);
      return key.getPublicKey();
    } catch (error) {
      // If kid not found, force-refresh the JWKS and retry once
      if (error.message?.includes('Unable to find a signing key')) {
        this.logger.warn(`Key ${kid} not found in cache, forcing JWKS refresh for ${provider.providerCode}`);
        this.invalidateClient(provider);
        
        try {
          const client = this.getClient(provider, jwksUri);
          const key = await client.getSigningKey(kid);
          return key.getPublicKey();
        } catch (retryError) {
          throw new InternalServerErrorException(
            `Key ${kid} not found even after JWKS refresh. Key may have been revoked.`
          );
        }
      }
      throw error;
    }
  }

  /**
   * Force-invalidate the cached JWKS client for a provider.
   * Useful when we detect a key mismatch.
   */
  invalidateClient(provider: IdpProvider): void {
    this.clients.delete(provider.id);
    this.logger.log(`JWKS cache invalidated for provider: ${provider.providerCode}`);
  }
}
```

---

## 3. SAML X.509 Certificate Rotation

SAML certificate rotation is more complex because certificates are embedded in metadata XML, and there's no standard auto-rotation mechanism like JWKS. Admins must manually update the metadata.

### Mermaid Diagram: SAML Certificate Rotation Challenge

```mermaid
flowchart TD
    subgraph "Before Rotation"
        META_OLD["SAML Metadata (Old)<br/>Contains: Certificate A"]
        STORED_OLD["Our DB:<br/>certificates = [Cert A]"]
        VERIFY_OLD["Signature Verification<br/>Against Cert A ✅"]
    end

    subgraph "During Rotation (Transition Period)"
        META_NEW["SAML Metadata (New)<br/>Contains: Cert A + Cert B"]
        STORED_TRANS["Our DB:<br/>certificates = [Cert A, Cert B]"]
        VERIFY_TRANS["Signature Verification<br/>Against Cert A or Cert B ✅"]
    end

    subgraph "After Rotation (Old Cert Removed)"
        META_FINAL["SAML Metadata (Final)<br/>Contains: Cert B only"]
        STORED_NEW["Our DB:<br/>certificates = [Cert B]"]
        VERIFY_NEW["Signature Verification<br/>Against Cert B ✅"]
    end

    META_OLD --> STORED_OLD --> VERIFY_OLD
    META_NEW --> STORED_TRANS --> VERIFY_TRANS
    META_FINAL --> STORED_NEW --> VERIFY_NEW

    style STORED_TRANS fill:#f39c12,color:#fff
```

### Code Implementation: Certificate Refresh Service

```typescript
// src/sso/services/certificate-refresh.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { SamlDiscoveryService } from './saml-discovery.service';
import { IdpProviderRepository } from '../repositories/idp-provider.repository';
import { IdpSecretManagerService } from './idp-secret-manager.service';

@Injectable()
export class CertificateRefreshService {
  private readonly logger = new Logger(CertificateRefreshService.name);

  constructor(
    private readonly samlDiscovery: SamlDiscoveryService,
    private readonly idpProviderRepo: IdpProviderRepository,
    private readonly secretManager: IdpSecretManagerService,
  ) {}

  /**
   * Runs every 6 hours to check for certificate updates.
   */
  @Cron(CronExpression.EVERY_6_HOURS)
  async refreshSamlCertificates(): Promise<void> {
    this.logger.log('Starting scheduled SAML certificate refresh...');

    const samlProviders = await this.idpProviderRepo.findAll({
      protocolType: 'SAML2',
      isEnabled: true,
      autoDiscovery: true,
    });

    for (const provider of samlProviders) {
      try {
        await this.refreshProviderCertificates(provider);
      } catch (error) {
        this.logger.error(
          `Failed to refresh certificates for ${provider.providerCode}: ${error.message}`
        );
        // Emit alert for admin notification
        // this.eventEmitter.emit('certificate.refresh_failed', { providerId: provider.id });
      }
    }
  }

  private async refreshProviderCertificates(provider: IdpProvider): Promise<void> {
    // 1. Decrypt current config to get metadata URL
    const config = await this.secretManager.decryptProviderConfig(provider);
    const metadataUrl = config.metadataUrl;

    if (!metadataUrl) {
      this.logger.debug(`No metadata URL for ${provider.providerCode}, skipping`);
      return;
    }

    // 2. Fetch latest metadata
    const metadata = await this.samlDiscovery.fetchMetadata(metadataUrl);
    const newCerts = metadata.certificates;

    if (!newCerts || newCerts.length === 0) {
      this.logger.warn(`No certificates found in metadata for ${provider.providerCode}`);
      return;
    }

    // 3. Compare with stored certificates
    const currentCerts = config.idpPublicCertificates || [];
    const certsChanged = !this.arraysEqual(currentCerts.sort(), newCerts.sort());

    if (certsChanged) {
      // 4. Update the config with new certificates
      config.idpPublicCertificates = newCerts;
      const { encryptedBlob, wrappedDek } = await this.secretManager.encryptProviderConfig(
        provider.id, config
      );

      await this.idpProviderRepo.update(provider.id, {
        configEncrypted: encryptedBlob,
        configDekWrapped: wrappedDek,
      });

      this.logger.log(`Updated certificates for ${provider.providerCode}: ${newCerts.length} certificate(s)`);

      // 5. Check for expiry warnings
      this.checkCertificateExpiry(provider.providerCode, newCerts);
    }
  }

  private checkCertificateExpiry(providerCode: string, certificates: string[]): void {
    for (const cert of certificates) {
      // Parse X.509 certificate to check expiry
      // In production, use a library like 'node-forge' or 'pkijs'
      try {
        const certInfo = this.parseCertificate(cert);
        const daysUntilExpiry = Math.floor(
          (certInfo.validTo.getTime() - Date.now()) / (1000 * 60 * 60 * 24)
        );

        if (daysUntilExpiry < 30) {
          this.logger.warn(
            `Certificate for ${providerCode} expires in ${daysUntilExpiry} days! ` +
            `Expiry: ${certInfo.validTo.toISOString()}`
          );
          // Emit alert
          // this.eventEmitter.emit('certificate.expiring_soon', { providerCode, daysUntilExpiry });
        }
      } catch (error) {
        this.logger.debug(`Could not parse certificate expiry for ${providerCode}`);
      }
    }
  }

  private arraysEqual(a: string[], b: string[]): boolean {
    return a.length === b.length && a.every((val, idx) => val === b[idx]);
  }
}
```

### Mermaid Diagram: Certificate Refresh Scheduler Flow

```mermaid
flowchart TD
    CRON["Cron Job<br/>Every 6 hours"] --> QUERY["Query DB:<br/>All enabled SAML providers<br/>with autoDiscovery = true"]
    QUERY --> LOOP["For each provider"]
    
    LOOP --> DECRYPT["Decrypt provider config<br/>(Envelope Encryption)"]
    DECRYPT --> URL{"Has<br/>metadataUrl?"}
    URL -->|"No"| SKIP["Skip this provider"]
    URL -->|"Yes"| FETCH["Fetch latest SAML metadata XML"]
    
    FETCH --> PARSE["Parse XML → Extract certificates"]
    PARSE --> COMPARE{"Certificates<br/>changed?"}
    
    COMPARE -->|"No"| NEXT["Move to next provider"]
    COMPARE -->|"Yes"| UPDATE["Re-encrypt config<br/>with new certificates"]
    UPDATE --> SAVE["Save to DB"]
    SAVE --> EXPIRY["Check certificate expiry"]
    
    EXPIRY --> EXPIRING{"Expiring in<br/>< 30 days?"}
    EXPIRING -->|"Yes"| ALERT["⚠️ Send admin alert<br/>via email / webhook"]
    EXPIRING -->|"No"| NEXT

    style CRON fill:#3498db,color:#fff
    style ALERT fill:#e74c3c,color:#fff
```

---

## 4. Graceful Key Rollover: Supporting Multiple Keys

During a transition period, the IdP might sign tokens with either the old or the new key. Our application must accept both.

### Mermaid Diagram: Multi-Key Verification Strategy

```mermaid
sequenceDiagram
    autonumber
    participant User as User's Browser
    participant SP as Service Provider
    participant Cache as Key Cache
    participant IdP as Identity Provider

    Note over SP,IdP: Token 1 (Signed with OLD key)
    User->>SP: OIDC Callback (token: kid=old-key)
    SP->>Cache: Get public key for kid=old-key
    Cache->>SP: Old public key ✅
    SP->>SP: Verify signature with old key ✅
    SP->>User: Login success

    Note over SP,IdP: Token 2 (Signed with NEW key)
    User->>SP: OIDC Callback (token: kid=new-key)
    SP->>Cache: Get public key for kid=new-key
    Cache->>Cache: Cache MISS → Fetch JWKS
    Cache->>SP: New public key ✅
    SP->>SP: Verify signature with new key ✅
    SP->>User: Login success
```

The key insight: by using the `kid` (Key ID) in the JWT header, we always fetch the *correct* key for verification. As long as the JWKS endpoint includes both keys during the transition, both old and new tokens work seamlessly.

---

## 5. SP Certificate Management (Our Signing Keys)

For SAML, we also need to manage our own signing certificates (used to sign AuthnRequests and encrypt assertions). These have the same expiry problem.

### Mermaid Diagram: SP Certificate Rotation Flow

```mermaid
flowchart TD
    subgraph "Current State"
        SP_CERT_A["SP Certificate A<br/>(Active, used for signing)"]
        IDP_CONFIG["IdP has Cert A<br/>in their trust store"]
    end

    subgraph "Step 1: Generate New Certificate"
        GEN["Generate SP Certificate B<br/>(Not yet trusted by IdP)"]
    end

    subgraph "Step 2: Dual Certificate Period"
        DUAL["Both Cert A and Cert B<br/>stored in our config"]
        DUAL_SIGN["Sign AuthnRequests<br/>with NEW Cert B"]
        DUAL_DECRYPT["Decrypt Assertions<br/>with EITHER Cert A or B"]
    end

    subgraph "Step 3: Notify IdP Admin"
        NOTIFY["Send new certificate to IdP admin<br/>to add to their trust store"]
    end

    subgraph "Step 4: Retire Old Certificate"
        RETIRE["Remove Cert A<br/>Use only Cert B"]
    end

    SP_CERT_A --> GEN
    GEN --> DUAL
    DUAL --> NOTIFY
    NOTIFY --> DUAL_SIGN
    DUAL_SIGN --> DUAL_DECRYPT
    DUAL_DECRYPT --> RETIRE

    style DUAL fill:#f39c12,color:#fff
    style RETIRE fill:#27ae60,color:#fff
```

### Code Implementation: SP Key Pair Management

```typescript
// src/sso/services/sp-key-management.service.ts
import { Injectable, Logger } from '@nestjs/common';
import * as crypto from 'crypto';
import { IdpSecretManagerService } from './idp-secret-manager.service';

export interface SpKeyPair {
  certificate: string;    // X.509 certificate (PEM)
  privateKey: string;     // Private key (PEM)
  fingerprint: string;    // SHA-256 fingerprint for identification
  createdAt: Date;
  expiresAt: Date;
}

@Injectable()
export class SpKeyManagementService {
  private readonly logger = new Logger(SpKeyManagementService.name);

  constructor(private readonly secretManager: IdpSecretManagerService) {}

  /**
   * Generates a new SP signing key pair for a provider.
   */
  async generateNewKeyPair(providerId: string): Promise<SpKeyPair> {
    // In production, use a proper X.509 certificate generation library
    // This is a simplified illustration
    const { publicKey, privateKey } = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: { type: 'spki', format: 'pem' },
      privateKeyEncoding: { type: 'pkcs8', format: 'pem' },
    });

    const fingerprint = crypto
      .createHash('sha256')
      .update(publicKey)
      .digest('hex');

    const now = new Date();
    const expiresAt = new Date(now);
    expiresAt.setFullYear(expiresAt.getFullYear() + 2); // 2-year validity

    const keyPair: SpKeyPair = {
      certificate: publicKey,
      privateKey,
      fingerprint,
      createdAt: now,
      expiresAt,
    };

    this.logger.log(`Generated new SP key pair for provider ${providerId}: ${fingerprint}`);
    return keyPair;
  }

  /**
   * Loads the active key pair for a provider (decrypted at runtime).
   */
  async getActiveKeyPair(providerId: string): Promise<SpKeyPair> {
    const config = await this.secretManager.decryptProviderConfigById(providerId);
    return config.spKeyPairs?.find((kp: SpKeyPair) => new Date(kp.expiresAt) > new Date());
  }

  /**
   * Returns all valid key pairs (for dual-certificate transition periods).
   */
  async getAllValidKeyPairs(providerId: string): Promise<SpKeyPair[]> {
    const config = await this.secretManager.decryptProviderConfigById(providerId);
    const now = new Date();
    return (config.spKeyPairs || []).filter((kp: SpKeyPair) => new Date(kp.expiresAt) > now);
  }
}
```

---

## 6. Admin Dashboard: Certificate Status Overview

Administrators need a clear view of all certificate statuses across all providers.

### Mermaid Diagram: Certificate Status Dashboard Flow

```mermaid
flowchart LR
    subgraph "Data Sources"
        JWKS["JWKS Service<br/>(OIDC key counts)"]
        SAML_META["SAML Discovery<br/>(Metadata certificates)"]
        SP_KEYS["SP Key Manager<br/>(Our signing keys)"]
        CONFIG["Provider Config<br/>(Certificate references)"]
    end

    subgraph "Aggregation Service"
        AGG["CertificateStatusService"]
        AGG --> EXPIRY["Calculate days until expiry"]
        AGG --> STATUS["Determine status:<br/>Active / ExpiringSoon / Expired"]
        AGG --> HISTORY["Track key rotation history"]
    end

    subgraph "Admin API"
        API["GET /api/admin/sso/certificates"]
        API --> RESPONSE["[{<br/>  providerCode: 'oidc.azure',<br/>  protocol: 'OIDC',<br/>  activeKeys: 2,<br/>  status: 'ACTIVE',<br/>  nextRotation: '2026-06-01'<br/>}, ...]"]
    end

    JWKS & SAML_META & SP_KEYS & CONFIG --> AGG
    AGG --> API
```

---

## 7. Emergency Key Revocation

When a key is compromised (e.g., private key leaked), immediate action is required.

### Mermaid Diagram: Emergency Key Revocation Flow

```mermaid
flowchart TD
    ALERT["🚨 Key Compromise Detected!"] --> DECIDE{"Protocol?"}

    DECIDE -->|"OIDC"| OIDC_FLOW["IdP rotates JWKS immediately<br/>(Remove compromised key)"]
    DECIDE -->|"SAML"| SAML_FLOW["Revoke certificate at IdP<br/>Update metadata"]

    OIDC_FLOW --> INVALIDATE["Invalidate our JWKS cache<br/>for this provider"]
    INVALIDATE --> RETRY["Next login will fetch<br/>fresh JWKS (without old key)"]

    SAML_FLOW --> UPDATE_DB["Update certificate in DB<br/>(via metadata refresh)"]
    UPDATE_DB --> SP_KEY["If OUR key compromised:<br/>Generate new SP key pair"]
    SP_KEY --> NOTIFY_IDP["Notify IdP admin<br/>to update trust store"]

    RETRY --> VERIFY["Logins using old tokens<br/>will fail gracefully<br/>(401 Unauthorized)"]

    style ALERT fill:#e74c3c,color:#fff
    style VERIFY fill:#27ae60,color:#fff
```

### Code Implementation: Emergency Cache Invalidation

```typescript
// src/sso/services/certificate-refresh.service.ts (addition)

  /**
   * Emergency: force-refresh all keys for a specific provider.
   * Called when a key compromise is suspected.
   */
  async emergencyKeyRefresh(providerId: string): Promise<void> {
    const provider = await this.idpProviderRepo.findById(providerId);
    if (!provider) return;

    this.logger.warn(`EMERGENCY: Force-refreshing keys for provider ${provider.providerCode}`);

    // 1. Invalidate JWKS cache
    this.jwksService.invalidateClient(provider);

    // 2. If SAML, force re-fetch metadata
    if (provider.protocolType === 'SAML2') {
      await this.refreshProviderCertificates(provider);
    }

    // 3. Log the emergency action for audit trail
    // this.eventEmitter.emit('security.emergency_key_refresh', {
    //   providerId,
    //   providerCode: provider.providerCode,
    //   triggeredBy: 'admin', // or 'system'
    //   timestamp: new Date(),
    // });
  }
```

---

## Conclusion

In Part 8, we have built a robust certificate and key management system. We handle OIDC JWKS rotation with cache invalidation and retry logic, manage SAML certificate updates via scheduled metadata polling, support dual-certificate transition periods for our own SP signing keys, and provide administrators with clear visibility into certificate health.

The final piece of the puzzle is scaling this to support multiple tenants, each with their own IdP configurations. In Part 9, we will explore **Multi-Tenant SSO**, where each customer can configure their own IdP while sharing the same application infrastructure.

<br><br><br>

---

---

## 簡介：計時炸彈

證書會過期。金鑰會被洩漏。IdPs 會定期輪換佢哋嘅簽名憑證——通常 SAML X.509 證書每 1-2 年一次，而 OIDC JWKS 就頻密好多（有啲 Provider 每 24 個鐘就輪換一次）。如果我哋個 App Cache 住一個突然失效嘅證書，所有 SSO 登入都會炒粉。對一個大企業嚟講，即係成百個用戶同時被鎖住。

喺第八集，我哋會實作 `SSO-008`：**自動化證書輪換與金鑰管理**。我哋會建立系統去自動發現新金鑰、優雅地處理金鑰過渡期（喺輪換窗口期間同時支援新舊金鑰），同埋喺證書過期之前警告管理員。

---

## 1. 證書生命週期

每一把證書同簽名金鑰都會經歷一個可預測嘅生命週期。理解呢個生命週期係管理佢嘅第一步。

### Mermaid 圖解：證書生命週期狀態機

```mermaid
stateDiagram-v2
    [*] --> Unknown: 未發現

    Unknown --> Active: 首次透過<br/>自動發現搵到

    Active --> ExpiringSoon: 有效期<br/>快到期
    Active --> Revoked: IdP 撤銷<br/>（被洩漏）

    ExpiringSoon --> Active: IdP 發佈<br/>新證書
    ExpiringSoon --> Expired: NotBefore/NotOnOrAfter<br/>過咗

    Revoked --> [*]: 由 Cache 移除

    Expired --> [*]: 由 Cache 移除

    note right of Active
        JWKS：Key 喺 Key Set 入面
        SAML：證書喺 Metadata 入面
    end note

    note right of ExpiringSoon
        警報閾值：30 日
        SAML 用，JWGS 唔適用
        （IdP 自動輪換）
    end note
```

---

## 2. OIDC JWKS 金鑰輪換

OIDC Providers 喺 `jwks_uri` endpoint 發佈佢哋嘅公鑰。呢啲金鑰有一個 `kid`（Key ID）去獨特識別每一把 Key。當一個 Provider 輪換金鑰，佢哋通常：

1. 將新金鑰加入 JWKS endpoint（新舊金鑰同時存在）
2. 開始用新金鑰簽新 Token
3. 喺 Grace period 之後移除舊金鑰

### Mermaid 圖解：OIDC JWKS 金鑰輪換流程

```mermaid
sequenceDiagram
    autonumber
    participant SP as Service Provider
    participant Cache as JWKS Cache<br/>(jwks-rsa)
    participant IdP as Identity Provider

    Note over SP,IdP: 正常運作
    SP->>Cache: 需要 kid: "key-001" 嘅公鑰
    Cache->>Cache: Cache HIT ✅
    Cache->>SP: Return 公鑰

    Note over SP,IdP: 金鑰輪換開始（IdP 加入新 Key）
    SP->>Cache: 需要 kid: "key-002" 嘅公鑰
    Cache->>Cache: Cache MISS (kid: "key-002")
    Cache->>IdP: GET /jwks
    IdP->>Cache: {"keys": [<br/>  {"kid": "key-001", ...},<br/>  {"kid": "key-002", ...}<br/>]}
    Cache->>Cache: Cache 新嘅 Key Set
    Cache->>SP: Return "key-002" 嘅公鑰

    Note over SP,IdP: 舊金鑰移除（Grace period 之後）
    SP->>Cache: 需要 kid: "key-001" 嘅公鑰
    Cache->>Cache: Cache HIT 但 Key 已經唔喺新 Set
    Cache->>IdP: GET /jwks (Refresh)
    IdP->>Cache: {"keys": [<br/>  {"kid": "key-002", ...}<br/>]}
    Cache->>SP: key-001 搵唔到 ❌
    Note over SP: 用 kid: "key-001" 簽嘅 Token 已經無效
```

### Code 實作：增強版 JWKS Service 支援輪換

```typescript
// src/sso/services/jwks.service.ts
import { Injectable, Logger, InternalServerErrorException } from '@nestjs/common';
import * as jwksClient from 'jwks-rsa';
import { IdpProvider } from '../entities/idp-provider.entity';

@Injectable()
export class JwksService {
  private readonly logger = new Logger(JwksService.name);
  private clients = new Map<string, jwksClient.JwksClient>();

  private getClient(provider: IdpProvider, jwksUri: string): jwksClient.JwksClient {
    if (!this.clients.has(provider.id)) {
      const client = jwksClient({
        jwksUri: jwksUri,
        cache: true,
        cacheMaxEntries: 10, // 輪換期間容許更多 Keys
        cacheMaxAge: 600000, // 10 分鐘——短啲可以更快發現新 Keys
        rateLimit: true,
        jwksRequestsPerMinute: 10,
        // 優雅地處理金鑰輪換
        getKeysInterceptor: (keys) => {
          this.logger.debug(`JWKS fetched: ${keys.length} keys available for ${provider.providerCode}`);
          return keys;
        },
      });
      this.clients.set(provider.id, client);
    }
    return this.clients.get(provider.id);
  }

  async getPublicKey(provider: IdpProvider, jwksUri: string, kid: string): Promise<string> {
    try {
      const client = this.getClient(provider, jwksUri);
      const key = await client.getSigningKey(kid);
      return key.getPublicKey();
    } catch (error) {
      // 如果搵唔到 kid，強制 Refresh JWKS 再試一次
      if (error.message?.includes('Unable to find a signing key')) {
        this.logger.warn(`Key ${kid} 喺 Cache 搵唔到，強制 Refresh JWKS (${provider.providerCode})`);
        this.invalidateClient(provider);
        
        try {
          const client = this.getClient(provider, jwksUri);
          const key = await client.getSigningKey(kid);
          return key.getPublicKey();
        } catch (retryError) {
          throw new InternalServerErrorException(
            `Key ${kid} Refresh 完都搵唔到。可能已被撤銷。`
          );
        }
      }
      throw error;
    }
  }

  /**
   * 強制清除某個 Provider 嘅 JWKS Cache。
   * 偵測到 Key Mismatch 嗰陣用。
   */
  invalidateClient(provider: IdpProvider): void {
    this.clients.delete(provider.id);
    this.logger.log(`JWKS Cache 已清除：${provider.providerCode}`);
  }
}
```

---

## 3. SAML X.509 證書輪換

SAML 證書輪換複雜好多，因為證書係嵌入喺 Metadata XML 入面，而且冇好似 JWKS 嘅自動輪換機制。Admin 必須手動更新 Metadata。

### Mermaid 圖解：SAML 證書輪換挑戰

```mermaid
flowchart TD
    subgraph "輪換之前"
        META_OLD["SAML Metadata（舊）<br/>包含：Certificate A"]
        STORED_OLD["我哋 DB：<br/>certificates = [Cert A]"]
        VERIFY_OLD["簽名驗證<br/>對比 Cert A ✅"]
    end

    subgraph "輪換期間（過渡期）"
        META_NEW["SAML Metadata（新）<br/>包含：Cert A + Cert B"]
        STORED_TRANS["我哋 DB：<br/>certificates = [Cert A, Cert B]"]
        VERIFY_TRANS["簽名驗證<br/>對比 Cert A 或 Cert B ✅"]
    end

    subgraph "輪換之後（舊證書移除）"
        META_FINAL["SAML Metadata（最終）<br/>包含：只有 Cert B"]
        STORED_NEW["我哋 DB：<br/>certificates = [Cert B]"]
        VERIFY_NEW["簽名驗證<br/>對比 Cert B ✅"]
    end

    META_OLD --> STORED_OLD --> VERIFY_OLD
    META_NEW --> STORED_TRANS --> VERIFY_TRANS
    META_FINAL --> STORED_NEW --> VERIFY_NEW

    style STORED_TRANS fill:#f39c12,color:#fff
```

### Code 實作：證書刷新服務

```typescript
// src/sso/services/certificate-refresh.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { SamlDiscoveryService } from './saml-discovery.service';
import { IdpProviderRepository } from '../repositories/idp-provider.repository';
import { IdpSecretManagerService } from './idp-secret-manager.service';

@Injectable()
export class CertificateRefreshService {
  private readonly logger = new Logger(CertificateRefreshService.name);

  constructor(
    private readonly samlDiscovery: SamlDiscoveryService,
    private readonly idpProviderRepo: IdpProviderRepository,
    private readonly secretManager: IdpSecretManagerService,
  ) {}

  /**
   * 每 6 個鐘 Run 一次，Check 吓有冇證書更新。
   */
  @Cron(CronExpression.EVERY_6_HOURS)
  async refreshSamlCertificates(): Promise<void> {
    this.logger.log('開始定期 SAML 證書刷新...');

    const samlProviders = await this.idpProviderRepo.findAll({
      protocolType: 'SAML2',
      isEnabled: true,
      autoDiscovery: true,
    });

    for (const provider of samlProviders) {
      try {
        await this.refreshProviderCertificates(provider);
      } catch (error) {
        this.logger.error(
          `刷新 ${provider.providerCode} 嘅證書失敗：${error.message}`
        );
        // 射個警報俾 Admin
        // this.eventEmitter.emit('certificate.refresh_failed', { providerId: provider.id });
      }
    }
  }

  private async refreshProviderCertificates(provider: IdpProvider): Promise<void> {
    // 1. 解密現有 Config 攞 Metadata URL
    const config = await this.secretManager.decryptProviderConfig(provider);
    const metadataUrl = config.metadataUrl;

    if (!metadataUrl) {
      this.logger.debug(`${provider.providerCode} 冇 metadata URL，跳過`);
      return;
    }

    // 2. Fetch 最新 Metadata
    const metadata = await this.samlDiscovery.fetchMetadata(metadataUrl);
    const newCerts = metadata.certificates;

    if (!newCerts || newCerts.length === 0) {
      this.logger.warn(`${provider.providerCode} 嘅 Metadata 入面搵唔到證書`);
      return;
    }

    // 3. 同儲存咗嘅證書對比
    const currentCerts = config.idpPublicCertificates || [];
    const certsChanged = !this.arraysEqual(currentCerts.sort(), newCerts.sort());

    if (certsChanged) {
      // 4. 用新證書更新 Config
      config.idpPublicCertificates = newCerts;
      const { encryptedBlob, wrappedDek } = await this.secretManager.encryptProviderConfig(
        provider.id, config
      );

      await this.idpProviderRepo.update(provider.id, {
        configEncrypted: encryptedBlob,
        configDekWrapped: wrappedDek,
      });

      this.logger.log(`已更新 ${provider.providerCode} 嘅證書：${newCerts.length} 把`);

      // 5. Check 吓有冇證書快到期
      this.checkCertificateExpiry(provider.providerCode, newCerts);
    }
  }

  private checkCertificateExpiry(providerCode: string, certificates: string[]): void {
    for (const cert of certificates) {
      // Parse X.509 證書睇吓幾時到期
      // Production 環境用 'node-forge' 或者 'pkijs'
      try {
        const certInfo = this.parseCertificate(cert);
        const daysUntilExpiry = Math.floor(
          (certInfo.validTo.getTime() - Date.now()) / (1000 * 60 * 60 * 24)
        );

        if (daysUntilExpiry < 30) {
          this.logger.warn(
            `${providerCode} 嘅證書 ${daysUntilExpiry} 日後到期！` +
            `到期日：${certInfo.validTo.toISOString()}`
          );
          // 射警報
          // this.eventEmitter.emit('certificate.expiring_soon', { providerCode, daysUntilExpiry });
        }
      } catch (error) {
        this.logger.debug(`解析唔到 ${providerCode} 嘅證書到期日`);
      }
    }
  }

  private arraysEqual(a: string[], b: string[]): boolean {
    return a.length === b.length && a.every((val, idx) => val === b[idx]);
  }
}
```

### Mermaid 圖解：證書刷新排程流程

```mermaid
flowchart TD
    CRON["Cron Job<br/>每 6 個鐘"] --> QUERY["Query DB：<br/>所有 Enabled 嘅 SAML Providers<br/>autoDiscovery = true"]
    QUERY --> LOOP["For each provider"]
    
    LOOP --> DECRYPT["解密 Provider Config<br/>（信封加密）"]
    DECRYPT --> URL{"有冇<br/>metadataUrl？"}
    URL -->|"冇"| SKIP["Skip 呢個 Provider"]
    URL -->|"有"| FETCH["Fetch 最新 SAML Metadata XML"]
    
    FETCH --> PARSE["Parse XML → 抽出證書"]
    PARSE --> COMPARE{"證書有冇<br/>變？"}
    
    COMPARE -->|"冇"| NEXT["去下一個 Provider"]
    COMPARE -->|"有"| UPDATE["重新加密 Config<br/>用新證書"]
    UPDATE --> SAVE["Save 落 DB"]
    SAVE --> EXPIRY["Check 證書到期日"]
    
    EXPIRY --> EXPIRING{"30 日內<br/>到期？"}
    EXPIRING -->|"係"| ALERT["⚠️ 發 Admin 警報<br/>Email / Webhook"]
    EXPIRING -->|"唔係"| NEXT

    style CRON fill:#3498db,color:#fff
    style ALERT fill:#e74c3c,color:#fff
```

---

## 4. 優雅金鑰輪換：支援多把 Key

喺過渡期間，IdP 可能用舊金鑰或者新金鑰簽 Token。我哋個 App 必須兩把都 Accept。

### Mermaid 圖解：多金鑰驗證策略

```mermaid
sequenceDiagram
    autonumber
    participant User as 用戶嘅 Browser
    participant SP as Service Provider
    participant Cache as Key Cache
    participant IdP as Identity Provider

    Note over SP,IdP: Token 1（用舊 Key 簽）
    User->>SP: OIDC Callback (token: kid=old-key)
    SP->>Cache: 攞 kid=old-key 嘅公鑰
    Cache->>SP: 舊公鑰 ✅
    SP->>SP: 用舊 Key 驗證簽名 ✅
    SP->>User: 登入成功

    Note over SP,IdP: Token 2（用新 Key 簽）
    User->>SP: OIDC Callback (token: kid=new-key)
    SP->>Cache: 攞 kid=new-key 嘅公鑰
    Cache->>Cache: Cache MISS → Fetch JWKS
    Cache->>SP: 新公鑰 ✅
    SP->>SP: 用新 Key 驗證簽名 ✅
    SP->>User: 登入成功
```

關鍵在於：靠 JWT Header 入面嘅 `kid`（Key ID），我哋永遠會搵到 *正確嘅* Key 嚟驗證。只要 JWKS endpoint 喺過渡期同時包含兩把 Key，新舊 Token 都可以無縫運作。

---

## 5. SP 證書管理（我哋自己嘅簽名金鑰）

SAML 入面，我哋仲要管自己嘅簽名證書（用嚟簽 AuthnRequests 同解密 Assertions）。佢哋都有同樣嘅過期問題。

### Mermaid 圖解：SP 證書輪換流程

```mermaid
flowchart TD
    subgraph "現狀"
        SP_CERT_A["SP Certificate A<br/>（Active，用嚟簽名）"]
        IDP_CONFIG["IdP 信任列表有<br/>Cert A"]
    end

    subgraph "Step 1：生成新證書"
        GEN["生成 SP Certificate B<br/>（IdP 仲未信）"]
    end

    subgraph "Step 2：雙證書期"
        DUAL["Cert A 同 Cert B<br/>同時存喺 Config"]
        DUAL_SIGN["用新 Cert B<br/>簽 AuthnRequests"]
        DUAL_DECRYPT["用 Cert A 或 B<br/>解密 Assertions"]
    end

    subgraph "Step 3：通知 IdP Admin"
        NOTIFY["將新證書發俾 IdP Admin<br/>加入佢哋嘅信任列表"]
    end

    subgraph "Step 4：淘汰舊證書"
        RETIRE["移除 Cert A<br/>淨係用 Cert B"]
    end

    SP_CERT_A --> GEN
    GEN --> DUAL
    DUAL --> NOTIFY
    NOTIFY --> DUAL_SIGN
    DUAL_SIGN --> DUAL_DECRYPT
    DUAL_DECRYPT --> RETIRE

    style DUAL fill:#f39c12,color:#fff
    style RETIRE fill:#27ae60,color:#fff
```

### Code 實作：SP 金鑰對管理

```typescript
// src/sso/services/sp-key-management.service.ts
import { Injectable, Logger } from '@nestjs/common';
import * as crypto from 'crypto';
import { IdpSecretManagerService } from './idp-secret-manager.service';

export interface SpKeyPair {
  certificate: string;    // X.509 證書 (PEM)
  privateKey: string;     // 私鑰 (PEM)
  fingerprint: string;    // SHA-256 指紋用嚟識別
  createdAt: Date;
  expiresAt: Date;
}

@Injectable()
export class SpKeyManagementService {
  private readonly logger = new Logger(SpKeyManagementService.name);

  constructor(private readonly secretManager: IdpSecretManagerService) {}

  /**
   * 為一個 Provider 生成新嘅 SP 簽名金鑰對。
   */
  async generateNewKeyPair(providerId: string): Promise<SpKeyPair> {
    // Production 環境用正經嘅 X.509 證書生成 Library
    // 呢度只係簡化示意
    const { publicKey, privateKey } = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: { type: 'spki', format: 'pem' },
      privateKeyEncoding: { type: 'pkcs8', format: 'pem' },
    });

    const fingerprint = crypto
      .createHash('sha256')
      .update(publicKey)
      .digest('hex');

    const now = new Date();
    const expiresAt = new Date(now);
    expiresAt.setFullYear(expiresAt.getFullYear() + 2); // 有效期 2 年

    const keyPair: SpKeyPair = {
      certificate: publicKey,
      privateKey,
      fingerprint,
      createdAt: now,
      expiresAt,
    };

    this.logger.log(`為 Provider ${providerId} 生成咗新 SP 金鑰對：${fingerprint}`);
    return keyPair;
  }

  /**
   * 載入一個 Provider 嘅 Active 金鑰對（Runtime 解密）。
   */
  async getActiveKeyPair(providerId: string): Promise<SpKeyPair> {
    const config = await this.secretManager.decryptProviderConfigById(providerId);
    return config.spKeyPairs?.find((kp: SpKeyPair) => new Date(kp.expiresAt) > new Date());
  }

  /**
   * Return 所有有效嘅金鑰對（雙證書過渡期用）。
   */
  async getAllValidKeyPairs(providerId: string): Promise<SpKeyPair[]> {
    const config = await this.secretManager.decryptProviderConfigById(providerId);
    const now = new Date();
    return (config.spKeyPairs || []).filter((kp: SpKeyPair) => new Date(kp.expiresAt) > now);
  }
}
```

---

## 6. 管理員 Dashboard：證書狀態總覽

Admin 需要一個清晰嘅視圖睇到所有 Provider 嘅證書狀態。

### Mermaid 圖解：證書狀態 Dashboard 流程

```mermaid
flowchart LR
    subgraph "數據源"
        JWKS["JWKS Service<br/>(OIDC Key 數量)"]
        SAML_META["SAML Discovery<br/>(Metadata 證書)"]
        SP_KEYS["SP Key Manager<br/>(我哋嘅簽名金鑰)"]
        CONFIG["Provider Config<br/>(證書引用)"]
    end

    subgraph "聚合服務"
        AGG["CertificateStatusService"]
        AGG --> EXPIRY["計算距離到期日幾多日"]
        AGG --> STATUS["判斷狀態：<br/>Active / ExpiringSoon / Expired"]
        AGG --> HISTORY["追蹤金鑰輪換歷史"]
    end

    subgraph "Admin API"
        API["GET /api/admin/sso/certificates"]
        API --> RESPONSE["[{<br/>  providerCode: 'oidc.azure',<br/>  protocol: 'OIDC',<br/>  activeKeys: 2,<br/>  status: 'ACTIVE',<br/>  nextRotation: '2026-06-01'<br/>}, ...]"]
    end

    JWKS & SAML_META & SP_KEYS & CONFIG --> AGG
    AGG --> API
```

---

## 7. 緊急金鑰撤銷

當一把金鑰被洩漏（例如私鑰外泄），必須立即行動。

### Mermaid 圖解：緊急金鑰撤銷流程

```mermaid
flowchart TD
    ALERT["🚨 偵測到金鑰洩漏！"] --> DECIDE{"Protocol？"}

    DECIDE -->|"OIDC"| OIDC_FLOW["IdP 即時輪換 JWKS<br/>（移除被洩漏嘅 Key）"]
    DECIDE -->|"SAML"| SAML_FLOW["喺 IdP 撤銷證書<br/>更新 Metadata"]

    OIDC_FLOW --> INVALIDATE["清除我哋嘅 JWKS Cache<br/>（呢個 Provider）"]
    INVALIDATE --> RETRY["下次登入會 Fetch<br/>全新 JWKS（冇舊 Key）"]

    SAML_FLOW --> UPDATE_DB["更新 DB 入面嘅證書<br/>（透過 Metadata Refresh）"]
    UPDATE_DB --> SP_KEY["如果係我哋嘅 Key 出事：<br/>生成新 SP 金鑰對"]
    SP_KEY --> NOTIFY_IDP["通知 IdP Admin<br/>更新信任列表"]

    RETRY --> VERIFY["用舊 Token 登入<br/>會優雅地失敗<br/>（401 Unauthorized）"]

    style ALERT fill:#e74c3c,color:#fff
    style VERIFY fill:#27ae60,color:#fff
```

### Code 實作：緊急 Cache 清除

```typescript
// src/sso/services/certificate-refresh.service.ts（新增部份）

  /**
   * 緊急：強制 Refresh 某個 Provider 嘅所有金鑰。
   * 偵測到金鑰洩漏嗰陣用。
   */
  async emergencyKeyRefresh(providerId: string): Promise<void> {
    const provider = await this.idpProviderRepo.findById(providerId);
    if (!provider) return;

    this.logger.warn(`緊急：強制 Refresh Provider ${provider.providerCode} 嘅金鑰`);

    // 1. 清除 JWKS Cache
    this.jwksService.invalidateClient(provider);

    // 2. 如果係 SAML，強制重新 Fetch Metadata
    if (provider.protocolType === 'SAML2') {
      await this.refreshProviderCertificates(provider);
    }

    // 3. 將緊急行動寫入審計日誌
    // this.eventEmitter.emit('security.emergency_key_refresh', {
    //   providerId,
    //   providerCode: provider.providerCode,
    //   triggeredBy: 'admin', // 或 'system'
    //   timestamp: new Date(),
    // });
  }
```

---

## 結語

喺第八集，我哋建立咗一個穩健嘅證書同金鑰管理系統。我哋處理 OIDC JWKS 輪換用 Cache 清除加重試邏輯，管理 SAML 證書更新靠定期 Metadata Polling，支援我哋自己 SP 簽名金鑰嘅雙證書過渡期，仲俾管理員一個清晰嘅證書健康視圖。

最後一塊拼圖就係將呢套嘢 Scale 到支援多個租戶，每個客都有自己嘅 IdP 設定。喺第九集，我哋會探討 **多租戶 SSO（Multi-Tenant SSO）**，每個客戶可以 Config 自己嘅 IdP，同時共享同一套 Application 基礎設施。

<br><br><br>
