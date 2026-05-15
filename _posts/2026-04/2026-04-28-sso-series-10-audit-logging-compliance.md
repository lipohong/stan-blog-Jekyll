---
title: "SSO Series Part 10: Audit Logging & Compliance | SSO 系列之十：審計日誌與合規"
date: 2026-04-28 15:45:00 +0800
categories: [Security, SSO Series]
tags: [sso, audit, compliance, soc2, iso27001, logging, security, typescript, nodejs, assisted_by_ai]
mermaid: true
toc: true
---

## Introduction: If You Didn't Log It, It Didn't Happen

In enterprise security, an event that isn't logged is an event that never occurred. When a security incident happens—whether it's a compromised account, a suspicious login from an unusual location, or an unauthorized configuration change—the first question investigators ask is: "What happened?" If your system cannot answer that question with precise, timestamped, tamper-proof logs, you are flying blind.

Regulatory frameworks like **SOC 2 Type II**, **ISO 27001**, **HIPAA**, and **GDPR** all mandate comprehensive audit logging for authentication events. Enterprise customers will not sign a contract unless you can prove that every SSO login, logout, configuration change, and failure is recorded with sufficient detail for forensic analysis.

In Part 10, we will implement `SSO-010`: **Audit Logging & Compliance**. We will design an immutable audit log system, define the event taxonomy for SSO operations, build alerting rules for suspicious patterns, and prepare the data structures needed for compliance reports.

---

## 1. The Audit Log Architecture

An audit log is fundamentally different from application logs. Application logs are for developers (debugging, monitoring). Audit logs are for security teams and compliance officers (forensics, evidence). They must be:

1. **Immutable** — Once written, never modified or deleted
2. **Append-only** — New events are added, never inserted
3. **Tamper-evident** — Any modification is detectable
4. **Complete** — Every relevant event is captured
5. **Queryable** — Security teams can search and filter efficiently

### Mermaid Diagram: Audit Log Architecture

```mermaid
flowchart TD
    subgraph "Event Sources"
        LOGIN["Login Events"]
        LOGOUT["Logout Events"]
        CONFIG["Config Changes"]
        ADMIN["Admin Actions"]
        ERROR["Security Errors"]
    end

    subgraph "Audit Service"
        EMIT["Event Emitter"]
        ENRICH["Enrichment Layer<br/>(IP, User-Agent, Tenant)"]
        HASH["Hash Chain<br/>(Tamper Detection)"]
        WRITE["Write to Store"]
    end

    subgraph "Storage Layer"
        PRIMARY[(PostgreSQL<br/>audit_logs table<br/>Partitioned by month)]
        ARCHIVE[(Cold Storage<br/>S3 / Azure Blob<br/>Encrypted at rest)]
    end

    subgraph "Query Layer"
        API["Audit Query API"]
        ALERT["Alert Engine<br/>(Suspicious patterns)"]
        REPORT["Compliance Report Generator"]
    end

    LOGIN & LOGOUT & CONFIG & ADMIN & ERROR --> EMIT
    EMIT --> ENRICH --> HASH --> WRITE
    WRITE --> PRIMARY
    PRIMARY -->|"After 90 days"| ARCHIVE
    PRIMARY --> API & ALERT & REPORT

    style HASH fill:#e74c3c,color:#fff
    style PRIMARY fill:#3498db,color:#fff
```

---

## 2. SSO Event Taxonomy

We need a comprehensive, standardized set of event types that covers every SSO operation.

### Mermaid Diagram: SSO Event Taxonomy Tree

```mermaid
graph TD
    ROOT["SSO Audit Events"]

    ROOT --> AUTH["Authentication Events"]
    AUTH --> AUTH_01["SSO_LOGIN_INITIATED"]
    AUTH --> AUTH_02["SSO_LOGIN_SUCCESS"]
    AUTH --> AUTH_03["SSO_LOGIN_FAILED"]
    AUTH --> AUTH_04["SSO_LOGIN_BLOCKED"]
    AUTH --> AUTH_05["TOKEN_VALIDATION_FAILED"]

    ROOT --> SESSION["Session Events"]
    SESSION --> SESS_01["SESSION_CREATED"]
    SESSION --> SESS_02["SESSION_DESTROYED"]
    SESSION --> SESS_03["SESSION_EXPIRED"]
    SESSION --> SESS_04["SESSION_TERMINATED_BCL"]

    ROOT --> LOGOUT["Logout Events"]
    LOGOUT --> LOG_01["LOGOUT_INITIATED_SP"]
    LOGOUT --> LOG_02["LOGOUT_SUCCESS_SP"]
    LOGOUT --> LOG_03["LOGOUT_INITIATED_IDP"]
    LOGOUT --> LOG_04["LOGOUT_FAILED"]

    ROOT --> CONFIG["Configuration Events"]
    CONFIG --> CFG_01["PROVIDER_CREATED"]
    CONFIG --> CFG_02["PROVIDER_UPDATED"]
    CONFIG --> CFG_03["PROVIDER_ENABLED"]
    CONFIG --> CFG_04["PROVIDER_DISABLED"]
    CONFIG --> CFG_05["MAPPINGS_UPDATED"]
    CONFIG --> CFG_06["CERTIFICATE_ROTATED"]

    ROOT --> SECURITY["Security Events"]
    SECURITY --> SEC_01["REPLAY_ATTACK_DETECTED"]
    SECURITY --> SEC_02["CSRF_ATTACK_DETECTED"]
    SECURITY --> SEC_03["XSW_ATTACK_DETECTED"]
    SECURITY --> SEC_04["CROSS_TENANT_ATTEMPT"]
    SECURITY --> SEC_05["EMERGENCY_KEY_REFRESH"]

    style ROOT fill:#2c3e50,color:#fff
    style SECURITY fill:#e74c3c,color:#fff
```

---

## 3. The Audit Log Entity

### Code Implementation: Audit Log Entity

```typescript
// src/audit/entities/audit-log.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, Index } from 'typeorm';

export enum AuditEventType {
  // Authentication
  SSO_LOGIN_INITIATED = 'SSO_LOGIN_INITIATED',
  SSO_LOGIN_SUCCESS = 'SSO_LOGIN_SUCCESS',
  SSO_LOGIN_FAILED = 'SSO_LOGIN_FAILED',
  SSO_LOGIN_BLOCKED = 'SSO_LOGIN_BLOCKED',
  TOKEN_VALIDATION_FAILED = 'TOKEN_VALIDATION_FAILED',

  // Session
  SESSION_CREATED = 'SESSION_CREATED',
  SESSION_DESTROYED = 'SESSION_DESTROYED',
  SESSION_EXPIRED = 'SESSION_EXPIRED',
  SESSION_TERMINATED_BCL = 'SESSION_TERMINATED_BCL',

  // Logout
  LOGOUT_INITIATED_SP = 'LOGOUT_INITIATED_SP',
  LOGOUT_SUCCESS_SP = 'LOGOUT_SUCCESS_SP',
  LOGOUT_INITIATED_IDP = 'LOGOUT_INITIATED_IDP',
  LOGOUT_FAILED = 'LOGOUT_FAILED',

  // Configuration
  PROVIDER_CREATED = 'PROVIDER_CREATED',
  PROVIDER_UPDATED = 'PROVIDER_UPDATED',
  PROVIDER_ENABLED = 'PROVIDER_ENABLED',
  PROVIDER_DISABLED = 'PROVIDER_DISABLED',
  MAPPINGS_UPDATED = 'MAPPINGS_UPDATED',
  CERTIFICATE_ROTATED = 'CERTIFICATE_ROTATED',

  // Security
  REPLAY_ATTACK_DETECTED = 'REPLAY_ATTACK_DETECTED',
  CSRF_ATTACK_DETECTED = 'CSRF_ATTACK_DETECTED',
  CROSS_TENANT_ATTEMPT = 'CROSS_TENANT_ATTEMPT',
  EMERGENCY_KEY_REFRESH = 'EMERGENCY_KEY_REFRESH',
}

export enum AuditSeverity {
  INFO = 'INFO',
  WARNING = 'WARNING',
  CRITICAL = 'CRITICAL',
}

@Entity('audit_logs')
@Index(['tenantId', 'eventType', 'createdAt'])
@Index(['tenantId', 'userId', 'createdAt'])
@Index(['createdAt']) // For partition pruning
export class AuditLog {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'tenant_id' })
  tenantId: string;

  @Column({ name: 'event_type', type: 'enum', enum: AuditEventType })
  eventType: AuditEventType;

  @Column({ type: 'enum', enum: AuditSeverity, default: AuditSeverity.INFO })
  severity: AuditSeverity;

  @Column({ name: 'user_id', nullable: true })
  userId: string;

  @Column({ name: 'user_email', nullable: true })
  userEmail: string;

  @Column({ name: 'provider_id', nullable: true })
  providerId: string;

  @Column({ name: 'provider_code', nullable: true })
  providerCode: string;

  @Column({ name: 'session_id', nullable: true })
  sessionId: string;

  @Column({ name: 'ip_address', length: 45 }) // IPv6 max length
  ipAddress: string;

  @Column({ name: 'user_agent', length: 512, nullable: true })
  userAgent: string;

  @Column({ type: 'jsonb', nullable: true })
  details: Record<string, any>;

  @Column({ name: 'error_message', nullable: true })
  errorMessage: string;

  @Column({ name: 'previous_hash', length: 64, nullable: true })
  previousHash: string;

  @Column({ name: 'event_hash', length: 64 })
  eventHash: string;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

---

## 4. The Audit Service

### Code Implementation: Core Audit Service

```typescript
// src/audit/services/audit.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import * as crypto from 'crypto';
import { AuditLog, AuditEventType, AuditSeverity } from '../entities/audit-log.entity';

