# Security Model

## Overview

This document outlines the comprehensive security model for the Message Queues monitoring system, covering authentication, authorization, data protection, and compliance requirements for monitoring Kafka infrastructure across multiple providers.

## Security Architecture

### Security Layers

```
Security Architecture
├── Authentication Layer
│   ├── New Relic SSO
│   ├── SAML Integration
│   ├── API Key Management
│   └── Service Authentication
├── Authorization Layer
│   ├── Role-Based Access Control (RBAC)
│   ├── Resource Permissions
│   ├── Entity-Level Security
│   └── Feature Flags
├── Data Protection Layer
│   ├── Encryption in Transit
│   ├── Encryption at Rest
│   ├── Data Masking
│   └── Audit Logging
└── Compliance Layer
    ├── SOC 2 Compliance
    ├── GDPR Requirements
    ├── Industry Standards
    └── Security Policies
```

## Authentication

### User Authentication

```typescript
interface AuthenticationConfig {
  methods: {
    sso: {
      enabled: boolean;
      provider: 'SAML' | 'OIDC' | 'OAuth2';
      configuration: SSOConfig;
    };
    apiKey: {
      enabled: boolean;
      rotation: RotationPolicy;
      scopes: string[];
    };
    mfa: {
      required: boolean;
      methods: ('totp' | 'sms' | 'push')[];
    };
  };
  session: {
    timeout: number;
    renewal: boolean;
    concurrent: boolean;
  };
}

class AuthenticationManager {
  // Authenticate user via SSO
  async authenticateSSO(samlResponse: string): Promise<AuthResult> {
    try {
      // Validate SAML assertion
      const assertion = await this.validateSAMLAssertion(samlResponse);
      
      // Extract user attributes
      const userAttributes = this.extractUserAttributes(assertion);
      
      // Create or update user
      const user = await this.findOrCreateUser(userAttributes);
      
      // Generate session
      const session = await this.createSession(user, {
        authMethod: 'SSO',
        metadata: {
          idp: assertion.issuer,
          sessionIndex: assertion.sessionIndex
        }
      });
      
      // Log authentication event
      await this.auditLog.record({
        event: 'authentication.success',
        userId: user.id,
        method: 'SSO',
        ip: this.request.ip,
        timestamp: Date.now()
      });
      
      return {
        success: true,
        user,
        session,
        permissions: await this.loadUserPermissions(user)
      };
    } catch (error) {
      await this.auditLog.record({
        event: 'authentication.failure',
        method: 'SSO',
        error: error.message,
        ip: this.request.ip,
        timestamp: Date.now()
      });
      
      throw new AuthenticationError(error.message);
    }
  }
  
  // API key authentication
  async authenticateAPIKey(apiKey: string): Promise<AuthResult> {
    const keyHash = this.hashAPIKey(apiKey);
    const keyRecord = await this.getAPIKeyRecord(keyHash);
    
    if (!keyRecord || keyRecord.revoked) {
      throw new AuthenticationError('Invalid API key');
    }
    
    if (this.isKeyExpired(keyRecord)) {
      throw new AuthenticationError('API key expired');
    }
    
    // Check rate limits
    await this.rateLimiter.check(keyRecord.id);
    
    // Update last used
    await this.updateKeyUsage(keyRecord.id);
    
    return {
      success: true,
      apiKey: keyRecord,
      permissions: keyRecord.scopes,
      rateLimit: await this.rateLimiter.getStatus(keyRecord.id)
    };
  }
}
```

### Service Authentication

