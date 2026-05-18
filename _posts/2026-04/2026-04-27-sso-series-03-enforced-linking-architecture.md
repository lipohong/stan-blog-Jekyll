---
title: "SSO Series Part 3: User Matching and The ENFORCED Linking Architecture | SSO 系列之三：用戶匹配與強制連結架構"
date: 2026-04-27 18:10:00 +0800
categories: [Security, SSO Series]
tags: [sso, architecture, account-linking, security, 2fa, typescript, nodejs, assisted_by_ai]
toc: true
---

## Introduction: Where Identity Meets the Database

In Part 1, we secured the protocol exchanges. In Part 2, we built an engine to map and normalize the messy data from Identity Providers (IdPs) into clean `SsoUserClaims`. Now, in Part 3, we must answer the most critical question in any SSO implementation: **Who is this person?**

Identity resolution is fraught with security perils. If we match a user incorrectly, we grant an attacker full access to someone else's account (Account Takeover / ATO). Furthermore, enterprise customers often require strict enforcement: once a user links an SSO account, they should *never* be allowed to log in with a local password again. 

Today, we will implement `SSO-005` from our specification: The User Matching and ENFORCED Account Linking Architecture. We will handle secure user lookups, dynamic account linking, blocking local passwords, and bypassing TOTP (2FA) for trusted federated identities.

---

## 1. The SSO Profile Entity

A local `User` might have multiple external identities (e.g., an Entra ID account and a GitHub account). Therefore, we cannot simply add an `sso_id` column to the `users` table. We need a one-to-many relationship mapping external identities to internal users.

### Clean Architecture: The SSO Profile Model

```typescript
// src/sso/entities/user-sso-profile.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, UpdateDateColumn, ManyToOne, JoinColumn, Index } from 'typeorm';
import { User } from '../../users/entities/user.entity';
import { IdpProvider } from './idp-provider.entity';

@Entity('user_sso_profiles')
// Critical: Ensure a user cannot link the same external ID from the same provider twice
@Index(['idpProvider', 'extUserId'], { unique: true })
export class UserSsoProfile {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => User, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'user_id' })
  user: User;

  @ManyToOne(() => IdpProvider, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'idp_provider_id' })
  idpProvider: IdpProvider;

  @Column({ length: 255 })
  extUserId: string; // e.g., the 'sub' or 'NameID' from the IdP

  @Column({ length: 255, nullable: true })
  extEmail: string;

  @Column({ length: 255, nullable: true })
  extDisplayName: string;

  @Column({ type: 'timestamp', nullable: true })
  lastSsoLoginAt: Date;

  @Column({ default: 0 })
  loginCount: number;

  @Column({ type: 'enum', enum: ['SSO', 'ADMIN'], default: 'SSO' })
  linkedBy: 'SSO' | 'ADMIN';

  @CreateDateColumn()
  linkedAt: Date;
}
```

---

## 2. User Matching: Resolving the Identity

When the `AttributeMapperService` (from Part 2) returns the normalized `SsoUserClaims`, we must use the configured `identifierField` to find the local user. 

### Security Rule: No Just-In-Time (JIT) Provisioning
Our enterprise spec specifically prohibits on-the-fly account creation. Accounts must be pre-provisioned by an administrator. If the SSO user does not match an existing local user, we must explicitly reject the login.