export interface AuditEventContext {
  tenantId: string;
  userId?: string;
  userEmail?: string;
  providerId?: string;
  providerCode?: string;
  sessionId?: string;
  ipAddress?: string;
  userAgent?: string;
  details?: Record<string, any>;
  errorMessage?: string;
}

@Injectable()
export class AuditService {
  private readonly logger = new Logger(AuditService.name);

  constructor(
    @InjectRepository(AuditLog)
    private readonly auditRepo: Repository<AuditLog>,
  ) {}

  async log(
    eventType: AuditEventType,
    severity: AuditSeverity,
    context: AuditEventContext,
  ): Promise<void> {
    try {
      // 1. Get the previous hash for chain integrity
      const previousLog = await this.auditRepo.findOne({
        where: { tenantId: context.tenantId },
        order: { createdAt: 'DESC' },
        select: ['eventHash'],
      });

      // 2. Compute hash of this event
      const eventPayload = JSON.stringify({
        eventType,
        severity,
        ...context,
        timestamp: new Date().toISOString(),
      });
      const eventHash = crypto.createHash('sha256').update(eventPayload).digest('hex');

      // 3. Create the audit log entry
      const log = this.auditRepo.create({
        tenantId: context.tenantId,
        eventType,
        severity,
        userId: context.userId,
        userEmail: context.userEmail,
        providerId: context.providerId,
        providerCode: context.providerCode,
        sessionId: context.sessionId,
        ipAddress: context.ipAddress,
        userAgent: context.userAgent,
        details: context.details,
        errorMessage: context.errorMessage,
        previousHash: previousLog?.eventHash || null,
        eventHash,
      });

      await this.auditRepo.save(log);

      // 4. Check alert rules asynchronously
      this.checkAlertRules(eventType, severity, context);
    } catch (error) {
      // Audit logging must NEVER crash the main application
      this.logger.error(`Failed to write audit log: ${error.message}`, error.stack);
    }
  }

  // Convenience methods
  async logInfo(eventType: AuditEventType, context: AuditEventContext): Promise<void> {
    return this.log(eventType, AuditSeverity.INFO, context);
  }

  async logWarning(eventType: AuditEventType, context: AuditEventContext): Promise<void> {
    return this.log(eventType, AuditSeverity.WARNING, context);
  }

  async logCritical(eventType: AuditEventType, context: AuditEventContext): Promise<void> {
    return this.log(eventType, AuditSeverity.CRITICAL, context);
  }

  private checkAlertRules(
    eventType: AuditEventType,
    severity: AuditSeverity,
    context: AuditEventContext,
  ): void {
    // Async alert checking — see Section 6
    if (severity === AuditSeverity.CRITICAL) {
      // this.eventEmitter.emit('audit.critical', { eventType, context });
    }
  }
}
```

---

## 5. Instrumenting the SSO Flow

We must add audit logging calls at every critical point in the SSO flow.

### Mermaid Diagram: Audit Points in the SSO Flow

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant SP as Service Provider
    participant Audit as Audit Service
    participant IdP as Identity Provider

    U->>SP: Click "Login with SSO"
    SP->>Audit: SSO_LOGIN_INITIATED ℹ️

    SP->>IdP: Redirect to IdP
    IdP->>U: Login page
    U->>IdP: Enter credentials
    IdP->>SP: Callback with code/assertion

    alt Login Success
        SP->>Audit: SSO_LOGIN_SUCCESS ℹ️
        SP->>Audit: SESSION_CREATED ℹ️
        SP->>U: Redirect to dashboard
    else Login Failed (Invalid Token)
        SP->>Audit: TOKEN_VALIDATION_FAILED ⚠️
        SP->>U: 401 Unauthorized
    else Login Failed (No Matching User)
        SP->>Audit: SSO_LOGIN_BLOCKED ⚠️
        SP->>U: 401 No matching account
    end

    Note over SP,Audit: Later: User clicks Logout
    U->>SP: Click "Logout"
    SP->>Audit: LOGOUT_INITIATED_SP ℹ️
    SP->>SP: Destroy local session
    SP->>Audit: SESSION_DESTROYED ℹ️
    SP->>Audit: LOGOUT_SUCCESS_SP ℹ️
    SP->>IdP: Redirect to IdP logout
```

### Code Implementation: Audit Instrumentation in SSO Flow

```typescript
// src/sso/services/sso-callback.service.ts (updated with audit logging)

import { AuditService } from '../../audit/services/audit.service';
import { AuditEventType, AuditSeverity } from '../../audit/entities/audit-log.entity';

@Injectable()
export class SsoCallbackService {
  constructor(
    // ... existing dependencies
    private readonly auditService: AuditService,
  ) {}

  async handleCallback(providerId: string, payload: any, request: any): Promise<any> {
    const context = {
      tenantId: request.tenantId,
      providerId,
      ipAddress: request.ip,
      userAgent: request.headers['user-agent'],
    };

    try {
      // ... existing callback logic ...

      // Log successful login
      await this.auditService.logInfo(AuditEventType.SSO_LOGIN_SUCCESS, {
        ...context,
        userId: user.id,
        userEmail: user.email,
        sessionId: session.id,
        details: {
          protocol: provider.protocolType,
          providerCode: provider.providerCode,
        },
      });

      await this.auditService.logInfo(AuditEventType.SESSION_CREATED, {
        ...context,
        userId: user.id,
        sessionId: session.id,
      });

      return { user, session };
    } catch (error) {
      // Log failed login
      const eventType = error instanceof UnauthorizedException
        ? AuditEventType.TOKEN_VALIDATION_FAILED
        : AuditEventType.SSO_LOGIN_FAILED;

      await this.auditService.logWarning(eventType, {
        ...context,
        errorMessage: error.message,
        details: {
          errorCode: error.getStatus?.() || 500,
        },
      });

      throw error;
    }
  }
}
```

---

## 6. Alert Rules: Detecting Suspicious Patterns

Automated alerting is critical for real-time threat detection.

### Mermaid Diagram: Alert Rule Engine

