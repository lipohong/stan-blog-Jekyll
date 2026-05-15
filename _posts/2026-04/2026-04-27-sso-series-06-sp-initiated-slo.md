---
title: "SSO Series Part 6: Single Logout (SLO) - SP-Initiated Flows | SSO 系列之六：單一登出 (SLO) - SP 發起流程"
date: 2026-04-27 18:25:00 +0800
categories: [Security, SSO Series]
tags: [sso, slo, logout, oidc, saml, redis, session-management, typescript, nodejs, assisted_by_ai]
toc: true
---

## Introduction: The Difficulty of Leaving

Logging a user in via SSO is only half the battle. In an enterprise environment, logging them out correctly is arguably more critical. If a user clicks "Logout" in your portal but their session at the Identity Provider (IdP) remains active, anyone who opens that browser can simply click "Login" and get straight back in without a password. This defeats the entire purpose of logging out.

To solve this, we must implement **Single Logout (SLO)**. SLO ensures that terminating a session in our platform (the Service Provider, or SP) also notifies the IdP to terminate the global session, and vice versa.

In Part 6, we will implement `SSO-006`: **SP-Initiated Single Logout**. We will explore how to identify an SSO-backed session, construct proper OIDC and SAML 2.0 logout requests, securely tear down our local Redis session, and clean up the reverse session index.

---

## 1. The Reverse Session Index

To support complex logout flows (especially IdP-initiated logout, which we will cover in Part 7), our backend must maintain a mapping between the IdP's session identifier and our platform's local session identifier.

When a user successfully logs in via SSO (as seen in Parts 4 and 5), we extract the `sid` (OIDC) or `SessionIndex` (SAML). We store this in Redis as a **Reverse Session Index**.

```typescript
// src/sso/services/sso-session.service.ts
import { Injectable } from '@nestjs/common';
import { Redis } from 'ioredis';

@Injectable()
export class SsoSessionTracker {
  constructor(private readonly redisClient: Redis) {}

  /**
   * Creates a reverse mapping: IdP Session ID -> Local Platform Session ID
   */
  async trackSsoSession(idpProviderId: string, idpSessionId: string, localSessionId: string, ttlSeconds: number) {
    const reverseKey = `sso:sid-map:${idpProviderId}:${idpSessionId}`;
    
    // Store the mapping with the exact same TTL as the local platform session
    await this.redisClient.set(reverseKey, `sess:${localSessionId}`, 'EX', ttlSeconds);
  }

  /**
   * Cleans up the reverse mapping. MUST be called on every SP-Initiated logout.
   */
  async untrackSsoSession(idpProviderId: string, idpSessionId: string) {
    const reverseKey = `sso:sid-map:${idpProviderId}:${idpSessionId}`;
    await this.redisClient.del(reverseKey);
  }
}
```

---

## 2. SP-Initiated Logout: The Core Service

When a user clicks "Logout" in the frontend, the frontend calls `DELETE /api/v1/auth/session`. We must determine if this session was established via SSO. If so, we need to orchestrate the IdP logout before (or while) we tear down the local session.

```typescript
// src/sso/services/sso-logout.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { SessionService } from '../../auth/services/session.service';
import { SsoSessionTracker } from './sso-session.service';
import { IdpProvider } from '../entities/idp-provider.entity';
import { SsoStrategyFactory } from './sso-strategy.factory';

@Injectable()
export class SsoLogoutService {
  private readonly logger = new Logger(SsoLogoutService.name);

  constructor(
    private readonly sessionService: SessionService,
    private readonly ssoTracker: SsoSessionTracker,
    private readonly strategyFactory: SsoStrategyFactory,
  ) {}

  async processLogout(localSessionId: string, userSsoContext?: any): Promise<string> {
    // 1. Teardown Local Session FIRST
    // Always destroy the local session immediately. If the IdP redirect fails, 
    // at least the user is logged out of our platform.
    await this.sessionService.destroySession(localSessionId);

    // If this was a standard password login, we are done. Return to local login page.
    if (!userSsoContext) {
      return '/login';
    }

    const { idpProviderId, idpSessionId, originalIdToken } = userSsoContext;

    // 2. Teardown the Reverse Session Index
    await this.ssoTracker.untrackSsoSession(idpProviderId, idpSessionId);

    // 3. Fetch Provider Config
    const providerConfig = await this.getProviderConfig(idpProviderId);
    
    // If SLO is not enabled for this provider, just return to local login page
    if (!providerConfig.sloEnabled || !providerConfig.sloUrl) {
      return '/login';
    }

    // 4. Delegate to Strategy to generate IdP Logout URL
    const strategy = this.strategyFactory.getStrategy(providerConfig.protocolType);
    
    try {
      const idpLogoutUrl = await strategy.generateLogoutUrl(
        providerConfig, 
        idpSessionId, 
        originalIdToken,
        'https://platform.com/login' // Post-logout redirect back to us
      );
      
      // Return the IdP logout URL so the frontend can redirect the browser there
      return idpLogoutUrl;
    } catch (error) {
      // If IdP logout generation fails, log it, but STILL return the user to local login.
      // We already tore down the local session in Step 1.
      this.logger.error(`Failed to generate SLO URL for provider ${providerConfig.providerCode}`, error.stack);
      return '/login?logout_warning=idp_slo_failed';
    }
  }
}
```