```typescript
interface ServiceAuthConfig {
  kafka: {
    aws: {
      authentication: 'IAM_ROLE' | 'ACCESS_KEY';
      assumeRole?: {
        roleArn: string;
        externalId?: string;
        duration?: number;
      };
      credentials?: {
        accessKeyId: string;
        secretAccessKey: string;
        sessionToken?: string;
      };
    };
    confluent: {
      authentication: 'API_KEY';
      credentials: {
        apiKey: string;
        apiSecret: string;
      };
      schemaRegistry?: {
        url: string;
        authentication: 'BASIC' | 'API_KEY';
      };
    };
  };
  newrelic: {
    accountId: string;
    apiKey: string;
    region: 'US' | 'EU';
  };
}

class ServiceAuthManager {
  // AWS IAM role assumption
  async assumeAWSRole(config: ServiceAuthConfig['kafka']['aws']): Promise<AWSCredentials> {
    if (config.authentication !== 'IAM_ROLE') {
      throw new Error('Role assumption requires IAM_ROLE authentication');
    }
    
    const sts = new AWS.STS();
    const params = {
      RoleArn: config.assumeRole!.roleArn,
      RoleSessionName: `NewRelicKafkaMonitoring-${Date.now()}`,
      ExternalId: config.assumeRole!.externalId,
      DurationSeconds: config.assumeRole!.duration || 3600
    };
    
    const result = await sts.assumeRole(params).promise();
    
    // Store credentials securely
    await this.credentialStore.save({
      service: 'AWS_MSK',
      credentials: result.Credentials,
      expiresAt: result.Credentials.Expiration
    });
    
    return result.Credentials;
  }
  
  // Secure credential storage
  private credentialStore = {
    async save(creds: StoredCredential): Promise<void> {
      const encrypted = await this.encrypt(creds);
      await SecureVault.store(creds.service, encrypted);
    },
    
    async retrieve(service: string): Promise<StoredCredential> {
      const encrypted = await SecureVault.get(service);
      return this.decrypt(encrypted);
    },
    
    async rotate(service: string): Promise<void> {
      const current = await this.retrieve(service);
      const rotated = await this.rotateCredentials(service, current);
      await this.save(rotated);
      
      // Audit rotation
      await this.auditLog.record({
        event: 'credential.rotation',
        service,
        timestamp: Date.now()
      });
    }
  };
}
```

## Authorization

### Role-Based Access Control (RBAC)

```typescript
interface RBACModel {
  roles: Role[];
  permissions: Permission[];
  assignments: RoleAssignment[];
}

interface Role {
  id: string;
  name: string;
  description: string;
  permissions: string[];
  inherits?: string[]; // Role inheritance
}

interface Permission {
  id: string;
  resource: string;
  actions: string[];
  conditions?: PermissionCondition[];
}

class RBACManager {
  private roles: Map<string, Role> = new Map([
    ['admin', {
      id: 'admin',
      name: 'Administrator',
      description: 'Full system access',
      permissions: ['*']
    }],
    ['kafka_operator', {
      id: 'kafka_operator',
      name: 'Kafka Operator',
      description: 'Manage Kafka infrastructure',
      permissions: [
        'kafka:read',
        'kafka:write',
        'kafka:configure',
        'alerts:manage',
        'dashboards:create'
      ]
    }],
    ['kafka_viewer', {
      id: 'kafka_viewer',
      name: 'Kafka Viewer',
      description: 'Read-only access to Kafka metrics',
      permissions: [
        'kafka:read',
        'dashboards:view',
        'alerts:view'
      ]
    }],
    ['developer', {
      id: 'developer',
      name: 'Developer',
      description: 'Application developer with limited Kafka access',
      permissions: [
        'kafka:read:owned',
        'apm:read',
        'dashboards:view:shared'
      ],
      inherits: ['kafka_viewer']
    }]
  ]);
  
  // Check permission
  async hasPermission(
    userId: string,
    resource: string,
    action: string,
    context?: PermissionContext
  ): Promise<boolean> {
    const userRoles = await this.getUserRoles(userId);
    
    for (const role of userRoles) {
      if (await this.roleHasPermission(role, resource, action, context)) {
        // Log permission grant
        await this.auditLog.record({
          event: 'authorization.granted',
          userId,
          resource,
          action,
          role: role.id,
          timestamp: Date.now()
        });
        
        return true;
      }
    }
    
    // Log permission denial
    await this.auditLog.record({
      event: 'authorization.denied',
      userId,
      resource,
      action,
      timestamp: Date.now()
    });
    
    return false;
  }
  
  // Entity-level security
  async filterEntitiesByPermission(
    userId: string,
    entities: Entity[]
  ): Promise<Entity[]> {
    const permissions = await this.getUserPermissions(userId);
    
    return entities.filter(entity => {
      // Check ownership
      if (permissions.includes('kafka:read:owned')) {
        return entity.tags?.owner === userId || 
               entity.tags?.team === this.getUserTeam(userId);
      }
      
      // Check specific entity permissions
      if (permissions.includes(`kafka:read:${entity.type.toLowerCase()}`)) {
        return true;
      }
      
      // Check wildcard permissions
      if (permissions.includes('kafka:read') || permissions.includes('*')) {
        return true;
      }
      
      return false;
    });
  }
}
```

### Attribute-Based Access Control (ABAC)

