---
title: "SSO Series Part 4: OIDC Callback Processing & ID Token Verification | SSO 系列之四：OIDC 回調處理與 ID Token 驗證"
date: 2026-04-27 18:15:00 +0800
categories: [Security, SSO Series]
tags: [sso, oidc, jwt, jwks, security, typescript, nodejs, assisted_by_ai]
toc: true
---

## Introduction: Trust, but Verify

In the previous parts, we set up the IdP configuration, designed the attribute mapping engine, and built the enforced user-matching flow. Now, it is time to look at the heart of the OpenID Connect (OIDC) authentication process: **The Callback Processing**.

When the Identity Provider (IdP) redirects the user back to our application, they hand us an **ID Token**. This token is essentially a JSON Web Token (JWT) that asserts the identity of the user. But how do we know the IdP actually issued this token? How do we know it wasn't intercepted, altered, or replayed by a malicious actor?

In Part 4, we will implement `SSO-003`. We will build a robust OIDC strategy that fetches the IdP's JSON Web Key Set (JWKS), mathematically verifies the JWT signature, defends against replay attacks using nonces, and securely extracts the claims needed for our mapping engine.

---

## 1. The ID Token: A Cryptographic Assertion

An OIDC ID Token is a signed JWT. It consists of three parts:
1. **Header**: Contains the algorithm (usually `RS256`) and the Key ID (`kid`) used to sign it.
2. **Payload**: Contains the claims (e.g., `iss`, `sub`, `aud`, `exp`, `iat`, `nonce`).
3. **Signature**: The cryptographic proof of authenticity.

### Security Rule: Never Trust the Payload Before Signature Verification
Many developers make the catastrophic mistake of decoding the payload using `jwt.decode()`, reading the `email`, and logging the user in. **This completely bypasses security.** You must always use `jwt.verify()` with the IdP's public key.

---

## 2. JWKS: Fetching the Public Keys

To verify the signature, we need the public key corresponding to the private key the IdP used to sign the token. We get this from the IdP's JWKS endpoint (which we auto-discovered in Part 2).

Since fetching the JWKS on every single login would be terribly slow, we must cache it using a library like `jwks-rsa`.

```typescript
// src/sso/services/jwks.service.ts
import { Injectable, InternalServerErrorException } from '@nestjs/common';
import * as jwksClient from 'jwks-rsa';
import { IdpProvider } from '../entities/idp-provider.entity';

@Injectable()
export class JwksService {
  private clients = new Map<string, jwksClient.JwksClient>();

  /**
   * Retrieves or initializes a cached JWKS client for a specific IdP.
   */
  private getClient(provider: IdpProvider, jwksUri: string): jwksClient.JwksClient {
    if (!this.clients.has(provider.id)) {
      const client = jwksClient({
        jwksUri: jwksUri,
        cache: true,
        cacheMaxEntries: 5, // Usually an IdP only has 1 or 2 active keys
        cacheMaxAge: 36000000, // Cache for 10 hours
        rateLimit: true, // Prevent flooding the IdP if something goes wrong
        jwksRequestsPerMinute: 10,
      });
      this.clients.set(provider.id, client);
    }
    return this.clients.get(provider.id);
  }

  /**
   * Fetches the specific public key required to verify a JWT signature based on the 'kid' header.
   */
  async getPublicKey(provider: IdpProvider, jwksUri: string, kid: string): Promise<string> {
    try {
      const client = this.getClient(provider, jwksUri);
      const key = await client.getSigningKey(kid);
      return key.getPublicKey();
    } catch (error) {
      throw new InternalServerErrorException(`Failed to retrieve public key (kid: ${kid}) from JWKS endpoint.`);
    }
  }
}
```

---

## 3. The OIDC Strategy: Validation Pipeline

Now we assemble the `OidcStrategy` (from our Strategy Pattern in Part 1) to process the callback, verify the token, and protect against replay attacks.

### The Callback Processor