```typescript
// src/sso/services/sso-matching.service.ts
import { Injectable, UnauthorizedException, BadRequestException, Logger } from '@nestjs/common';
import { SsoUserClaims } from '../dto/sso-user-claims.dto';
import { IdpProvider } from '../entities/idp-provider.entity';
import { User } from '../../users/entities/user.entity';
import { UserSsoProfile } from '../entities/user-sso-profile.entity';
import { LocalField } from '../interfaces/attribute-mapping.interface';
import { Repository } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';

@Injectable()
export class SsoMatchingService {
  private readonly logger = new Logger(SsoMatchingService.name);

  constructor(
    @InjectRepository(User) private readonly userRepo: Repository<User>,
    @InjectRepository(UserSsoProfile) private readonly ssoProfileRepo: Repository<UserSsoProfile>,
  ) {}

  async resolveAndLinkUser(claims: SsoUserClaims, provider: IdpProvider): Promise<User> {
    let localUser: User;

    // 1. Resolve User based on configured Identifier
    if (claims.identifierField === LocalField.EXTERNAL_USER_ID) {
      // Look up via the SSO Profile table (Composite Index: providerId + extUserId)
      const existingProfile = await this.ssoProfileRepo.findOne({
        where: { idpProvider: { id: provider.id }, extUserId: claims.identifierValue },
        relations: ['user'],
      });
      if (existingProfile) localUser = existingProfile.user;
    } else {
      // Look up directly on the User table (e.g., matching by Email or Username)
      localUser = await this.userRepo.findOne({
        where: { [claims.identifierField]: claims.identifierValue },
      });
    }

    // 2. Enforce Pre-Provisioning Rule
    if (!localUser) {
      this.logger.warn(`SSO Login rejected: No matching local account for ${claims.identifierField}=${claims.identifierValue}`);
      throw new UnauthorizedException('No matching account found. Please contact your administrator to create your account before using SSO login.');
    }

    // 3. Validate Account State
    if (!localUser.isActive || localUser.isLocked) {
      throw new UnauthorizedException('Your account is currently inactive or locked.');
    }

    // 4. Upsert the SSO Profile Linkage
    await this.upsertSsoProfile(localUser, provider, claims);

    // 5. Sync 'Sync-On-Login' Attributes
    await this.syncUserAttributes(localUser, claims.fieldsToSync);

    return localUser;
  }

  private async upsertSsoProfile(user: User, provider: IdpProvider, claims: SsoUserClaims) {
    let profile = await this.ssoProfileRepo.findOne({
      where: { user: { id: user.id }, idpProvider: { id: provider.id } }
    });

    if (!profile) {
      // First-time linking
      profile = this.ssoProfileRepo.create({
        user,
        idpProvider: provider,
        extUserId: claims[LocalField.EXTERNAL_USER_ID] || claims.identifierValue, // Fallback if ext id wasn't explicitly mapped
        linkedBy: 'SSO',
      });
      // Fire 'First SSO Login' Event for Audit/Notifications
      // this.eventEmitter.emit('sso.account_linked', { userId: user.id, providerId: provider.id });
    }

    // Update login stats and cached display data
    profile.extEmail = claims[LocalField.EMAIL] || profile.extEmail;
    profile.extDisplayName = claims[LocalField.DISPLAY_NAME] || profile.extDisplayName;
    profile.lastSsoLoginAt = new Date();
    profile.loginCount += 1;

    await this.ssoProfileRepo.save(profile);
  }

  private async syncUserAttributes(user: User, fieldsToSync: Record<string, string>) {
    if (Object.keys(fieldsToSync).length === 0) return;

    for (const [field, value] of Object.entries(fieldsToSync)) {
      user[field] = value;
    }
    await this.userRepo.save(user);
  }
}
```

---

## 3. The ENFORCED Mode: Blocking Local Passwords

Our spec defines a system configuration `auth_sso_support` with three states: `DISABLED`, `ENABLED`, and `ENFORCED` (Mandatory).

If the system is in `ENFORCED` mode, users *must* log in via SSO. However, new users who haven't linked an SSO account yet must log in with their local password **once** to establish the initial link. After that link is established, their local password must be effectively disabled.

### The Password Login Interceptor

We must intercept the standard password login flow to check two things:
1. If the user *already* has an SSO link, and mode is `ENFORCED`, block the password login.
2. If the user *does not* have an SSO link, and mode is `ENFORCED`, return a `206 Partial Content` (or a specific error code) telling the frontend to redirect the user to the SSO Linking Prompt.