```mermaid
flowchart TD
    EVENT["Audit Event<br/>received"] --> RULES["Alert Rule Engine"]

    RULES --> R1{"Brute Force?<br/>> 5 failed logins<br/>in 10 min from same IP"}
    RULES --> R2{"Credential Stuffing?<br/>> 20 failed logins<br/>from different IPs<br/>for same user"}
    RULES --> R3{"Unusual Location?<br/>Login from new country<br/>for this user"}
    RULES --> R4{"Config Tampering?<br/>Provider enabled/disabled<br/>by non-admin"}
    RULES --> R5{"Mass Session Kill?<br/>> 10 sessions terminated<br/>via BCL in 1 min"}

    R1 -->|"Yes"| ALERT1["🚨 Alert: Possible brute force<br/>→ Block IP temporarily<br/>→ Notify SOC team"]
    R2 -->|"Yes"| ALERT2["🚨 Alert: Credential stuffing<br/>→ Lock account<br/>→ Notify user + SOC"]
    R3 -->|"Yes"| ALERT3["⚠️ Alert: Unusual location<br/>→ Require MFA<br/>→ Notify user"]
    R4 -->|"Yes"| ALERT4["🚨 Alert: Unauthorized config change<br/>→ Revert change<br/>→ Notify tenant admin"]
    R5 -->|"Yes"| ALERT5["⚠️ Alert: Mass session termination<br/>→ Investigate IdP status<br/>→ Notify ops team"]

    style ALERT1 fill:#e74c3c,color:#fff
    style ALERT2 fill:#e74c3c,color:#fff
    style ALERT3 fill:#f39c12,color:#fff
    style ALERT4 fill:#e74c3c,color:#fff
    style ALERT5 fill:#f39c12,color:#fff
```

### Code Implementation: Alert Rule Service

```typescript
// src/audit/services/alert-rule.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Redis } from 'ioredis';
import { AuditEventType, AuditSeverity } from '../entities/audit-log.entity';

@Injectable()
export class AlertRuleService {
  private readonly logger = new Logger(AlertRuleService.name);

  constructor(private readonly redisClient: Redis) {}

  async checkBruteForce(
    tenantId: string,
    ipAddress: string,
    eventType: AuditEventType,
  ): Promise<boolean> {
    if (eventType !== AuditEventType.TOKEN_VALIDATION_FAILED &&
        eventType !== AuditEventType.SSO_LOGIN_FAILED) {
      return false;
    }

    const key = `alert:brute:${tenantId}:${ipAddress}`;
    const count = await this.redisClient.incr(key);
    await this.redisClient.expire(key, 600); // 10-minute window

    if (count > 5) {
      this.logger.warn(`Brute force detected: ${count} failures from IP ${ipAddress} for tenant ${tenantId}`);
      return true;
    }
    return false;
  }

  async checkCredentialStuffing(
    tenantId: string,
    userId: string,
    eventType: AuditEventType,
  ): Promise<boolean> {
    if (eventType !== AuditEventType.SSO_LOGIN_FAILED) {
      return false;
    }

    const key = `alert:stuffing:${tenantId}:${userId}`;
    const count = await this.redisClient.incr(key);
    await this.redisClient.expire(key, 600);

    if (count > 20) {
      this.logger.warn(`Credential stuffing suspected: ${count} failures for user ${userId}`);
      return true;
    }
    return false;
  }
}
```

---

## 7. Compliance Reports

SOC 2 and ISO 27001 auditors need structured reports showing that your SSO controls are operating effectively.

### Mermaid Diagram: Compliance Report Generation Flow

```mermaid
flowchart LR
    subgraph "Report Request"
        REQ["Admin / Auditor<br/>requests report"]
        PERIOD["Select time period<br/>(e.g., last 90 days)"]
        TYPE["Select report type"]
    end

    subgraph "Report Types"
        T1["Login Activity Report<br/>- Total logins<br/>- Success/failure rate<br/>- By provider"]
        T2["Security Incident Report<br/>- Failed validations<br/>- Replay attempts<br/>- Cross-tenant attempts"]
        T3["Configuration Change Report<br/>- Provider changes<br/>- Who changed what<br/>- When"]
        T4["Session Management Report<br/>- Sessions created<br/>- Sessions terminated<br/>- BCL events"]
    end

    subgraph "Data Aggregation"
        QUERY["Query audit_logs<br/>WHERE tenant_id = X<br/>AND created_at BETWEEN Y AND Z"]
        AGGREGATE["Aggregate by event_type,<br/>severity, provider, user"]
    end

    subgraph "Output"
        PDF["PDF Report"]
        CSV["CSV Export"]
        API["JSON API Response"]
    end

    REQ --> PERIOD --> TYPE
    TYPE --> T1 & T2 & T3 & T4
    T1 & T2 & T3 & T4 --> QUERY --> AGGREGATE
    AGGREGATE --> PDF & CSV & API
```

### Code Implementation: Compliance Report Generator

```typescript
// src/audit/services/compliance-report.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, Between } from 'typeorm';
import { AuditLog, AuditEventType, AuditSeverity } from '../entities/audit-log.entity';

export interface ComplianceReport {
  tenantId: string;
  period: { from: Date; to: Date };
  generatedAt: Date;
  summary: {
    totalLogins: number;
    successfulLogins: number;
    failedLogins: number;
    successRate: number;
    securityIncidents: number;
    configChanges: number;
    sessionsCreated: number;
    sessionsTerminated: number;
  };
  byProvider: Record<string, {
    logins: number;
    failures: number;
  }>;
  securityEvents: Array<{
    eventType: AuditEventType;
    severity: AuditSeverity;
    timestamp: Date;
    details: any;
  }>;
  configChanges: Array<{
    eventType: AuditEventType;
    userId: string;
    timestamp: Date;
    details: any;
  }>;
}

@Injectable()
export class ComplianceReportService {
  constructor(
    @InjectRepository(AuditLog)
    private readonly auditRepo: Repository<AuditLog>,
  ) {}

  async generateReport(
    tenantId: string,
    from: Date,
    to: Date,
  ): Promise<ComplianceReport> {
    const logs = await this.auditRepo.find({
      where: {
        tenantId,
        createdAt: Between(from, to),
      },
      order: { createdAt: 'ASC' },
    });

    const loginEvents = logs.filter(l =>
      [AuditEventType.SSO_LOGIN_SUCCESS, AuditEventType.SSO_LOGIN_FAILED].includes(l.eventType)
    );

    const successful = loginEvents.filter(l => l.eventType === AuditEventType.SSO_LOGIN_SUCCESS);
    const failed = loginEvents.filter(l => l.eventType === AuditEventType.SSO_LOGIN_FAILED);

    const securityEvents = logs.filter(l => l.severity === AuditSeverity.CRITICAL);
    const configChanges = logs.filter(l => l.eventType.startsWith('PROVIDER_') || l.eventType === 'MAPPINGS_UPDATED');

    const byProvider: Record<string, { logins: number; failures: number }> = {};
    for (const log of loginEvents) {
      const code = log.providerCode || 'unknown';
      if (!byProvider[code]) byProvider[code] = { logins: 0, failures: 0 };
      if (log.eventType === AuditEventType.SSO_LOGIN_SUCCESS) byProvider[code].logins++;
      else byProvider[code].failures++;
    }

    return {
      tenantId,
      period: { from, to },
      generatedAt: new Date(),
      summary: {
        totalLogins: loginEvents.length,
        successfulLogins: successful.length,
        failedLogins: failed.length,
        successRate: loginEvents.length > 0
          ? Math.round((successful.length / loginEvents.length) * 100)
          : 0,
        securityIncidents: securityEvents.length,
        configChanges: configChanges.length,
        sessionsCreated: logs.filter(l => l.eventType === AuditEventType.SESSION_CREATED).length,
        sessionsTerminated: logs.filter(l =>
          [AuditEventType.SESSION_DESTROYED, AuditEventType.SESSION_TERMINATED_BCL].includes(l.eventType)
        ).length,
      },
      byProvider,
      securityEvents: securityEvents.map(l => ({
        eventType: l.eventType,
        severity: l.severity,
        timestamp: l.createdAt,
        details: l.details,
      })),
      configChanges: configChanges.map(l => ({
        eventType: l.eventType,
        userId: l.userId,
        timestamp: l.createdAt,
        details: l.details,
      })),
    };
  }
}
```

---

## 8. Log Retention & Archival