```typescript
// src/sso/strategies/oidc.strategy.ts
import { Injectable, UnauthorizedException, BadRequestException } from '@nestjs/common';
import * as jwt from 'jsonwebtoken';
import { ISsoProtocolStrategy } from '../interfaces/sso-strategy.interface';
import { IdpProviderConfig } from '../entities/idp-provider.entity';
import { JwksService } from '../services/jwks.service';
import { SsoStateContext } from '../services/sso-state.service';
import { OidcDiscoveryService } from '../services/oidc-discovery.service';

@Injectable()
export class OidcStrategy implements ISsoProtocolStrategy {
  constructor(
    private readonly jwksService: JwksService,
    private readonly discoveryService: OidcDiscoveryService,
  ) {}

  async processCallback(
    providerConfig: IdpProviderConfig, 
    payload: any, 
    storedState: SsoStateContext
  ): Promise<any> {
    
    // 1. Extract the ID Token (Assuming Implicit Flow or after Code Exchange)
    const idToken = payload.id_token;
    if (!idToken) {
      throw new BadRequestException('No id_token found in the OIDC callback.');
    }

    // 2. Decode the header to find the Key ID (kid) and Algorithm
    const decodedHeader = jwt.decode(idToken, { complete: true });
    if (!decodedHeader || !decodedHeader.header || !decodedHeader.header.kid) {
      throw new UnauthorizedException('Invalid ID Token format. Missing header or kid.');
    }

    // 3. Fetch IdP Metadata (Usually cached)
    const metadata = await this.discoveryService.fetchMetadata(providerConfig.issuerUrl);

    // 4. Get the Public Key
    const publicKey = await this.jwksService.getPublicKey(
      providerConfig, 
      metadata.jwks_uri, 
      decodedHeader.header.kid
    );

    // 5. Verify the Signature and Standard Claims
    let verifiedPayload: any;
    try {
      verifiedPayload = jwt.verify(idToken, publicKey, {
        algorithms: ['RS256'], // Enforce strong algorithms
        issuer: metadata.issuer,
        audience: providerConfig.clientId, // Prevent Cross-Client attacks
        maxAge: '5m', // Token must have been issued recently
        clockTolerance: 60, // Allow 60 seconds of clock drift
      });
    } catch (error) {
      throw new UnauthorizedException(`ID Token verification failed: ${error.message}`);
    }

    // 6. Nonce Validation (Replay Protection)
    if (storedState.nonce) {
      if (verifiedPayload.nonce !== storedState.nonce) {
        throw new UnauthorizedException('Nonce mismatch. Potential replay attack detected.');
      }
    }

    // 7. Track Session Identifier for Back-Channel Logout (Spec Part 3.4)
    if (verifiedPayload.sid) {
      // Store 'sid' in our local session context for later use in Back-Channel Logout
      storedState.idpSessionId = verifiedPayload.sid;
    } else {
      // Fallback to 'sub' if 'sid' is missing
      storedState.idpSessionId = verifiedPayload.sub;
    }

    return verifiedPayload;
  }
}
```

### Explanation of Validations

1. **Algorithm Checking (`RS256`)**: We strictly enforce that the token is signed with RSA. An attacker might try to change the header to `alg: "none"` or use `HS256` (HMAC) with a public key to bypass validation. `jwt.verify` must explicitly restrict allowed algorithms.
2. **Issuer (`iss`)**: Ensures the token was actually issued by the IdP we expect (e.g., `https://sts.windows.net/tenant-id/`), preventing token injection from a different IdP.
3. **Audience (`aud`)**: Ensures the token was generated *for our application* (`clientId`). This prevents an attacker from taking a valid token generated for a *different* application (like a mobile app) and using it to log into our web portal.
4. **Nonce (`nonce`)**: The `nonce` we generated during initiation (stored in Redis) is checked against the `nonce` inside the signed JWT. If an attacker intercepts the callback URL and tries to replay it, the `state` will be gone from Redis, and the `nonce` validation will fail.

---

## 4. Fallback: The UserInfo Service

In the Authorization Code flow, the ID Token might be kept deliberately small to save bandwidth. It might only contain the `sub` claim. To get the rich profile data (`email`, `name`, `department`), we must make a server-to-server call to the IdP's UserInfo endpoint using the Access Token we obtained during the code exchange.