```typescript
// src/auth/services/local-auth.service.ts
import { Injectable, UnauthorizedException, ForbiddenException } from '@nestjs/common';
// ... other imports

@Injectable()
export class LocalAuthService {
  
  async validateUserPassword(username: string, pass: string): Promise<User> {
    const user = await this.userRepo.findOne({ where: { username }, relations: ['ssoProfiles'] });
    if (!user || !(await bcrypt.compare(pass, user.passwordHash))) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const ssoPolicy = await this.systemConfig.get('auth_sso_support'); // DISABLED, ENABLED, ENFORCED

    if (ssoPolicy === 'ENFORCED') {
      // Exempt System Admins from ENFORCED mode so they don't get locked out if the IdP goes down
      if (user.role === 'SYSTEM_ADMIN') {
        return user;
      }

      if (user.ssoProfiles && user.ssoProfiles.length > 0) {
        // Rule: User has linked an SSO account. Local password login is strictly blocked.
        throw new ForbiddenException('Your account is linked to an Identity Provider. You must use SSO to log in.');
      } else {
        // Rule: User has NO linked account. They must link one now.
        // We throw a custom exception that the Frontend catches to show the "Please Link SSO" modal.
        throw new SsoLinkingRequiredException(user.id);
      }
    }

    return user;
  }
}
```

---

## 4. Bypassing 2FA (TOTP) for SSO Users

In Part 1-10 of our TOTP series, we strictly enforced 2FA for all users. However, in an Enterprise SSO environment, the Identity Provider (Azure AD, Okta) is typically responsible for Multi-Factor Authentication (MFA). 

If we force a user to do Microsoft Authenticator (for Entra ID) and then *also* our platform's TOTP, they will hate the system. The spec mandates: **"SSO users always bypass the platform's TOTP 2FA requirement."**

We handle this in the final session generation step:

```typescript
// src/auth/services/session.service.ts

@Injectable()
export class SessionService {
  
  async issueSessionTokens(user: User, authMethod: 'LOCAL' | 'SSO'): Promise<Tokens> {
    
    let requires2fa = false;

    if (authMethod === 'LOCAL') {
      const is2faGloballyEnabled = await this.systemConfig.get('AUTH_2FA_TOTP_SUPPORT');
      if (is2faGloballyEnabled && user.isTotpEnabled) {
        requires2fa = true;
      }
    } 
    // If authMethod === 'SSO', requires2fa remains false. We trust the IdP.

    if (requires2fa) {
      // Return a partial session token that only has access to the /verify-2fa endpoint
      return this.generatePartialToken(user);
    }

    // Generate full access JWTs
    return this.generateFullTokens(user);
  }
}
```

## Conclusion

In Part 3, we have successfully bridged the gap between federated identities and our local database. By implementing a one-to-many `UserSsoProfile` entity, we support users linking multiple IdPs. By enforcing strict pre-provisioning rules and intercepting local password logins, we ensure that an `ENFORCED` SSO policy is actually secure and cannot be bypassed. Finally, we respected User Experience (UX) by delegating 2FA responsibilities to the trusted Identity Provider.

In Part 4, we will dive into the deepest waters of protocol security: **OAuth 2.0 & OIDC Callback Processing**. We will dissect ID Tokens, implement JWKS signature verification from scratch, and protect against token replay attacks using Nonces.

<br><br><br>

---
---

## 簡介：當身份認證遇上 Database

喺第一集，我哋搞掂咗 Protocol 之間嘅加密交換。喺第二集，我哋起咗個 Engine 去將 Identity Providers (IdPs) 掟埋嚟嗰堆亂七八糟嘅 Data，Map 同 Normalize 做乾淨嘅 `SsoUserClaims`。依家嚟到第三集，我哋要回答任何 SSO 實作入面最致命嘅一個問題：**呢個人到底係邊個？**

Identity Resolution (身份解析) 係充滿保安陷阱嘅。如果我哋 Match 錯咗個 User，就等於雙手奉上咗另一個人嘅 Account 俾黑客 (Account Takeover / ATO)。而且，企業客戶通常要求極度嚴格：一旦用戶 Link 咗個 SSO 帳戶，就 *絕對* 唔可以再俾佢用返 Local password 登入。