Audit logs must be retained for specific periods depending on the compliance framework.

### Mermaid Diagram: Log Retention Lifecycle

```mermaid
flowchart LR
    subgraph "Hot Storage (0-90 days)"
        HOT["PostgreSQL<br/>audit_logs table<br/>Partitioned by month"]
        HOT_QUERY["Fast queries<br/>Real-time alerts"]
    end

    subgraph "Warm Storage (90 days - 1 year)"
        WARM["PostgreSQL<br/>Compressed partitions"]
        WARM_QUERY["Compliance reports<br/>Incident investigation"]
    end

    subgraph "Cold Storage (1-7 years)"
        COLD["S3 / Azure Blob<br/>Encrypted, immutable"]
        COLD_QUERY["Legal discovery<br/>Regulatory audit"]
    end

    subgraph "Deletion (After 7 years)"
        DELETE["Secure deletion<br/>Cryptographic erasure"]
    end

    HOT -->|"90 days"| WARM -->|"1 year"| COLD -->|"7 years"| DELETE

    style HOT fill:#27ae60,color:#fff
    style WARM fill:#f39c12,color:#fff
    style COLD fill:#3498db,color:#fff
    style DELETE fill:#e74c3c,color:#fff
```

| Compliance Framework | Minimum Retention | Recommended |
|---|---|---|
| **SOC 2** | 1 year | 3 years |
| **ISO 27001** | 3 years | 5 years |
| **HIPAA** | 6 years | 7 years |
| **GDPR** | As long as needed | Minimize, then delete |

---

## 9. Data Privacy: GDPR Considerations

Audit logs contain personal data (user IDs, emails, IP addresses). Under GDPR, we must balance audit requirements with data minimization.

### Mermaid Diagram: GDPR Compliance for Audit Logs

```mermaid
flowchart TD
    LOG["Audit Log Entry<br/>(Contains PII)"] --> ANONYMIZE{"Anonymize<br/>after retention?"}

    ANONYMIZE -->|"After 1 year"| HASH_USER["Hash userId<br/>SHA-256(userId + salt)"]
    ANONYMIZE -->|"After 2 years"| MASK_IP["Mask IP address<br/>192.168.1.xxx"]
    ANONYMIZE -->|"After 3 years"| REMOVE_EMAIL["Remove userEmail<br/>Keep eventType only"]

    HASH_USER --> KEEP_HASH["Keep hash for<br/>pattern analysis"]
    MASK_IP --> KEEP_MASKED["Keep masked IP<br/>for geo analysis"]
    REMOVE_EMAIL --> KEEP_MINIMAL["Keep minimal record<br/>for compliance count"]

    style ANONYMIZE fill:#9b59b6,color:#fff
    style KEEP_MINIMAL fill:#27ae60,color:#fff
```

### Code Implementation: GDPR Anonymization

```typescript
// src/audit/services/audit-retention.service.ts
@Injectable()
export class AuditRetentionService {
  constructor(
    @InjectRepository(AuditLog) private readonly auditRepo: Repository<AuditLog>,
    private readonly configService: ConfigService,
  ) {}

  @Cron(CronExpression.EVERY_DAY_AT_3AM)
  async anonymizeOldLogs(): Promise<void> {
    const salt = this.configService.get('AUDIT_ANONYMIZE_SALT');
    const oneYearAgo = new Date();
    oneYearAgo.setFullYear(oneYearAgo.getFullYear() - 1);

    // Anonymize user identifiers older than 1 year
    await this.auditRepo
      .createQueryBuilder()
      .update(AuditLog)
      .set({
        userId: () => `encode(sha256((user_id || '${salt}')::bytea), 'hex')`,
        userEmail: '***@***.***',
      })
      .where('created_at < :date', { date: oneYearAgo })
      .andWhere('user_id IS NOT NULL')
      .execute();

    this.logger.log('Anonymized audit logs older than 1 year');
  }
}
```

---

## 10. The Complete SSO Audit Dashboard

### Mermaid Diagram: Audit Dashboard Data Flow

```mermaid
flowchart LR
    subgraph "Real-Time Metrics"
        RT1["Active Sessions<br/>(Current count)"]
        RT2["Login Rate<br/>(per minute)"]
        RT3["Failure Rate<br/>(last hour)"]
        RT4["BCL Events<br/>(last 24h)"]
    end

    subgraph "Trend Analysis"
        TR1["Daily login volume"]
        TR2["Provider usage split"]
        TR3["Failure reasons breakdown"]
        TR4["Geographic distribution"]
    end

    subgraph "Security Alerts"
        AL1["Active investigations"]
        AL2["Blocked IPs"]
        AL3["Locked accounts"]
        AL4["Key rotation status"]
    end

    subgraph "API Endpoints"
        GET1["GET /audit/dashboard/realtime"]
        GET2["GET /audit/dashboard/trends?period=30d"]
        GET3["GET /audit/dashboard/security"]
        GET4["GET /audit/reports/compliance?from=&to="]
    end

    RT1 & RT2 & RT3 & RT4 --> GET1
    TR1 & TR2 & TR3 & TR4 --> GET2
    AL1 & AL2 & AL3 & AL4 --> GET3
    GET4 --> REPORT["Generate PDF / CSV"]
```

---

## Conclusion: The Complete SSO Journey

We have reached the end of our 10-part SSO series. Let's recap the complete architecture we've built:

### Mermaid Diagram: The Complete SSO Architecture (Final)

```mermaid
flowchart TB
    subgraph "Part 1: Foundation"
        P1_STRAT["Strategy Pattern"]
        P1_ENV["Envelope Encryption"]
        P1_PKCE["PKCE"]
        P1_STATE["State Management"]
    end

    subgraph "Part 2: Configuration"
        P2_DISC["Auto-Discovery"]
        P2_MAP["Attribute Mapping Engine"]
        P2_TRANSFORM["Transform Engine"]
    end

    subgraph "Part 3: Identity"
        P3_MATCH["User Matching"]
        P3_ENFORCED["ENFORCED Mode"]
        P3_PROFILE["SSO Profile Linking"]
    end

    subgraph "Part 4-5: Protocols"
        P4_OIDC["OIDC: JWT + JWKS"]
        P5_SAML["SAML: XML + C14N"]
    end

    subgraph "Part 6-7: Logout"
        P6_SP["SP-Initiated SLO"]
        P7_BCL["IdP-Initiated BCL"]
    end

    subgraph "Part 8: Keys"
        P8_ROTATION["Key Rotation"]
        P8_DUAL["Dual Certificates"]
    end

    subgraph "Part 9: Scale"
        P9_TENANT["Multi-Tenant"]
        P9_ISOLATION["Tenant Isolation"]
    end

    subgraph "Part 10: Visibility"
        P10_AUDIT["Audit Logging"]
        P10_ALERT["Alert Rules"]
        P10_COMPLIANCE["Compliance Reports"]
    end

    P1_STRAT --> P4_OIDC & P5_SAML
    P1_ENV --> P8_ROTATION
    P1_PKCE --> P4_OIDC
    P2_DISC --> P4_OIDC & P5_SAML
    P2_MAP --> P3_MATCH
    P3_MATCH --> P3_PROFILE
    P4_OIDC & P5_SAML --> P6_SP & P7_BCL
    P6_SP & P7_BCL --> P10_AUDIT
    P9_TENANT --> P9_ISOLATION
    P10_AUDIT --> P10_ALERT --> P10_COMPLIANCE

    style P1_STRAT fill:#3498db,color:#fff
    style P1_ENV fill:#9b59b6,color:#fff
    style P10_AUDIT fill:#27ae60,color:#fff
```

From Strategy Patterns to Envelope Encryption, from PKCE to Back-Channel Logout, from single-tenant to multi-tenant, and from raw events to compliance reports—this is the complete enterprise SSO architecture.

Thank you for following this series. Keep building securely.

<br><br><br>

---

---

## 簡介：冇 Log 就等於冇發生過