```typescript
import axios from 'axios';

// Inside processCallback (after Code Exchange and ID Token validation)
async fetchUserInfo(accessToken: string, userInfoEndpoint: string): Promise<any> {
  try {
    const response = await axios.get(userInfoEndpoint, {
      headers: {
        Authorization: `Bearer ${accessToken}`,
      },
    });
    return response.data;
  } catch (error) {
    throw new UnauthorizedException('Failed to retrieve user profile from UserInfo endpoint.');
  }
}

// Merge the payloads
const finalPayload = {
  ...verifiedIdTokenPayload,
  ...userInfoPayload,
};
```

This `finalPayload` is exactly what gets passed into the `AttributeMapperService` we built in Part 2!

## Conclusion

In Part 4, we have locked down the OIDC authentication layer. By implementing strict JWKS caching, enforcing algorithm restrictions, validating the Issuer and Audience, and leveraging Nonces, we have created an impenetrable fortress against token forgery and replay attacks. We also prepared for enterprise session termination by extracting the `sid` claim.

In Part 5, we will dive into the legacy giant: **SAML 2.0 Integration & Assertion Processing**. We will deal with XML canonicalization, signing certificates, and defeating XML Signature Wrapping (XSW) attacks.

<br><br><br>

---
---

## 簡介：信任，但必須驗證 (Trust, but Verify)

喺之前幾集，我哋搞掂咗 IdP 嘅設定、設計咗屬性映射引擎 (Attribute Mapping Engine)，同埋起好咗強制 (ENFORCED) 嘅用戶匹配 Flow。依家，係時候直搗 OpenID Connect (OIDC) 認證過程嘅心臟地帶：**回調處理 (Callback Processing)**。

當 Identity Provider (IdP) 將個 User Redirect 返嚟我哋個 Application 嗰陣，佢哋會交出一個 **ID Token**。呢個 Token 本質上係一個 JSON Web Token (JWT)，用嚟證明個 User 嘅身份。但係我哋點知呢個 Token 真係由嗰個 IdP 發出嚟？我哋點知佢冇俾黑客中途攔截、竄改，或者重放 (Replay)？

喺第四集，我哋會實作 `SSO-003`。我哋會寫一個堅如磐石嘅 OIDC Strategy，佢識得去攞 IdP 嘅 JSON Web Key Set (JWKS)、用數學方法驗證 JWT 嘅簽名、利用 Nonces 去防禦重放攻擊，同埋安全咁抽出 Mapping engine 需要嘅 Claims。

---

## 1. ID Token：一份密碼學聲明

一個 OIDC ID Token 係一個簽咗名嘅 JWT。佢分三個部份：
1. **Header (標頭)**: 寫明用咩 Algorithm (通常係 `RS256`) 同埋用咗邊條 Key ID (`kid`) 去簽。
2. **Payload (載荷)**: 裝住啲 Claims (例如 `iss`, `sub`, `aud`, `exp`, `iat`, `nonce`)。
3. **Signature (簽名)**: 真實性嘅密碼學鐵證。

### 保安守則：未驗證簽名之前，絕對唔好信 Payload 裡面嘅嘢！
好多 Developer 犯下一個彌天大錯：就咁用 `jwt.decode()` 解開個 Payload，見到個 `email` 就信到十足，直接俾個 User 登入。**咁樣係完全 Bypass 晒所有保安！** 你必須、一定、無論如何都要用 `jwt.verify()` 配合 IdP 嘅 Public key 去驗證。

---

## 2. JWKS：獲取公鑰 (Public Keys)

要驗證個 Signature，我哋需要 IdP 簽 Token 嗰條 Private key 對應嘅 Public key。我哋要喺 IdP 嘅 JWKS endpoint (即係我哋喺第二集 Auto-discover 嗰條 URL) 攞。

因為如果每次有人 Login 我哋都去 Fetch 一次 JWKS 嘅話，個 Server 一定慢到飛起，所以我哋必須用好似 `jwks-rsa` 呢啲 Library 去 Cache 住佢。