今日，我哋會實作 Requirement Spec 入面嘅 `SSO-005`：用戶匹配與強制連結架構 (User Matching and ENFORCED Account Linking Architecture)。我哋會處理安全嘅用戶搜尋、動態帳戶連結、封殺 Local 密碼，同埋為受信任嘅聯邦身份 Bypass TOTP (2FA)。

---

## 1. SSO Profile 實體 (Entity)

一個 Local 嘅 `User` 隨時有多過一個 External identities (例如，有一個 Entra ID 帳號，又有一個 GitHub 帳號)。所以，我哋唔可以直接喺 `users` Table 加一條 `sso_id` Column 就算數。我哋需要一個一對多 (One-to-Many) 嘅 Relationship，將外部身份 Map 返去內部用戶度。

### Clean Architecture: SSO Profile Model

```typescript
// src/sso/entities/user-sso-profile.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, UpdateDateColumn, ManyToOne, JoinColumn, Index } from 'typeorm';
import { User } from '../../users/entities/user.entity';
import { IdpProvider } from './idp-provider.entity';

@Entity('user_sso_profiles')
// 極度重要：確保同一個 IdP 嘅同一個 External ID 唔可以俾人 Link 兩次
@Index(['idpProvider', 'extUserId'], { unique: true })
export class UserSsoProfile {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => User, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'user_id' })
  user: User;

  @ManyToOne(() => IdpProvider, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'idp_provider_id' })
  idpProvider: IdpProvider;

  @Column({ length: 255 })
  extUserId: string; // 例如 IdP 俾嘅 'sub' 或者 'NameID'

  @Column({ length: 255, nullable: true })
  extEmail: string;

  @Column({ length: 255, nullable: true })
  extDisplayName: string;

  @Column({ type: 'timestamp', nullable: true })
  lastSsoLoginAt: Date;

  @Column({ default: 0 })
  loginCount: number;

  @Column({ type: 'enum', enum: ['SSO', 'ADMIN'], default: 'SSO' })
  linkedBy: 'SSO' | 'ADMIN';

  @CreateDateColumn()
  linkedAt: Date;
}
```

---

## 2. 用戶匹配 (User Matching)：找出真身

當第二集嘅 `AttributeMapperService` 俾返份 Normalize 靚晒嘅 `SsoUserClaims` 俾我哋嗰陣，我哋就要用 Config 咗做 `identifierField` 嗰個 Value 去搵返個 Local user 出嚟。

### 保安守則：嚴禁即時開戶 (No JIT Provisioning)
我哋份 Enterprise Spec 寫明嚴禁「On-the-fly」開 Account。Account 必須要由 Admin 預先開好 (Pre-provisioned)。如果個 SSO User 喺我哋個 Database 搵唔到對應嘅 Local user，我哋必須要無情咁 Reject 個登入。