---

## 3. Protocol Specific Logout URL Generation

Now we implement `generateLogoutUrl` in our Strategies. OIDC and SAML handle this very differently.

### OIDC Single Logout
OIDC SLO is relatively straightforward. We redirect the browser to the IdP's `end_session_endpoint` (discovered in Part 2). We must provide the `id_token_hint` (the original ID Token) so the IdP knows *who* is logging out.

```typescript
// src/sso/strategies/oidc.strategy.ts

  async generateLogoutUrl(
    config: IdpProviderConfig, 
    idpSessionId: string, 
    idTokenHint?: string,
    postLogoutRedirectUri?: string
  ): Promise<string> {
    
    const endSessionEndpoint = config.sloUrl; // from Auto-Discovery
    if (!endSessionEndpoint) return null;

    const url = new URL(endSessionEndpoint);
    
    // Provide the original ID Token as a hint
    if (idTokenHint) {
      url.searchParams.append('id_token_hint', idTokenHint);
    }
    
    // Tell the IdP where to send the user back after logout
    if (postLogoutRedirectUri) {
      url.searchParams.append('post_logout_redirect_uri', postLogoutRedirectUri);
    }

    // Optional: Add state to prevent CSRF on the logout callback
    url.searchParams.append('state', crypto.randomBytes(16).toString('hex'));

    return url.toString();
  }
```

### SAML 2.0 Single Logout
SAML SLO is much more complex. We must generate a `<samlp:LogoutRequest>` XML document, embed the `NameID` and `SessionIndex`, encode it via DEFLATE compression, Base64 it, and append it to the URL (HTTP-Redirect binding). If configured, we must also sign the query string.

```typescript
// src/sso/strategies/saml.strategy.ts

  async generateLogoutUrl(
    config: IdpProviderConfig, 
    idpSessionId: string, 
    idTokenHint?: string, // Not used in SAML
    postLogoutRedirectUri?: string // SAML uses RelayState or implicit routing
  ): Promise<string> {
    
    const samlClient = this.buildSamlClient(config); // From Part 5
    
    // We must rebuild a mock User object that node-saml expects for logout
    const logoutContext = {
      nameID: idpSessionId, // Or the actual NameID if different from SessionIndex
      sessionIndex: idpSessionId
    };

    try {
      // node-saml generates the LogoutRequest XML, deflates it, and generates the URL
      const logoutUrl = await samlClient.getLogoutUrlAsync(logoutContext, postLogoutRedirectUri, {});
      return logoutUrl;
    } catch (error) {
      throw new InternalServerErrorException('Failed to generate SAML LogoutRequest');
    }
  }
```

## Conclusion

In Part 6, we have solved the "Difficulty of Leaving". We established a robust Reverse Session Index in Redis. We implemented a fail-safe logout service that guarantees the local session is destroyed *before* attempting to communicate with the IdP. Finally, we implemented OIDC's `end_session_endpoint` and SAML's `<LogoutRequest>` generation.

However, SP-Initiated logout assumes the user actively clicks "Logout" in *our* application. What happens if an IT Admin clicks "Revoke Sessions" in the Entra ID or Okta portal? How does our application know to terminate the session immediately? 

In Part 7, we will explore the crown jewel of enterprise session management: **IdP-Initiated Back-Channel Logout (BCL)**.

<br><br><br>

---
---

## 簡介：離場的難題 (The Difficulty of Leaving)

透過 SSO 幫 User 登入只係贏咗上半場。喺一個企業級環境入面，點樣正確咁幫佢哋登出 (Logout) 其實更加緊要。如果一個 User 喺你個 Portal 度撳咗「登出」，但佢喺 Identity Provider (IdP) 嗰邊個 Session 仲係 Active，咁下一個用呢部機開 Browser 嘅人，只要輕輕撳吓「登入」，就可以唔使密碼直闖入去。咁樣完全抹殺咗「登出」嘅意義。

為咗解決呢個問題，我哋必須要實作 **單一登出 (Single Logout, SLO)**。SLO 確保當我哋平台 (Service Provider, SP) 終止 Session 嗰陣，會同時通知 IdP 去終止個 Global session，反之亦然。