```typescript
interface ABACPolicy {
  id: string;
  description: string;
  effect: 'allow' | 'deny';
  subjects: AttributeCondition[];
  resources: AttributeCondition[];
  actions: string[];
  conditions?: EnvironmentCondition[];
}

class ABACEvaluator {
  evaluate(
    subject: Subject,
    resource: Resource,
    action: string,
    environment: Environment
  ): boolean {
    const applicablePolicies = this.findApplicablePolicies(
      subject,
      resource,
      action
    );
    
    // Evaluate policies (deny takes precedence)
    let allowed = false;
    let denied = false;
    
    for (const policy of applicablePolicies) {
      if (this.evaluatePolicy(policy, subject, resource, environment)) {
        if (policy.effect === 'deny') {
          denied = true;
          break;
        } else {
          allowed = true;
        }
      }
    }
    
    return allowed && !denied;
  }
  
  private evaluatePolicy(
    policy: ABACPolicy,
    subject: Subject,
    resource: Resource,
    environment: Environment
  ): boolean {
    // Check subject attributes
    if (!this.matchAttributes(policy.subjects, subject.attributes)) {
      return false;
    }
    
    // Check resource attributes
    if (!this.matchAttributes(policy.resources, resource.attributes)) {
      return false;
    }
    
    // Check environment conditions
    if (policy.conditions) {
      if (!this.evaluateConditions(policy.conditions, environment)) {
        return false;
      }
    }
    
    return true;
  }
}
```

## Data Protection

### Encryption

```typescript
interface EncryptionConfig {
  inTransit: {
    tls: {
      minVersion: 'TLS1.2' | 'TLS1.3';
      cipherSuites: string[];
      certificates: CertificateConfig;
    };
    kafka: {
      enabled: boolean;
      protocol: 'SSL' | 'SASL_SSL';
      truststore: TruststoreConfig;
    };
  };
  atRest: {
    database: {
      enabled: boolean;
      algorithm: 'AES-256-GCM';
      keyRotation: number; // days
    };
    files: {
      enabled: boolean;
      algorithm: 'AES-256-GCM';
    };
  };
}

class EncryptionManager {
  // Encrypt sensitive data
  async encryptSensitiveData(data: any): Promise<EncryptedData> {
    const key = await this.getDataEncryptionKey();
    const iv = crypto.randomBytes(16);
    
    const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
    
    const encrypted = Buffer.concat([
      cipher.update(JSON.stringify(data), 'utf8'),
      cipher.final()
    ]);
    
    const authTag = cipher.getAuthTag();
    
    return {
      data: encrypted.toString('base64'),
      iv: iv.toString('base64'),
      authTag: authTag.toString('base64'),
      keyVersion: key.version,
      algorithm: 'AES-256-GCM'
    };
  }
  
  // TLS configuration for Kafka connections
  getKafkaTLSConfig(): KafkaTLSConfig {
    return {
      rejectUnauthorized: true,
      ca: this.loadCACertificates(),
      cert: this.loadClientCertificate(),
      key: this.loadClientKey(),
      minVersion: 'TLSv1.2',
      ciphers: [
        'ECDHE-RSA-AES256-GCM-SHA384',
        'ECDHE-RSA-AES128-GCM-SHA256',
        'ECDHE-RSA-AES256-SHA384'
      ].join(':')
    };
  }
}
```

### Data Masking