```typescript
// src/sso/services/jwks.service.ts
import { Injectable, InternalServerErrorException } from '@nestjs/common';
import * as jwksClient from 'jwks-rsa';
import { IdpProvider } from '../entities/idp-provider.entity';

@Injectable()
export class JwksService {
  private clients = new Map<string, jwksClient.JwksClient>();

  /**
   * 攞返或者 Initialize 一個專屬於某個 IdP 嘅 JWKS Client (連埋 Cache 機制)。
   */
  private getClient(provider: IdpProvider, jwksUri: string): jwksClient.JwksClient {
    if (!this.clients.has(provider.id)) {
      const client = jwksClient({
        jwksUri: jwksUri,
        cache: true,
        cacheMaxEntries: 5, // 通常一個 IdP 同一時間得 1-2 條 Active keys
        cacheMaxAge: 36000000, // Cache 10 個鐘
        rateLimit: true, // 萬一出事都唔好狂 DDoS 人哋個 IdP
        jwksRequestsPerMinute: 10,
      });
      this.clients.set(provider.id, client);
    }
    return this.clients.get(provider.id);
  }

  /**
   * 根據 JWT header 入面嘅 'kid'，去 JWKS endpoint 抽嗰條特定嘅 Public key 出嚟驗證。
   */
  async getPublicKey(provider: IdpProvider, jwksUri: string, kid: string): Promise<string> {
    try {
      const client = this.getClient(provider, jwksUri);
      const key = await client.getSigningKey(kid);
      return key.getPublicKey();
    } catch (error) {
      throw new InternalServerErrorException(`無法喺 JWKS endpoint 搵到對應嘅 Public key (kid: ${kid})。`);
    }
  }
}
```

---

## 3. OIDC 策略：驗證流水線 (Validation Pipeline)

依家我哋將個 `OidcStrategy` (第一集講嘅 Strategy Pattern) 砌埋一齊，去處理 Callback、驗證 Token，同埋擋住啲 Replay attacks。

### Callback 處理器

```typescript
// src/sso/strategies/oidc.strategy.ts
import { Injectable, UnauthorizedException, BadRequestException } from '@nestjs/common';
import * as jwt from 'jsonwebtoken';
import { ISsoProtocolStrategy } from '../interfaces/sso-strategy.interface';
import { IdpProviderConfig } from '../entities/idp-provider.entity';
import { JwksService } from '../services/jwks.service';
import { SsoStateContext } from '../services/sso-state.service';
import { OidcDiscoveryService } from '../services/oidc-discovery.service';

@Injectable()
export class OidcStrategy implements ISsoProtocolStrategy {
  constructor(
    private readonly jwksService: JwksService,
    private readonly discoveryService: OidcDiscoveryService,
  ) {}

  async processCallback(
    providerConfig: IdpProviderConfig, 
    payload: any, 
    storedState: SsoStateContext
  ): Promise<any> {
    
    // 1. 抽出 ID Token (假設係 Implicit Flow 或者已經用 Code 換咗 Token)
    const idToken = payload.id_token;
    if (!idToken) {
      throw new BadRequestException('OIDC callback 入面搵唔到 id_token。');
    }

    // 2. Decode 個 Header 睇吓用咗咩 Algorithm 同 Key ID (kid)
    const decodedHeader = jwt.decode(idToken, { complete: true });
    if (!decodedHeader || !decodedHeader.header || !decodedHeader.header.kid) {
      throw new UnauthorizedException('ID Token 格式有錯。漏咗 Header 或者 kid。');
    }

    // 3. 攞 IdP 嘅 Metadata (通常已經 Cache 咗)
    const metadata = await this.discoveryService.fetchMetadata(providerConfig.issuerUrl);

    // 4. 攞條 Public Key 返嚟
    const publicKey = await this.jwksService.getPublicKey(
      providerConfig, 
      metadata.jwks_uri, 
      decodedHeader.header.kid
    );

    // 5. 驗證簽名同埋標準 Claims
    let verifiedPayload: any;
    try {
      verifiedPayload = jwt.verify(idToken, publicKey, {
        algorithms: ['RS256'], // 強制只准用強力 Algorithm
        issuer: metadata.issuer,
        audience: providerConfig.clientId, // 防禦 Cross-Client 攻擊
        maxAge: '5m', // 確保 Token 係啱啱新鮮出爐嘅
        clockTolerance: 60, // 容忍 60 秒嘅時鐘漂移
      });
    } catch (error) {
      throw new UnauthorizedException(`ID Token 驗證失敗: ${error.message}`);
    }

    // 6. Nonce 驗證 (防禦重放攻擊 Replay Protection)
    if (storedState.nonce) {
      if (verifiedPayload.nonce !== storedState.nonce) {
        throw new UnauthorizedException('Nonce 唔 match。懷疑受到重放攻擊。');
      }
    }

    // 7. 記錄 Session 標識符，留返俾之後嘅 Back-Channel Logout 用 (Spec Part 3.4)
    if (verifiedPayload.sid) {
      // 將 'sid' Save 喺我哋 Local session 入面
      storedState.idpSessionId = verifiedPayload.sid;
    } else {
      // 如果冇 'sid'，就唯有用 'sub' 頂住先
      storedState.idpSessionId = verifiedPayload.sub;
    }

    return verifiedPayload;
  }
}
```