```typescript
// src/sso/services/sso-matching.service.ts
import { Injectable, UnauthorizedException, BadRequestException, Logger } from '@nestjs/common';
import { SsoUserClaims } from '../dto/sso-user-claims.dto';
import { IdpProvider } from '../entities/idp-provider.entity';
import { User } from '../../users/entities/user.entity';
import { UserSsoProfile } from '../entities/user-sso-profile.entity';
import { LocalField } from '../interfaces/attribute-mapping.interface';
import { Repository } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';

@Injectable()
export class SsoMatchingService {
  private readonly logger = new Logger(SsoMatchingService.name);

  constructor(
    @InjectRepository(User) private readonly userRepo: Repository<User>,
    @InjectRepository(UserSsoProfile) private readonly ssoProfileRepo: Repository<UserSsoProfile>,
  ) {}

  async resolveAndLinkUser(claims: SsoUserClaims, provider: IdpProvider): Promise<User> {
    let localUser: User;

    // 1. 根據 Config 咗嘅 Identifier 去搵 User
    if (claims.identifierField === LocalField.EXTERNAL_USER_ID) {
      // 透過 SSO Profile table 去搵 (利用 providerId + extUserId 嘅 Composite Index)
      const existingProfile = await this.ssoProfileRepo.findOne({
        where: { idpProvider: { id: provider.id }, extUserId: claims.identifierValue },
        relations: ['user'],
      });
      if (existingProfile) localUser = existingProfile.user;
    } else {
      // 直接喺 User table 度搵 (例如用 Email 或者 Username 去 Match)
      localUser = await this.userRepo.findOne({
        where: { [claims.identifierField]: claims.identifierValue },
      });
    }

    // 2. 執行「必須預先開戶」嘅死線
    if (!localUser) {
      this.logger.warn(`SSO 登入被拒絕：搵唔到對應嘅 Local account。${claims.identifierField}=${claims.identifierValue}`);
      throw new UnauthorizedException('搵唔到對應嘅帳戶。請聯絡你嘅管理員開咗帳戶先，然後再用 SSO 登入。');
    }

    // 3. 驗證帳戶狀態
    if (!localUser.isActive || localUser.isLocked) {
      throw new UnauthorizedException('你嘅帳戶目前處於停用或鎖定狀態。');
    }

    // 4. 建立或更新 SSO Profile 嘅 Linkage (Upsert)
    await this.upsertSsoProfile(localUser, provider, claims);

    // 5. 將需要 'Sync-On-Login' 嘅 Attributes Sync 返落 User 度
    await this.syncUserAttributes(localUser, claims.fieldsToSync);

    return localUser;
  }

  private async upsertSsoProfile(user: User, provider: IdpProvider, claims: SsoUserClaims) {
    let profile = await this.ssoProfileRepo.findOne({
      where: { user: { id: user.id }, idpProvider: { id: provider.id } }
    });

    if (!profile) {
      // 第一次 Link 埋一齊
      profile = this.ssoProfileRepo.create({
        user,
        idpProvider: provider,
        extUserId: claims[LocalField.EXTERNAL_USER_ID] || claims.identifierValue, // 萬一 ext id 無被 Explicitly map 到嘅 fallback
        linkedBy: 'SSO',
      });
      // 可以喺度射個 'First SSO Login' Event 出去俾 Audit/Notifications 聽
      // this.eventEmitter.emit('sso.account_linked', { userId: user.id, providerId: provider.id });
    }

    // 更新登入統計同埋 Cached display data
    profile.extEmail = claims[LocalField.EMAIL] || profile.extEmail;
    profile.extDisplayName = claims[LocalField.DISPLAY_NAME] || profile.extDisplayName;
    profile.lastSsoLoginAt = new Date();
    profile.loginCount += 1;

    await this.ssoProfileRepo.save(profile);
  }

  private async syncUserAttributes(user: User, fieldsToSync: Record<string, string>) {
    if (Object.keys(fieldsToSync).length === 0) return;

    for (const [field, value] of Object.entries(fieldsToSync)) {
      user[field] = value;
    }
    await this.userRepo.save(user);
  }
}
```

---

## 3. ENFORCED 模式：封殺 Local 密碼

我哋嘅 Spec 定義咗一個叫 `auth_sso_support` 嘅 System config，有三個 States：`DISABLED`, `ENABLED`, 同埋 `ENFORCED` (強制)。

如果個系統轉咗做 `ENFORCED` 模式，所有用戶 *必須* 經 SSO 登入。不過，對於一啲仲未 Link 過 SSO 帳戶嘅新 User，佢哋必須要用 Local 密碼登入 **一次**，去建立第一次嘅 Linkage。當 Link 成功咗之後，佢條 Local 密碼就必須要被封印 (Disabled)。

### 密碼登入攔截器 (The Password Login Interceptor)

我哋必須要攔截標準嘅 Password login flow 去 Check 兩樣嘢：
1. 如果個 User *已經* 有一條 SSO link，而家係 `ENFORCED` 模式，我哋要 Block 咗個 Password login。
2. 如果個 User *仲未* 有 SSO link，而家係 `ENFORCED` 模式，我哋要 Return 一個 `206 Partial Content` (或者特定嘅 Error code)，叫 Frontend 將個 User 踢去 SSO 連結提示畫面 (Linking Prompt)。