喺企業級保安入面，一個冇被記錄嘅事件就等於一個從來冇發生過嘅事件。當一件保安事故發生——無論係 Account 被入侵、可疑嘅異地登入、定係未經授權嘅設定變更——調查員第一個問嘅問題就係：「發生咗咩事？」如果你個 System 冇辦法用精準嘅、有時間戳嘅、防篡改嘅 Logs 嚟回答呢個問題，你就係盲摸摸咁飛。

好似 **SOC 2 Type II**、**ISO 27001**、**HIPAA** 同 **GDPR** 呢啲法規框架，全部都規定認證事件必須有全面嘅審計日誌。如果你冇辦法證明每一個 SSO 登入、登出、設定變更同失敗都有被記錄到足夠嘅詳情去做取證分析，企業客戶根本唔會同你簽合約。

喺第十集（最後一集！），我哋會實作 `SSO-010`：**審計日誌與合規**。我哋會設計一個不可變嘅審計日誌系統、定義 SSO 操作嘅事件分類法、建立可疑模式嘅警報規則，同埋準備合規報告所需嘅數據結構。

---

## 1. 審計日誌架構

審計日誌同 Application Logs 有根本性嘅唔同。Application Logs 係俾 Developer 用（Debug、監控）。審計日誌係俾保安團隊同合規官員用（取證、證據）。佢哋必須係：

1. **不可變（Immutable）** — 寫咗之後永遠唔可以改或者 Delete
2. **僅追加（Append-only）** — 新 Events 係加落去，唔係插入
3. **防篡改（Tamper-evident）** — 任何修改都可以被偵測到
4. **完整（Complete）** — 每一個相關嘅 Event 都被捕捉
5. **可查詢（Queryable）** — 保安團隊可以有效咁 Search 同 Filter

### Mermaid 圖解：審計日誌架構

```mermaid
flowchart TD
    subgraph "事件來源"
        LOGIN["登入事件"]
        LOGOUT["登出事件"]
        CONFIG["設定變更"]
        ADMIN["Admin 操作"]
        ERROR["保安錯誤"]
    end

    subgraph "審計服務"
        EMIT["Event Emitter"]
        ENRICH["豐富層<br/>（IP、User-Agent、租戶）"]
        HASH["Hash Chain<br/>（防篡改偵測）"]
        WRITE["寫入儲存"]
    end

    subgraph "儲存層"
        PRIMARY[(PostgreSQL<br/>audit_logs table<br/>按月分區)]
        ARCHIVE[(冷儲存<br/>S3 / Azure Blob<br/>靜態加密)]
    end

    subgraph "查詢層"
        API["審計查詢 API"]
        ALERT["警報引擎<br/>（可疑模式）"]
        REPORT["合規報告產生器"]
    end

    LOGIN & LOGOUT & CONFIG & ADMIN & ERROR --> EMIT
    EMIT --> ENRICH --> HASH --> WRITE
    WRITE --> PRIMARY
    PRIMARY -->|"90 日後"| ARCHIVE
    PRIMARY --> API & ALERT & REPORT

    style HASH fill:#e74c3c,color:#fff
    style PRIMARY fill:#3498db,color:#fff
```

---

## 2. SSO 事件分類法

我哋需要一套全面、標準化嘅事件類型，覆蓋每一個 SSO 操作。

### Mermaid 圖解：SSO 事件分類樹

```mermaid
graph TD
    ROOT["SSO 審計事件"]

    ROOT --> AUTH["認證事件"]
    AUTH --> AUTH_01["SSO_LOGIN_INITIATED"]
    AUTH --> AUTH_02["SSO_LOGIN_SUCCESS"]
    AUTH --> AUTH_03["SSO_LOGIN_FAILED"]
    AUTH --> AUTH_04["SSO_LOGIN_BLOCKED"]
    AUTH --> AUTH_05["TOKEN_VALIDATION_FAILED"]

    ROOT --> SESSION["Session 事件"]
    SESSION --> SESS_01["SESSION_CREATED"]
    SESSION --> SESS_02["SESSION_DESTROYED"]
    SESSION --> SESS_03["SESSION_EXPIRED"]
    SESSION --> SESS_04["SESSION_TERMINATED_BCL"]

    ROOT --> LOGOUT["登出事件"]
    LOGOUT --> LOG_01["LOGOUT_INITIATED_SP"]
    LOGOUT --> LOG_02["LOGOUT_SUCCESS_SP"]
    LOGOUT --> LOG_03["LOGOUT_INITIATED_IDP"]
    LOGOUT --> LOG_04["LOGOUT_FAILED"]

    ROOT --> CONFIG["設定事件"]
    CONFIG --> CFG_01["PROVIDER_CREATED"]
    CONFIG --> CFG_02["PROVIDER_UPDATED"]
    CONFIG --> CFG_03["PROVIDER_ENABLED"]
    CONFIG --> CFG_04["PROVIDER_DISABLED"]
    CONFIG --> CFG_05["MAPPINGS_UPDATED"]
    CONFIG --> CFG_06["CERTIFICATE_ROTATED"]

    ROOT --> SECURITY["保安事件"]
    SECURITY --> SEC_01["REPLAY_ATTACK_DETECTED"]
    SECURITY --> SEC_02["CSRF_ATTACK_DETECTED"]
    SECURITY --> SEC_03["XSW_ATTACK_DETECTED"]
    SECURITY --> SEC_04["CROSS_TENANT_ATTEMPT"]
    SECURITY --> SEC_05["EMERGENCY_KEY_REFRESH"]

    style ROOT fill:#2c3e50,color:#fff
    style SECURITY fill:#e74c3c,color:#fff
```

---

## 3. 審計日誌 Entity

### Code 實作：審計日誌 Entity

```typescript
// src/audit/entities/audit-log.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, Index } from 'typeorm';

export enum AuditEventType {
  // 認證
  SSO_LOGIN_INITIATED = 'SSO_LOGIN_INITIATED',
  SSO_LOGIN_SUCCESS = 'SSO_LOGIN_SUCCESS',
  SSO_LOGIN_FAILED = 'SSO_LOGIN_FAILED',
  SSO_LOGIN_BLOCKED = 'SSO_LOGIN_BLOCKED',
  TOKEN_VALIDATION_FAILED = 'TOKEN_VALIDATION_FAILED',

  // Session
  SESSION_CREATED = 'SESSION_CREATED',
  SESSION_DESTROYED = 'SESSION_DESTROYED',
  SESSION_EXPIRED = 'SESSION_EXPIRED',
  SESSION_TERMINATED_BCL = 'SESSION_TERMINATED_BCL',

  // 登出
  LOGOUT_INITIATED_SP = 'LOGOUT_INITIATED_SP',
  LOGOUT_SUCCESS_SP = 'LOGOUT_SUCCESS_SP',
  LOGOUT_INITIATED_IDP = 'LOGOUT_INITIATED_IDP',
  LOGOUT_FAILED = 'LOGOUT_FAILED',

  // 設定
  PROVIDER_CREATED = 'PROVIDER_CREATED',
  PROVIDER_UPDATED = 'PROVIDER_UPDATED',
  PROVIDER_ENABLED = 'PROVIDER_ENABLED',
  PROVIDER_DISABLED = 'PROVIDER_DISABLED',
  MAPPINGS_UPDATED = 'MAPPINGS_UPDATED',
  CERTIFICATE_ROTATED = 'CERTIFICATE_ROTATED',

  // 保安
  REPLAY_ATTACK_DETECTED = 'REPLAY_ATTACK_DETECTED',
  CSRF_ATTACK_DETECTED = 'CSRF_ATTACK_DETECTED',
  CROSS_TENANT_ATTEMPT = 'CROSS_TENANT_ATTEMPT',
  EMERGENCY_KEY_REFRESH = 'EMERGENCY_KEY_REFRESH',
}

export enum AuditSeverity {
  INFO = 'INFO',
  WARNING = 'WARNING',
  CRITICAL = 'CRITICAL',
}

@Entity('audit_logs')
@Index(['tenantId', 'eventType', 'createdAt'])
@Index(['tenantId', 'userId', 'createdAt'])
@Index(['createdAt'])
export class AuditLog {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'tenant_id' })
  tenantId: string;

  @Column({ name: 'event_type', type: 'enum', enum: AuditEventType })
  eventType: AuditEventType;

  @Column({ type: 'enum', enum: AuditSeverity, default: AuditSeverity.INFO })
  severity: AuditSeverity;

  @Column({ name: 'user_id', nullable: true })
  userId: string;

  @Column({ name: 'user_email', nullable: true })
  userEmail: string;

  @Column({ name: 'provider_id', nullable: true })
  providerId: string;

  @Column({ name: 'provider_code', nullable: true })
  providerCode: string;

  @Column({ name: 'session_id', nullable: true })
  sessionId: string;

  @Column({ name: 'ip_address', length: 45 })
  ipAddress: string;

  @Column({ name: 'user_agent', length: 512, nullable: true })
  userAgent: string;

  @Column({ type: 'jsonb', nullable: true })
  details: Record<string, any>;

  @Column({ name: 'error_message', nullable: true })
  errorMessage: string;

  @Column({ name: 'previous_hash', length: 64, nullable: true })
  previousHash: string;

  @Column({ name: 'event_hash', length: 64 })
  eventHash: string;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;
}
```