### 驗證規則拆解

1. **鎖死 Algorithm (`RS256`)**: 我哋嚴格限制 Token 必須用 RSA 簽名。黑客可能會嘗試將 Header 改做 `alg: "none"`，或者用 `HS256` (HMAC) 配搭 Public key 嚟呃個 Validator。`jwt.verify` 必須寫明只准用邊幾隻 Algorithm。
2. **Issuer 驗證 (`iss`)**: 確保個 Token 真係由我哋預期嗰個 IdP (例如 `https://sts.windows.net/tenant-id/`) 發出，防止其他 IdP 嘅 Token 混水摸魚。
3. **Audience 驗證 (`aud`)**: 確保個 Token 係 *專登派俾我哋個 App* (`clientId`) 嘅。咁做可以防止黑客攞住一個派俾「其他 App (例如 Mobile app)」嘅 Valid Token，用嚟強行 Login 我哋個 Web portal。
4. **Nonce 驗證 (`nonce`)**: 我哋喺 Initiate 登入嗰陣 Generate 兼 Save 喺 Redis 嘅 `nonce`，必須要同 signed JWT 入面嗰個 `nonce` 一模一樣。如果黑客攔截咗條 Callback URL 嘗試 Replay，Redis 入面個 `state` 已經俾我哋 Delete 咗，個 `nonce` 驗證就一定會炒粉。

---

## 4. 後備方案：UserInfo 服務

喺 Authorization Code flow 入面，有時 IdP 為了慳 Bandwidth，會特登整到個 ID Token 鬼死咁細，入面得個 `sub` claim。要攞齊詳細嘅 Profile data (`email`, `name`, `department`)，我哋就要用 Code exchange 換返嚟嗰個 Access Token，發起一個 Server-to-server 嘅 Request 去 IdP 嘅 UserInfo endpoint。

```typescript
import axios from 'axios';

// 喺 processCallback 入面 (完成 Code Exchange 同 ID Token 驗證之後)
async fetchUserInfo(accessToken: string, userInfoEndpoint: string): Promise<any> {
  try {
    const response = await axios.get(userInfoEndpoint, {
      headers: {
        Authorization: `Bearer ${accessToken}`,
      },
    });
    return response.data;
  } catch (error) {
    throw new UnauthorizedException('無法由 UserInfo endpoint 獲取用戶資料。');
  }
}

// 將兩邊嘅 Payload 溝埋一齊
const finalPayload = {
  ...verifiedIdTokenPayload,
  ...userInfoPayload,
};
```

呢個溝埋一齊嘅 `finalPayload`，就係會原封不動咁掟入去我哋喺第二集寫好嗰個 `AttributeMapperService` 度啦！

## 結語

喺第四集，我哋將 OIDC 嘅認證層鎖到實一實。透過實作嚴格嘅 JWKS Caching、限制 Algorithm、驗證 Issuer 同 Audience，以及運用 Nonces，我哋築起咗一道防禦 Token 偽造同埋重放攻擊嘅鋼鐵防線。我哋亦為企業級嘅 Session 終止 (Back-Channel Logout) 做好咗準備，成功抽出咗 `sid` Claim。

喺第五集，我哋將會潛入上古巨獸嘅領域：**SAML 2.0 整合與 Assertion 處理**。我哋會面對 XML Canonicalization (規範化)、Signing Certificates，同埋點樣擊退 XML Signature Wrapping (XSW) 攻擊。

<br><br><br>