```typescript
interface DataMaskingPolicy {
  fields: MaskingRule[];
  defaultAction: 'mask' | 'show';
}

interface MaskingRule {
  fieldPath: string;
  condition?: MaskingCondition;
  maskingType: 'full' | 'partial' | 'hash' | 'tokenize';
  preserveFormat?: boolean;
}

class DataMasker {
  private policies: Map<string, DataMaskingPolicy> = new Map([
    ['kafka_credentials', {
      fields: [
        {
          fieldPath: 'password',
          maskingType: 'full'
        },
        {
          fieldPath: 'apiKey',
          maskingType: 'partial',
          preserveFormat: true
        },
        {
          fieldPath: 'connectionString',
          maskingType: 'tokenize'
        }
      ],
      defaultAction: 'show'
    }],
    ['pii_data', {
      fields: [
        {
          fieldPath: '*.email',
          maskingType: 'partial'
        },
        {
          fieldPath: '*.ipAddress',
          maskingType: 'partial'
        }
      ],
      defaultAction: 'mask'
    }]
  ]);
  
  maskData(data: any, policyName: string, userPermissions: string[]): any {
    const policy = this.policies.get(policyName);
    if (!policy) return data;
    
    // Check if user has unmask permission
    if (userPermissions.includes('data:unmask:all')) {
      return data;
    }
    
    return this.applyMaskingRules(data, policy.fields);
  }
  
  private applyMaskingRules(data: any, rules: MaskingRule[]): any {
    const masked = { ...data };
    
    for (const rule of rules) {
      const paths = this.findPaths(masked, rule.fieldPath);
      
      for (const path of paths) {
        const value = this.getValueAtPath(masked, path);
        const maskedValue = this.maskValue(value, rule);
        this.setValueAtPath(masked, path, maskedValue);
      }
    }
    
    return masked;
  }
  
  private maskValue(value: any, rule: MaskingRule): any {
    if (!value) return value;
    
    switch (rule.maskingType) {
      case 'full':
        return '********';
        
      case 'partial':
        if (typeof value === 'string') {
          const visibleLength = Math.min(4, Math.floor(value.length / 4));
          return value.substring(0, visibleLength) + '*'.repeat(value.length - visibleLength);
        }
        return '****';
        
      case 'hash':
        return crypto.createHash('sha256').update(value.toString()).digest('hex');
        
      case 'tokenize':
        return this.tokenizationService.tokenize(value);
        
      default:
        return value;
    }
  }
}
```

## Audit Logging

### Comprehensive Audit Trail

```typescript
interface AuditEvent {
  id: string;
  timestamp: number;
  eventType: string;
  userId?: string;
  sessionId?: string;
  resource?: string;
  action?: string;
  result: 'success' | 'failure';
  metadata: Record<string, any>;
  risk: 'low' | 'medium' | 'high' | 'critical';
}

class AuditLogger {
  private readonly HIGH_RISK_EVENTS = [
    'authentication.failure.repeated',
    'authorization.privilege_escalation',
    'data.export.sensitive',
    'configuration.security_change',
    'api_key.creation',
    'role.assignment.admin'
  ];
  
  async logEvent(event: Partial<AuditEvent>): Promise<void> {
    const fullEvent: AuditEvent = {
      id: generateId(),
      timestamp: Date.now(),
      result: 'success',
      risk: this.calculateRiskLevel(event),
      ...event,
      metadata: {
        ...event.metadata,
        ip: this.request?.ip,
        userAgent: this.request?.userAgent,
        correlationId: this.request?.correlationId
      }
    };
    
    // Store in audit log
    await this.persistEvent(fullEvent);
    
    // Real-time security monitoring
    if (fullEvent.risk === 'high' || fullEvent.risk === 'critical') {
      await this.securityMonitor.alert(fullEvent);
    }
    
    // Check for patterns
    await this.detectSecurityPatterns(fullEvent);
  }
  
  private async detectSecurityPatterns(event: AuditEvent): Promise<void> {
    // Failed authentication attempts
    if (event.eventType === 'authentication.failure') {
      const recentFailures = await this.getRecentEvents({
        eventType: 'authentication.failure',
        userId: event.userId,
        since: Date.now() - 300000 // 5 minutes
      });
      
      if (recentFailures.length >= 5) {
        await this.logEvent({
          eventType: 'authentication.failure.repeated',
          userId: event.userId,
          risk: 'high',
          metadata: {
            failureCount: recentFailures.length,
            action: 'account_locked'
          }
        });
        
        await this.accountManager.lockAccount(event.userId!);
      }
    }
    
    // Privilege escalation detection
    if (event.eventType === 'role.assignment') {
      const newRole = event.metadata?.newRole;
      if (this.isPrivilegedRole(newRole)) {
        await this.logEvent({
          eventType: 'authorization.privilege_escalation',
          userId: event.metadata?.targetUserId,
          risk: 'high',
          metadata: {
            assignedBy: event.userId,
            newRole
          }
        });
      }
    }
  }
}
```

### Audit Queries

```sql
-- Security event analysis
FROM AuditEvent
SELECT 
  count(*) as eventCount,
  uniqueCount(userId) as uniqueUsers,
  percentage(count(*), WHERE result = 'failure') as failureRate
WHERE risk IN ('high', 'critical')
FACET eventType
SINCE 24 hours ago

-- Failed authentication patterns
FROM AuditEvent
SELECT 
  count(*) as attempts,
  latest(metadata.ip) as lastIP,
  latest(timestamp) as lastAttempt
WHERE eventType = 'authentication.failure'
FACET userId
SINCE 1 hour ago
HAVING count(*) > 3

-- Data access audit
FROM AuditEvent
SELECT 
  userId,
  resource,
  action,
  timestamp,
  metadata.dataSize as size
WHERE eventType LIKE 'data.%'
  AND metadata.sensitive = true
SINCE 7 days ago
ORDER BY timestamp DESC
```