---

## 4. 審計服務

### Code 實作：核心審計服務

```typescript
// src/audit/services/audit.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import * as crypto from 'crypto';
import { AuditLog, AuditEventType, AuditSeverity } from '../entities/audit-log.entity';

export interface AuditEventContext {
  tenantId: string;
  userId?: string;
  userEmail?: string;
  providerId?: string;
  providerCode?: string;
  sessionId?: string;
  ipAddress?: string;
  userAgent?: string;
  details?: Record<string, any>;
  errorMessage?: string;
}

@Injectable()
export class AuditService {
  private readonly logger = new Logger(AuditService.name);

  constructor(
    @InjectRepository(AuditLog)
    private readonly auditRepo: Repository<AuditLog>,
  ) {}

  async log(
    eventType: AuditEventType,
    severity: AuditSeverity,
    context: AuditEventContext,
  ): Promise<void> {
    try {
      // 1. 攞上一條 Hash 嚟維持 Chain 完整性
      const previousLog = await this.auditRepo.findOne({
        where: { tenantId: context.tenantId },
        order: { createdAt: 'DESC' },
        select: ['eventHash'],
      });

      // 2. 計算呢個 Event 嘅 Hash
      const eventPayload = JSON.stringify({
        eventType,
        severity,
        ...context,
        timestamp: new Date().toISOString(),
      });
      const eventHash = crypto.createHash('sha256').update(eventPayload).digest('hex');

      // 3. 新增審計日誌記錄
      const log = this.auditRepo.create({
        tenantId: context.tenantId,
        eventType,
        severity,
        userId: context.userId,
        userEmail: context.userEmail,
        providerId: context.providerId,
        providerCode: context.providerCode,
        sessionId: context.sessionId,
        ipAddress: context.ipAddress,
        userAgent: context.userAgent,
        details: context.details,
        errorMessage: context.errorMessage,
        previousHash: previousLog?.eventHash || null,
        eventHash,
      });

      await this.auditRepo.save(log);

      // 4. 非同步咁 Check 警報規則
      this.checkAlertRules(eventType, severity, context);
    } catch (error) {
      // 審計日誌絕對唔可以搞死主應用
      this.logger.error(`寫審計日誌失敗：${error.message}`, error.stack);
    }
  }

  // 便捷方法
  async logInfo(eventType: AuditEventType, context: AuditEventContext): Promise<void> {
    return this.log(eventType, AuditSeverity.INFO, context);
  }

  async logWarning(eventType: AuditEventType, context: AuditEventContext): Promise<void> {
    return this.log(eventType, AuditSeverity.WARNING, context);
  }

  async logCritical(eventType: AuditEventType, context: AuditEventContext): Promise<void> {
    return this.log(eventType, AuditSeverity.CRITICAL, context);
  }

  private checkAlertRules(
    eventType: AuditEventType,
    severity: AuditSeverity,
    context: AuditEventContext,
  ): void {
    // 非同步警報 Check — 見第六節
    if (severity === AuditSeverity.CRITICAL) {
      // this.eventEmitter.emit('audit.critical', { eventType, context });
    }
  }
}
```

---

## 5. 喺 SSO 流程加入審計

我哋必須喺 SSO 流程嘅每一個關鍵位加入審計日誌嘅 Call。

### Mermaid 圖解：SSO 流程入面嘅審計點

```mermaid
sequenceDiagram
    autonumber
    participant U as 用戶
    participant SP as Service Provider
    participant Audit as 審計服務
    participant IdP as Identity Provider

    U->>SP: 撳「用 SSO 登入」
    SP->>Audit: SSO_LOGIN_INITIATED ℹ️

    SP->>IdP: Redirect 去 IdP
    IdP->>U: 登入畫面
    U->>IdP: 入憑證
    IdP->>SP: Callback 帶 code/assertion

    alt 登入成功
        SP->>Audit: SSO_LOGIN_SUCCESS ℹ️
        SP->>Audit: SESSION_CREATED ℹ️
        SP->>U: Redirect 去 Dashboard
    else 登入失敗（Token 無效）
        SP->>Audit: TOKEN_VALIDATION_FAILED ⚠️
        SP->>U: 401 Unauthorized
    else 登入失敗（搵唔到用戶）
        SP->>Audit: SSO_LOGIN_BLOCKED ⚠️
        SP->>U: 401 搵唔到對應帳戶
    end

    Note over SP,Audit: 之後：用戶撳登出
    U->>SP: 撳「登出」
    SP->>Audit: LOGOUT_INITIATED_SP ℹ️
    SP->>SP: 炸毀 Local Session
    SP->>Audit: SESSION_DESTROYED ℹ️
    SP->>Audit: LOGOUT_SUCCESS_SP ℹ️
    SP->>IdP: Redirect 去 IdP Logout
```

### Code 實作：喺 SSO 流程加入審計

```typescript
// src/sso/services/sso-callback.service.ts（更新版，加入審計）

import { AuditService } from '../../audit/services/audit.service';
import { AuditEventType, AuditSeverity } from '../../audit/entities/audit-log.entity';

@Injectable()
export class SsoCallbackService {
  constructor(
    // ... 現有嘅 Dependencies
    private readonly auditService: AuditService,
  ) {}

  async handleCallback(providerId: string, payload: any, request: any): Promise<any> {
    const context = {
      tenantId: request.tenantId,
      providerId,
      ipAddress: request.ip,
      userAgent: request.headers['user-agent'],
    };

    try {
      // ... 現有嘅 Callback 邏輯 ...

      // 記錄成功登入
      await this.auditService.logInfo(AuditEventType.SSO_LOGIN_SUCCESS, {
        ...context,
        userId: user.id,
        userEmail: user.email,
        sessionId: session.id,
        details: {
          protocol: provider.protocolType,
          providerCode: provider.providerCode,
        },
      });

      await this.auditService.logInfo(AuditEventType.SESSION_CREATED, {
        ...context,
        userId: user.id,
        sessionId: session.id,
      });

      return { user, session };
    } catch (error) {
      // 記錄失敗登入
      const eventType = error instanceof UnauthorizedException
        ? AuditEventType.TOKEN_VALIDATION_FAILED
        : AuditEventType.SSO_LOGIN_FAILED;

      await this.auditService.logWarning(eventType, {
        ...context,
        errorMessage: error.message,
        details: {
          errorCode: error.getStatus?.() || 500,
        },
      });

      throw error;
    }
  }
}
```

---

## 6. 警報規則：偵測可疑模式

自動化警報對即時威脅偵測至關重要。

### Mermaid 圖解：警報規則引擎