喺第六集，我哋會實作 `SSO-006`: **SP 發起嘅單一登出 (SP-Initiated Single Logout)**。我哋會探討點樣識別一個由 SSO 建立嘅 Session、點樣 Generate 正確嘅 OIDC 同 SAML 2.0 Logout requests、點樣安全咁炸毀我哋 Local 嘅 Redis session，同埋點樣清理個反向 Session 索引 (Reverse Session Index)。

---

## 1. 反向會話索引 (The Reverse Session Index)

為咗支援極度複雜嘅 Logout flows (尤其係我哋會喺第 7 集講嘅 IdP-Initiated logout)，我哋個 Backend 必須要 Maintain 一個 Mapping，將 IdP 嗰邊嘅 Session ID 綁死落我哋平台嘅 Local Session ID 度。

當一個 User 成功透過 SSO 登入 (睇返第 4 同 5 集)，我哋會抽出個 `sid` (OIDC) 或者 `SessionIndex` (SAML)。我哋會將呢個值 Save 落 Redis 做一個 **Reverse Session Index**。

```typescript
// src/sso/services/sso-session.service.ts
import { Injectable } from '@nestjs/common';
import { Redis } from 'ioredis';

@Injectable()
export class SsoSessionTracker {
  constructor(private readonly redisClient: Redis) {}

  /**
   * 建立反向 Mapping: IdP Session ID -> Local Platform Session ID
   */
  async trackSsoSession(idpProviderId: string, idpSessionId: string, localSessionId: string, ttlSeconds: number) {
    const reverseKey = `sso:sid-map:${idpProviderId}:${idpSessionId}`;
    
    // 將個 Mapping Save 落 Redis，TTL 必須同 Local platform session 嘅 TTL 完全一致
    await this.redisClient.set(reverseKey, `sess:${localSessionId}`, 'EX', ttlSeconds);
  }

  /**
   * 清理反向 Mapping。每一次做 SP-Initiated logout 都 必須 Call 呢條 Function。
   */
  async untrackSsoSession(idpProviderId: string, idpSessionId: string) {
    const reverseKey = `sso:sid-map:${idpProviderId}:${idpSessionId}`;
    await this.redisClient.del(reverseKey);
  }
}
```

---

## 2. SP 發起登出：核心服務

當 User 喺 Frontend 撳「登出」，Frontend 會 Call `DELETE /api/v1/auth/session`。我哋必須判斷呢個 Session 係咪由 SSO 建立嘅。如果係，我哋就要喺炸毀 Local session 之前 (或者同時)，安排埋 IdP 嗰邊嘅 Logout。

```typescript
// src/sso/services/sso-logout.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { SessionService } from '../../auth/services/session.service';
import { SsoSessionTracker } from './sso-session.service';
import { IdpProvider } from '../entities/idp-provider.entity';
import { SsoStrategyFactory } from './sso-strategy.factory';

@Injectable()
export class SsoLogoutService {
  private readonly logger = new Logger(SsoLogoutService.name);

  constructor(
    private readonly sessionService: SessionService,
    private readonly ssoTracker: SsoSessionTracker,
    private readonly strategyFactory: SsoStrategyFactory,
  ) {}

  async processLogout(localSessionId: string, userSsoContext?: any): Promise<string> {
    // 1. 優先炸毀 Local Session！
    // 無論如何，第一時間炸毀 Local session。就算之後 Redirect 去 IdP 炒粉，
    // 個 User 喺我哋個平台都起碼真係 Logout 咗。
    await this.sessionService.destroySession(localSessionId);

    // 如果呢個只係一個普通嘅 Password login，咁就搞掂，直接 Return 去 Local login page。
    if (!userSsoContext) {
      return '/login';
    }

    const { idpProviderId, idpSessionId, originalIdToken } = userSsoContext;

    // 2. 炸毀埋 Reverse Session Index
    await this.ssoTracker.untrackSsoSession(idpProviderId, idpSessionId);

    // 3. 攞 Provider 嘅 Config
    const providerConfig = await this.getProviderConfig(idpProviderId);
    
    // 如果呢個 Provider 冇開 SLO，或者冇填 SLO URL，咁就直接返 Local login page。
    if (!providerConfig.sloEnabled || !providerConfig.sloUrl) {
      return '/login';
    }

    // 4. 將 Generate IdP Logout URL 嘅責任交俾 Strategy
    const strategy = this.strategyFactory.getStrategy(providerConfig.protocolType);
    
    try {
      const idpLogoutUrl = await strategy.generateLogoutUrl(
        providerConfig, 
        idpSessionId, 
        originalIdToken,
        'https://platform.com/login' // Logout 完之後叫 IdP 踢返個 User 返嚟我哋度
      );
      
      // Return 條 IdP logout URL 俾 Frontend，等 Frontend 自己 Redirect 個 Browser 過去
      return idpLogoutUrl;
    } catch (error) {
      // 如果 Generate IdP logout 炒粉，Mark 低條 Log，但 **照樣** 叫 User 返 Local login。
      // 因為我哋喺 Step 1 已經炸爛咗個 Local session，好安全。
      this.logger.error(`為 Provider ${providerConfig.providerCode} 產生 SLO URL 失敗`, error.stack);
      return '/login?logout_warning=idp_slo_failed';
    }
  }
}
```