## Compliance

### Compliance Framework

```typescript
interface ComplianceRequirements {
  standards: {
    soc2: SOC2Requirements;
    gdpr: GDPRRequirements;
    pci: PCIRequirements;
    hipaa: HIPAARequirements;
  };
  controls: SecurityControl[];
  assessments: ComplianceAssessment[];
}

class ComplianceManager {
  // GDPR compliance
  async handleDataRequest(
    requestType: 'access' | 'deletion' | 'portability',
    userId: string
  ): Promise<DataRequestResult> {
    await this.validateDataRequest(requestType, userId);
    
    switch (requestType) {
      case 'access':
        return this.exportUserData(userId);
        
      case 'deletion':
        return this.deleteUserData(userId);
        
      case 'portability':
        return this.exportPortableData(userId);
    }
  }
  
  // Data retention
  async enforceDataRetention(): Promise<void> {
    const policies = await this.getRetentionPolicies();
    
    for (const policy of policies) {
      const expiredData = await this.findExpiredData(policy);
      
      for (const data of expiredData) {
        await this.archiveOrDelete(data, policy);
        
        await this.auditLogger.logEvent({
          eventType: 'compliance.data_retention',
          metadata: {
            policy: policy.name,
            dataType: data.type,
            action: policy.action,
            recordCount: data.count
          }
        });
      }
    }
  }
  
  // Security controls validation
  async validateSecurityControls(): Promise<ValidationReport> {
    const controls = [
      this.validateEncryption(),
      this.validateAccessControls(),
      this.validateAuditLogging(),
      this.validateDataProtection(),
      this.validateIncidentResponse()
    ];
    
    const results = await Promise.all(controls);
    
    return {
      timestamp: Date.now(),
      controls: results,
      compliant: results.every(r => r.passed),
      report: this.generateComplianceReport(results)
    };
  }
}
```

## Security Monitoring

### Real-time Security Monitoring

```typescript
class SecurityMonitor {
  private rules: SecurityRule[] = [
    {
      name: 'Unusual Access Pattern',
      condition: (event) => {
        return event.metadata?.accessCount > 1000 &&
               event.metadata?.timeWindow < 60000;
      },
      severity: 'high',
      action: 'alert'
    },
    {
      name: 'Unauthorized API Usage',
      condition: (event) => {
        return event.eventType === 'api.unauthorized' &&
               event.metadata?.attempts > 10;
      },
      severity: 'critical',
      action: 'block'
    }
  ];
  
  async monitorSecurityEvents(): Promise<void> {
    const eventStream = this.auditLogger.getEventStream();
    
    eventStream.on('event', async (event) => {
      for (const rule of this.rules) {
        if (rule.condition(event)) {
          await this.handleSecurityIncident({
            rule: rule.name,
            event,
            severity: rule.severity,
            action: rule.action
          });
        }
      }
    });
  }
  
  private async handleSecurityIncident(incident: SecurityIncident): Promise<void> {
    // Log incident
    await this.incidentLogger.log(incident);
    
    // Execute action
    switch (incident.action) {
      case 'alert':
        await this.sendSecurityAlert(incident);
        break;
        
      case 'block':
        await this.blockAccess(incident.event.userId);
        break;
        
      case 'investigate':
        await this.createInvestigationTicket(incident);
        break;
    }
    
    // Update security dashboard
    await this.updateSecurityMetrics(incident);
  }
}
```

## Best Practices

### 1. Authentication & Authorization
- Implement SSO with MFA
- Use principle of least privilege
- Regular permission audits
- API key rotation policies

### 2. Data Protection
- Encrypt all sensitive data
- Implement field-level masking
- Secure credential storage
- Regular security assessments

### 3. Monitoring & Auditing
- Comprehensive audit logging
- Real-time security monitoring
- Regular compliance validation
- Incident response procedures

### 4. Compliance
- Regular compliance assessments
- Data retention policies
- Privacy by design
- Security training programs

## Conclusion

The security model provides comprehensive protection for the Message Queues monitoring system through multiple layers of security controls. By implementing strong authentication, fine-grained authorization, data protection, and continuous monitoring, the system maintains the highest security standards while enabling effective Kafka infrastructure monitoring.