```mermaid
flowchart TD
    EVENT["收到審計事件"] --> RULES["警報規則引擎"]

    RULES --> R1{"暴力破解？<br/>同一 IP 10 分鐘內<br/>> 5 次失敗"}
    RULES --> R2{"撞庫攻擊？<br/>同一用戶<br/>> 20 次失敗<br/>來自唔同 IP"}
    RULES --> R3{"異常地點？<br/>用戶首次喺<br/>新國家登入"}
    RULES --> R4{"設定篡改？<br/>非 Admin 做嘅<br/>Provider 開關"}
    RULES --> R5{"大量 Session 終止？<br/>1 分鐘內 BCL<br/>> 10 個 Sessions"}

    R1 -->|"係"| ALERT1["🚨 警報：疑似暴力破解<br/>→ 暫時封鎖 IP<br/>→ 通知 SOC 團隊"]
    R2 -->|"係"| ALERT2["🚨 警報：撞庫攻擊<br/>→ 鎖定 Account<br/>→ 通知用戶 + SOC"]
    R3 -->|"係"| ALERT3["⚠️ 警報：異常地點<br/>→ 要求 MFA<br/>→ 通知用戶"]
    R4 -->|"係"| ALERT4["🚨 警報：未經授權嘅設定變更<br/>→ 還原變更<br/>→ 通知租戶 Admin"]
    R5 -->|"係"| ALERT5["⚠️ 警報：大量 Session 終止<br/>→ 調查 IdP 狀態<br/>→ 通知 Ops 團隊"]

    style ALERT1 fill:#e74c3c,color:#fff
    style ALERT2 fill:#e74c3c,color:#fff
    style ALERT3 fill:#f39c12,color:#fff
    style ALERT4 fill:#e74c3c,color:#fff
    style ALERT5 fill:#f39c12,color:#fff
```

### Code 實作：警報規則服務

```typescript
// src/audit/services/alert-rule.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Redis } from 'ioredis';
import { AuditEventType, AuditSeverity } from '../entities/audit-log.entity';

@Injectable()
export class AlertRuleService {
  private readonly logger = new Logger(AlertRuleService.name);

  constructor(private readonly redisClient: Redis) {}

  async checkBruteForce(
    tenantId: string,
    ipAddress: string,
    eventType: AuditEventType,
  ): Promise<boolean> {
    if (eventType !== AuditEventType.TOKEN_VALIDATION_FAILED &&
        eventType !== AuditEventType.SSO_LOGIN_FAILED) {
      return false;
    }

    const key = `alert:brute:${tenantId}:${ipAddress}`;
    const count = await this.redisClient.incr(key);
    await this.redisClient.expire(key, 600); // 10 分鐘窗口

    if (count > 5) {
      this.logger.warn(`偵測到暴力破解：IP ${ipAddress} 喺租戶 ${tenantId} 失敗咗 ${count} 次`);
      return true;
    }
    return false;
  }

  async checkCredentialStuffing(
    tenantId: string,
    userId: string,
    eventType: AuditEventType,
  ): Promise<boolean> {
    if (eventType !== AuditEventType.SSO_LOGIN_FAILED) {
      return false;
    }

    const key = `alert:stuffing:${tenantId}:${userId}`;
    const count = await this.redisClient.incr(key);
    await this.redisClient.expire(key, 600);

    if (count > 20) {
      this.logger.warn(`疑似撞庫攻擊：用戶 ${userId} 失敗咗 ${count} 次`);
      return true;
    }
    return false;
  }
}
```

---

## 7. 合規報告

SOC 2 同 ISO 27001 嘅審計員需要結構化嘅報告，展示你嘅 SSO 控制措施係有效運作緊。

### Mermaid 圖解：合規報告產生流程

```mermaid
flowchart LR
    subgraph "報告請求"
        REQ["Admin / 審計員<br/>要求報告"]
        PERIOD["揀時間範圍<br/>（例如過去 90 日）"]
        TYPE["揀報告類型"]
    end

    subgraph "報告類型"
        T1["登入活動報告<br/>- 總登入數<br/>- 成功/失敗率<br/>- 按 Provider 分"]
        T2["保安事故報告<br/>- 失敗嘅驗證<br/>- Replay 嘗試<br/>- 跨租戶嘗試"]
        T3["設定變更報告<br/>- Provider 變更<br/>- 邊個改咗乜<br/>- 幾時改"]
        T4["Session 管理報告<br/>- Sessions 建立<br/>- Sessions 終止<br/>- BCL 事件"]
    end

    subgraph "數據聚合"
        QUERY["查詢 audit_logs<br/>WHERE tenant_id = X<br/>AND created_at BETWEEN Y AND Z"]
        AGGREGATE["按 event_type、<br/>severity、provider、user 聚合"]
    end

    subgraph "輸出"
        PDF["PDF 報告"]
        CSV["CSV 匯出"]
        API["JSON API Response"]
    end

    REQ --> PERIOD --> TYPE
    TYPE --> T1 & T2 & T3 & T4
    T1 & T2 & T3 & T4 --> QUERY --> AGGREGATE
    AGGREGATE --> PDF & CSV & API
```

### Code 實作：合規報告產生器

```typescript
// src/audit/services/compliance-report.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, Between } from 'typeorm';
import { AuditLog, AuditEventType, AuditSeverity } from '../entities/audit-log.entity';

export interface ComplianceReport {
  tenantId: string;
  period: { from: Date; to: Date };
  generatedAt: Date;
  summary: {
    totalLogins: number;
    successfulLogins: number;
    failedLogins: number;
    successRate: number;
    securityIncidents: number;
    configChanges: number;
    sessionsCreated: number;
    sessionsTerminated: number;
  };
  byProvider: Record<string, { logins: number; failures: number }>;
  securityEvents: Array<{
    eventType: AuditEventType;
    severity: AuditSeverity;
    timestamp: Date;
    details: any;
  }>;
  configChanges: Array<{
    eventType: AuditEventType;
    userId: string;
    timestamp: Date;
    details: any;
  }>;
}

@Injectable()
export class ComplianceReportService {
  constructor(
    @InjectRepository(AuditLog)
    private readonly auditRepo: Repository<AuditLog>,
  ) {}

  async generateReport(tenantId: string, from: Date, to: Date): Promise<ComplianceReport> {
    const logs = await this.auditRepo.find({
      where: { tenantId, createdAt: Between(from, to) },
      order: { createdAt: 'ASC' },
    });

    const loginEvents = logs.filter(l =>
      [AuditEventType.SSO_LOGIN_SUCCESS, AuditEventType.SSO_LOGIN_FAILED].includes(l.eventType)
    );
    const successful = loginEvents.filter(l => l.eventType === AuditEventType.SSO_LOGIN_SUCCESS);
    const failed = loginEvents.filter(l => l.eventType === AuditEventType.SSO_LOGIN_FAILED);
    const securityEvents = logs.filter(l => l.severity === AuditSeverity.CRITICAL);
    const configChanges = logs.filter(l => l.eventType.startsWith('PROVIDER_') || l.eventType === 'MAPPINGS_UPDATED');

    const byProvider: Record<string, { logins: number; failures: number }> = {};
    for (const log of loginEvents) {
      const code = log.providerCode || 'unknown';
      if (!byProvider[code]) byProvider[code] = { logins: 0, failures: 0 };
      if (log.eventType === AuditEventType.SSO_LOGIN_SUCCESS) byProvider[code].logins++;
      else byProvider[code].failures++;
    }

    return {
      tenantId,
      period: { from, to },
      generatedAt: new Date(),
      summary: {
        totalLogins: loginEvents.length,
        successfulLogins: successful.length,
        failedLogins: failed.length,
        successRate: loginEvents.length > 0
          ? Math.round((successful.length / loginEvents.length) * 100) : 0,
        securityIncidents: securityEvents.length,
        configChanges: configChanges.length,
        sessionsCreated: logs.filter(l => l.eventType === AuditEventType.SESSION_CREATED).length,
        sessionsTerminated: logs.filter(l =>
          [AuditEventType.SESSION_DESTROYED, AuditEventType.SESSION_TERMINATED_BCL].includes(l.eventType)
        ).length,
      },
      byProvider,
      securityEvents: securityEvents.map(l => ({
        eventType: l.eventType, severity: l.severity, timestamp: l.createdAt, details: l.details,
      })),
      configChanges: configChanges.map(l => ({
        eventType: l.eventType, userId: l.userId, timestamp: l.createdAt, details: l.details,
      })),
    };
  }
}
```