---

## 3. 協定專屬嘅 Logout URL 產生器

依家我哋喺 Strategies 入面實作 `generateLogoutUrl`。OIDC 同 SAML 處理呢家嘢嘅手法係完全唔同嘅。

### OIDC 單一登出 (Single Logout)
OIDC 嘅 SLO 相對直接。我哋將 Browser Redirect 去 IdP 嘅 `end_session_endpoint` (喺第二集 Auto-Discovery 搵到嗰條)。我哋必須要交出 `id_token_hint` (即係原本登入嗰陣個 ID Token)，等 IdP 知道到底係 *邊個* 想 Logout。

```typescript
// src/sso/strategies/oidc.strategy.ts

  async generateLogoutUrl(
    config: IdpProviderConfig, 
    idpSessionId: string, 
    idTokenHint?: string,
    postLogoutRedirectUri?: string
  ): Promise<string> {
    
    const endSessionEndpoint = config.sloUrl; // 由 Auto-Discovery 攞返嚟嘅
    if (!endSessionEndpoint) return null;

    const url = new URL(endSessionEndpoint);
    
    // 提供原本嘅 ID Token 作為線索 (Hint)
    if (idTokenHint) {
      url.searchParams.append('id_token_hint', idTokenHint);
    }
    
    // 講明俾 IdP 聽 Logout 完之後要掟個 User 去邊
    if (postLogoutRedirectUri) {
      url.searchParams.append('post_logout_redirect_uri', postLogoutRedirectUri);
    }

    // Optional: 加個 state 去防禦 Logout callback 嘅 CSRF
    url.searchParams.append('state', crypto.randomBytes(16).toString('hex'));

    return url.toString();
  }
```

### SAML 2.0 單一登出 (Single Logout)
SAML 嘅 SLO 複雜好多倍。我哋要 Generate 一份 `<samlp:LogoutRequest>` XML document、塞個 `NameID` 同 `SessionIndex` 入去、用 DEFLATE 壓縮、轉 Base64、最後 Append 喺條 URL 後面 (HTTP-Redirect binding)。如果 Config 有要求，我哋仲要用私鑰去 Sign 條 Query string。

```typescript
// src/sso/strategies/saml.strategy.ts

  async generateLogoutUrl(
    config: IdpProviderConfig, 
    idpSessionId: string, 
    idTokenHint?: string, // SAML 唔玩呢套
    postLogoutRedirectUri?: string // SAML 用 RelayState 或者預先 Config 好嘅 Routing
  ): Promise<string> {
    
    const samlClient = this.buildSamlClient(config); // 第五集寫落嗰個
    
    // 我哋要重新砌一個 node-saml 期望收到嘅 Logout Mock User Object
    const logoutContext = {
      nameID: idpSessionId, // 或者如果 NameID 同 SessionIndex 唔同，就要入真嗰個
      sessionIndex: idpSessionId
    };

    try {
      // node-saml 會自動 Generate LogoutRequest XML、壓縮，再砌條 URL 出嚟
      const logoutUrl = await samlClient.getLogoutUrlAsync(logoutContext, postLogoutRedirectUri, {});
      return logoutUrl;
    } catch (error) {
      throw new InternalServerErrorException('無法產生 SAML LogoutRequest');
    }
  }
```

## 結語

喺第六集，我哋成功拆解咗「離場的難題」。我哋喺 Redis 建立咗一個強大嘅 Reverse Session Index。我哋寫咗一個 Fail-safe 嘅 Logout 服務，確保喺同 IdP 溝通之前，Local session 一定已經被徹底炸毀。最後，我哋實作咗 OIDC 嘅 `end_session_endpoint` 同埋 SAML 嘅 `<LogoutRequest>` 產生邏輯。

不過，SP-Initiated logout 嘅前提係個 User 會好乖咁喺 *我哋* 嘅 Application 入面撳「登出」。萬一有個 IT Admin 喺 Entra ID 或者 Okta 個後台直接撳「Revoke Sessions (撤銷會話)」咁點算？我哋個 App 點樣會知要即刻踢個 User 出門口？

喺第七集，我哋會探索企業級 Session 管理嘅皇冠寶石：**IdP 發起嘅後台登出 (IdP-Initiated Back-Channel Logout, BCL)**。

<br><br><br>