```typescript
// src/auth/services/local-auth.service.ts
import { Injectable, UnauthorizedException, ForbiddenException } from '@nestjs/common';
// ... other imports

@Injectable()
export class LocalAuthService {
  
  async validateUserPassword(username: string, pass: string): Promise<User> {
    const user = await this.userRepo.findOne({ where: { username }, relations: ['ssoProfiles'] });
    if (!user || !(await bcrypt.compare(pass, user.passwordHash))) {
      throw new UnauthorizedException('帳號或密碼錯誤');
    }

    const ssoPolicy = await this.systemConfig.get('auth_sso_support'); // DISABLED, ENABLED, ENFORCED

    if (ssoPolicy === 'ENFORCED') {
      // System Admins 擁有豁免權，防止 IdP 瓜咗嗰陣連 Admin 都入唔到去救亡
      if (user.role === 'SYSTEM_ADMIN') {
        return user;
      }

      if (user.ssoProfiles && user.ssoProfiles.length > 0) {
        // 規則：User 已經 Link 咗 SSO Account。嚴格封殺 Local password 登入。
        throw new ForbiddenException('你嘅帳戶已經連結咗 Identity Provider。你必須使用 SSO 進行登入。');
      } else {
        // 規則：User 仲未 Link account。佢必須依家去 Link。
        // 我哋 Throw 一個 Custom Exception，等 Frontend Catch 到之後彈個 "Please Link SSO" Modal 出嚟。
        throw new SsoLinkingRequiredException(user.id);
      }
    }

    return user;
  }
}
```

---

## 4. 為 SSO 用戶 Bypass 2FA (TOTP)

喺我哋 TOTP 系列嘅 1-10 集入面，我哋死守住要所有 User 玩 2FA。不過，喺一個企業級 SSO 環境入面，Multi-Factor Authentication (MFA) 嘅責任通常已經交咗俾 Identity Provider (例如 Azure AD, Okta)。

如果我哋逼個 User 撳完 Microsoft Authenticator (為咗 Entra ID)，入到嚟又要再打多次我哋 Platform 嘅 TOTP，佢哋一定會反枱。Spec 寫得好清楚：**「SSO 用戶永遠豁免本平台嘅 TOTP 2FA 要求。」**

我哋會喺最後 Generate session 嗰步處理呢樣嘢：

```typescript
// src/auth/services/session.service.ts

@Injectable()
export class SessionService {
  
  async issueSessionTokens(user: User, authMethod: 'LOCAL' | 'SSO'): Promise<Tokens> {
    
    let requires2fa = false;

    if (authMethod === 'LOCAL') {
      const is2faGloballyEnabled = await this.systemConfig.get('AUTH_2FA_TOTP_SUPPORT');
      if (is2faGloballyEnabled && user.isTotpEnabled) {
        requires2fa = true;
      }
    } 
    // 如果 authMethod === 'SSO', requires2fa 就會 keep 住係 false。我哋完全信任 IdP。

    if (requires2fa) {
      // 俾個 Partial session token 佢，只可以 Access /verify-2fa endpoint
      return this.generatePartialToken(user);
    }

    // 俾足 Full access 嘅 JWTs 佢
    return this.generateFullTokens(user);
  }
}
```

## 結語

喺第三集，我哋成功搭好咗聯邦身份同 Local database 之間嘅橋樑。透過實作一對多嘅 `UserSsoProfile` Entity，我哋支援一個 User 連接多個 IdPs。透過強制執行 Pre-provisioning 規則同埋攔截 Local password 登入，我哋確保咗 `ENFORCED` SSO 政策係堅如磐石、無法被 Bypass 嘅。最後，我哋兼顧咗 User Experience (UX)，將 2FA 嘅責任交托俾可信嘅 Identity Provider。

喺第四集，我哋將會潛入 Protocol 保安嘅最深水區：**OAuth 2.0 & OIDC Callback 處理**。我哋會解剖 ID Tokens、由零開始實作 JWKS Signature Verification，同埋用 Nonces 嚟防禦 Token 重放攻擊。

<br><br><br>