---

## 8. 日誌保留與歸檔

審計日誌必須根據合規框架保留特定嘅期限。

### Mermaid 圖解：日誌保留生命週期

```mermaid
flowchart LR
    subgraph "熱儲存（0-90 日）"
        HOT["PostgreSQL<br/>audit_logs table<br/>按月分區"]
        HOT_QUERY["快速查詢<br/>即時警報"]
    end

    subgraph "溫儲存（90 日 - 1 年）"
        WARM["PostgreSQL<br/>壓縮分區"]
        WARM_QUERY["合規報告<br/>事故調查"]
    end

    subgraph "冷儲存（1-7 年）"
        COLD["S3 / Azure Blob<br/>加密、不可變"]
        COLD_QUERY["法律發現<br/>法規審計"]
    end

    subgraph "刪除（7 年後）"
        DELETE["安全刪除<br/>密碼學擦除"]
    end

    HOT -->|"90 日"| WARM -->|"1 年"| COLD -->|"7 年"| DELETE

    style HOT fill:#27ae60,color:#fff
    style WARM fill:#f39c12,color:#fff
    style COLD fill:#3498db,color:#fff
    style DELETE fill:#e74c3c,color:#fff
```

| 合規框架 | 最低保留期 | 建議保留期 |
|---|---|---|
| **SOC 2** | 1 年 | 3 年 |
| **ISO 27001** | 3 年 | 5 年 |
| **HIPAA** | 6 年 | 7 年 |
| **GDPR** | 需要幾耐就幾耐 | 最小化，然後刪除 |

---

## 9. 數據私隱：GDPR 考量

審計日誌包含個人數據（User ID、Email、IP 地址）。喺 GDPR 之下，我哋必須喺審計要求同數據最小化之間取得平衡。

### Mermaid 圖解：審計日誌嘅 GDPR 合規

```mermaid
flowchart TD
    LOG["審計日誌記錄<br/>（包含 PII）"] --> ANONYMIZE{"保留期後<br/>匿名化？"}

    ANONYMIZE -->|"1 年後"| HASH_USER["Hash userId<br/>SHA-256(userId + salt)"]
    ANONYMIZE -->|"2 年後"| MASK_IP["遮蔽 IP 地址<br/>192.168.1.xxx"]
    ANONYMIZE -->|"3 年後"| REMOVE_EMAIL["移除 userEmail<br/>淨係留 eventType"]

    HASH_USER --> KEEP_HASH["保留 Hash 用嚟<br/>模式分析"]
    MASK_IP --> KEEP_MASKED["保留遮蔽咗嘅 IP<br/>用嚟地理分析"]
    REMOVE_EMAIL --> KEEP_MINIMAL["保留最低限度記錄<br/>用嚟合規計數"]

    style ANONYMIZE fill:#9b59b6,color:#fff
    style KEEP_MINIMAL fill:#27ae60,color:#fff
```

### Code 實作：GDPR 匿名化

```typescript
// src/audit/services/audit-retention.service.ts
@Injectable()
export class AuditRetentionService {
  constructor(
    @InjectRepository(AuditLog) private readonly auditRepo: Repository<AuditLog>,
    private readonly configService: ConfigService,
  ) {}

  @Cron(CronExpression.EVERY_DAY_AT_3AM)
  async anonymizeOldLogs(): Promise<void> {
    const salt = this.configService.get('AUDIT_ANONYMIZE_SALT');
    const oneYearAgo = new Date();
    oneYearAgo.setFullYear(oneYearAgo.getFullYear() - 1);

    // 匿名化超過 1 年嘅用戶識別符
    await this.auditRepo
      .createQueryBuilder()
      .update(AuditLog)
      .set({
        userId: () => `encode(sha256((user_id || '${salt}')::bytea), 'hex')`,
        userEmail: '***@***.***',
      })
      .where('created_at < :date', { date: oneYearAgo })
      .andWhere('user_id IS NOT NULL')
      .execute();

    this.logger.log('已匿名化超過 1 年嘅審計日誌');
  }
}
```

---

## 10. 完整 SSO 審計 Dashboard

### Mermaid 圖解：審計 Dashboard 數據流

```mermaid
flowchart LR
    subgraph "即時指標"
        RT1["活躍 Sessions<br/>（當前數量）"]
        RT2["登入率<br/>（每分鐘）"]
        RT3["失敗率<br/>（過去一小時）"]
        RT4["BCL 事件<br/>（過去 24 小時）"]
    end

    subgraph "趨勢分析"
        TR1["每日登入量"]
        TR2["Provider 使用分佈"]
        TR3["失敗原因分析"]
        TR4["地理分佈"]
    end

    subgraph "保安警報"
        AL1["進行中嘅調查"]
        AL2["被封鎖嘅 IP"]
        AL3["被鎖定嘅 Account"]
        AL4["金鑰輪換狀態"]
    end

    subgraph "API Endpoints"
        GET1["GET /audit/dashboard/realtime"]
        GET2["GET /audit/dashboard/trends?period=30d"]
        GET3["GET /audit/dashboard/security"]
        GET4["GET /audit/reports/compliance?from=&to="]
    end

    RT1 & RT2 & RT3 & RT4 --> GET1
    TR1 & TR2 & TR3 & TR4 --> GET2
    AL1 & AL2 & AL3 & AL4 --> GET3
    GET4 --> REPORT["產生 PDF / CSV"]
```

---

## 結語：完整嘅 SSO 旅程

我哋已經到達 10 集 SSO 系列嘅終點。等我哋回顧一下我哋建立嘅完整架構：

### Mermaid 圖解：完整 SSO 架構（最終版）

```mermaid
flowchart TB
    subgraph "第一集：基礎"
        P1_STRAT["策略模式"]
        P1_ENV["信封加密"]
        P1_PKCE["PKCE"]
        P1_STATE["狀態管理"]
    end

    subgraph "第二集：設定"
        P2_DISC["自動發現"]
        P2_MAP["屬性映射引擎"]
        P2_TRANSFORM["轉換引擎"]
    end

    subgraph "第三集：身份"
        P3_MATCH["用戶匹配"]
        P3_ENFORCED["ENFORCED 模式"]
        P3_PROFILE["SSO Profile 連結"]
    end

    subgraph "第 4-5 集：協定"
        P4_OIDC["OIDC：JWT + JWKS"]
        P5_SAML["SAML：XML + C14N"]
    end

    subgraph "第 6-7 集：登出"
        P6_SP["SP 發起 SLO"]
        P7_BCL["IdP 發起 BCL"]
    end

    subgraph "第八集：金鑰"
        P8_ROTATION["金鑰輪換"]
        P8_DUAL["雙證書"]
    end

    subgraph "第九集：規模"
        P9_TENANT["多租戶"]
        P9_ISOLATION["租戶隔離"]
    end

    subgraph "第十集：可見度"
        P10_AUDIT["審計日誌"]
        P10_ALERT["警報規則"]
        P10_COMPLIANCE["合規報告"]
    end

    P1_STRAT --> P4_OIDC & P5_SAML
    P1_ENV --> P8_ROTATION
    P1_PKCE --> P4_OIDC
    P2_DISC --> P4_OIDC & P5_SAML
    P2_MAP --> P3_MATCH
    P3_MATCH --> P3_PROFILE
    P4_OIDC & P5_SAML --> P6_SP & P7_BCL
    P6_SP & P7_BCL --> P10_AUDIT
    P9_TENANT --> P9_ISOLATION
    P10_AUDIT --> P10_ALERT --> P10_COMPLIANCE

    style P1_STRAT fill:#3498db,color:#fff
    style P1_ENV fill:#9b59b6,color:#fff
    style P10_AUDIT fill:#27ae60,color:#fff
```

由策略模式到信封加密，由 PKCE 到 Back-Channel Logout，由單租戶到多租戶，再由原始事件到合規報告——呢個就係完整嘅企業級 SSO 架構。

多謝你跟晒成個系列。繼續安全寫 Code！

<br><br><br>
