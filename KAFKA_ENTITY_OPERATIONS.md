# 📙 Kafka Entity Platform: Operations and Management

<div align="center">

![Kafka](https://img.shields.io/badge/Apache%20Kafka-231F20?style=for-the-badge&logo=apache-kafka&logoColor=white)
![New Relic](https://img.shields.io/badge/New%20Relic-008C99?style=for-the-badge&logo=new-relic&logoColor=white)
![Documentation](https://img.shields.io/badge/Part_3_of_4-Operations-orange?style=for-the-badge)

**Master configuration management, testing, operational excellence, troubleshooting, and performance**

</div>

---

## 📑 Document Series Navigation

<table>
<tr>
<td width="25%" align="center">

### [📘 Part 1](KAFKA_ENTITY_FUNDAMENTALS.md)
**Fundamentals**
- Introduction
- Platform Basics
- Core Concepts
- Entity Hierarchy
- Lifecycle & Flow

</td>
<td width="25%" align="center">

### [📗 Part 2](KAFKA_ENTITY_IMPLEMENTATION.md)
**Implementation**
- Synthesis Engine
- Golden Metrics
- Relationships
- Providers
- Dashboards

</td>
<td width="25%" align="center" bgcolor="#fff3e0">

### 📙 Part 3 (This Doc)
**Operations**
- Configuration
- Testing
- Excellence
- Troubleshooting
- Performance

</td>
<td width="25%" align="center">

### [📕 Part 4](KAFKA_ENTITY_ADVANCED.md)
**Advanced**
- Best Practices
- Integration
- Security
- Future
- Reference

</td>
</tr>
</table>

---

## 📖 Table of Contents

11. [Configuration Management](#configuration-management)
12. [Testing and Validation Framework](#testing)
13. [Operational Excellence](#operational-excellence)
14. [Troubleshooting and Debugging](#troubleshooting)
15. [Performance Optimization](#performance-optimization)

---

## 11. Configuration Management {#configuration-management}

### Entity Definition Lifecycle

```
Development → Testing → Staging → Production

1. Development:
   - Create/modify definition files
   - Local validation
   - Unit testing

2. Testing:
   - Deploy to test environment
   - Integration testing
   - Metric validation

3. Staging:
   - Performance testing
   - Multi-provider validation
   - Dashboard verification

4. Production:
   - Gradual rollout
   - Monitoring
   - Rollback capability
```

### Configuration Distribution

The platform uses a sophisticated configuration management system:

<div style="background-color: #f5f5f5; border-radius: 8px; padding: 20px; margin: 20px 0;">

```
┌─────────────────┐
│   GitHub Repo   │  (Source of truth)
└────────┬────────┘
         │ CI/CD
         v
┌─────────────────┐
│    S3 Bucket    │  (Versioned storage)
└────────┬────────┘
         │
         v
┌─────────────────┐
│   Zookeeper     │  (Configuration paths)
└────────┬────────┘
         │ Watch
         v
┌─────────────────┐
│ Entity Defs     │
│  Publisher      │  (Distribution service)
└────────┬────────┘
         │ Publish
         v
┌─────────────────┐
│  Kafka Topics   │  (Event distribution)
└─────────────────┘
```

</div>

### Configuration Categories

<table>
<tr>
<th width="25%">📁 Category</th>
<th width="75%">📝 Contents</th>
</tr>
<tr>
<td><b>SYNTHESIS</b></td>
<td>

- Entity creation rules
- Tag mappings
- Provider variations

</td>
</tr>
<tr>
<td><b>METRICS</b></td>
<td>

- Golden metrics
- Summary metrics
- Calculated fields

</td>
</tr>
<tr>
<td><b>DASHBOARDS</b></td>
<td>

- Default dashboards
- Widget configurations
- Layout templates

</td>
</tr>
<tr>
<td><b>RELATIONSHIPS</b></td>
<td>

- Relationship rules
- TTL strategies
- Condition logic

</td>
</tr>
<tr>
<td><b>CONFIGURATIONS</b></td>
<td>

- Entity settings
- Behavior flags
- Expiration rules

</td>
</tr>
<tr>
<td><b>CANDIDATES</b></td>
<td>

- Experimental features
- Beta definitions
- A/B testing

</td>
</tr>
</table>

### File Organization

```
entity-definitions/
├── entity-types/
│   ├── message-queue-cluster/
│   │   ├── definition.yml
│   │   ├── golden_metrics.yml
│   │   ├── summary_metrics.yml
│   │   ├── dashboard.json
│   │   └── tests/
│   │       ├── nri-kafka.json
│   │       ├── aws-msk.json
│   │       └── confluent.json
│   └── [other entity types...]
└── relationships/
    ├── cluster-to-broker.yml
    ├── topic-to-partition.yml
    └── [other relationships...]
```

### Definition Validation

#### Schema Validation

<div style="background-color: #e3f2fd; border-radius: 8px; padding: 20px; margin: 20px 0;">

**Required Fields:**
- `domain`: Must be valid domain
- `type`: Must be unique within domain
- `synthesis.rules`: At least one rule
- `configuration`: Required settings

**Optional Fields:**
- `goldenTags`: Recommended
- `golden_metrics`: Recommended
- `dashboard`: Recommended

</div>

#### Syntax Validation
```bash
# Local validation
./gradlew validateDefinitions

# Checks performed:
- YAML syntax
- Required fields
- Type consistency
- Cross-references
```

#### Semantic Validation
```yaml
Business Rules:
  - GUIDs must be deterministic
  - TTLs must be appropriate
  - Relationships must be bidirectional
  - Metrics must have units
```

### Version Control

#### Change Management
```yaml
Version Strategy:
  - Git for source control
  - S3 versioning for rollback
  - Feature flags for gradual rollout
  - Canary deployments

Change Process:
  1. Create feature branch
  2. Modify definitions
  3. Add/update tests
  4. PR review
  5. Automated validation
  6. Merge to main
  7. Auto-deploy to S3
```

#### Rollback Procedures

<table>
<tr>
<th width="30%">🔄 Option</th>
<th width="70%">📝 Implementation</th>
</tr>
<tr>
<td><b>S3 Version Revert</b></td>
<td>

```bash
aws s3api restore-object \
  --bucket definitions \
  --key path \
  --version-id xxx
```

</td>
</tr>
<tr>
<td><b>Zookeeper Update</b></td>
<td>

```bash
zkCli.sh set /entity-definitions/path \
  "s3://bucket/old-version"
```

</td>
</tr>
<tr>
<td><b>Feature Flag</b></td>
<td>

```bash
feature-flag set kafka-entities-enabled false
```

</td>
</tr>
<tr>
<td><b>Manual Republish</b></td>
<td>

Force republish from specific version

</td>
</tr>
</table>

### Feature Flags

```yaml
Entity-Level Flags:
  - kafka-entities-enabled: Master switch
  - kafka-msk-streams-enabled: MSK streams support
  - kafka-confluent-enabled: Confluent support

Provider-Level Flags:
  - enable-self-managed-kafka: nri-kafka
  - enable-aws-msk: AWS MSK
  - enable-confluent-cloud: Confluent

Feature-Level Flags:
  - kafka-hot-partition-detection: Beta feature
  - kafka-consumer-predictions: Experimental
```

### Configuration Hot Reload

The platform supports configuration updates without restart:

<div style="background-color: #f0f8ff; border-radius: 8px; padding: 20px; margin: 20px 0;">

**Hot Reload Process:**
1. Update S3 file
2. Update Zookeeper path
3. Publisher detects change
4. Publishes to Kafka
5. Services consume update
6. Apply new configuration

**Reload Time:**
- Detection: <1 second
- Distribution: 5-10 seconds
- Application: 10-30 seconds
- Full rollout: 1-2 minutes

</div>

### Multi-Cell Configuration

```yaml
Cell-Specific Configuration:
  - Cell name in identifier
  - Regional S3 buckets
  - Local Zookeeper clusters
  - Cross-cell replication

Global Configuration:
  - Shared S3 bucket
  - Replicated Zookeeper
  - Global feature flags
  - Consistent versions
```

### Configuration Monitoring

<table>
<tr>
<th width="30%">📊 Metric</th>
<th width="70%">🎯 Purpose</th>
</tr>
<tr>
<td><b>Configuration lag</b></td>
<td>Time between update and application</td>
</tr>
<tr>
<td><b>Update frequency</b></td>
<td>Rate of configuration changes</td>
</tr>
<tr>
<td><b>Error rates</b></td>
<td>Failed updates or validations</td>
</tr>
<tr>
<td><b>Version drift</b></td>
<td>Inconsistency across cells</td>
</tr>
</table>

**Alerts:**
- Configuration update failures
- Version mismatches
- Publishing errors
- Validation failures

### Best Practices

#### Configuration as Code
```yaml
Principles:
  - All configuration in Git
  - Automated validation
  - PR-based changes
  - Audit trail

Benefits:
  - Version history
  - Rollback capability
  - Change attribution
  - Collaboration
```

#### Testing Strategy
```yaml
Test Levels:
  1. Unit: Individual rules
  2. Integration: Full pipeline
  3. Provider: Each provider
  4. Performance: Load testing
  5. Regression: Existing entities
```

#### Documentation
```yaml
Required Documentation:
  - Change description
  - Impact analysis
  - Testing performed
  - Rollback plan
  - Monitoring plan
```

---

## 12. Testing and Validation Framework {#testing}

### Test Data Structure

Each entity type includes comprehensive test data:

```
entity-types/message-queue-cluster/tests/
├── nri-kafka.json          # Self-managed Kafka
├── aws-msk-polling.json    # AWS MSK CloudWatch
├── aws-msk-streams.json    # AWS MSK Metric Streams
└── confluent-cloud.json    # Confluent Cloud
```

### Test Data Requirements

#### Complete Event Examples
```json
{
  "eventType": "KafkaClusterSample",
  "clusterName": "test-cluster",
  "kafka.version": "3.5.0",
  "cluster.activeControllerCount": 1,
  "cluster.offlinePartitionsCount": 0,
  "cluster.underReplicatedPartitions": 0,
  "integration.name": "nri-kafka",
  "integration.version": "3.0.0",
  "accountId": 12345678,
  "timestamp": 1700000000000
}
```

#### Edge Cases Coverage
```yaml
Test Scenarios:
  - Missing optional attributes
  - Null values
  - Empty strings
  - Maximum values
  - Minimum values
  - Special characters
  - Unicode handling
```

### Validation Layers

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin: 20px 0;">

<div style="background-color: #e8f5e9; padding: 15px; border-radius: 8px;">

#### 1. Syntax Validation
```yaml
Checks:
  - Valid YAML/JSON syntax
  - Proper indentation
  - Correct data types
  - No duplicate keys
```

</div>

<div style="background-color: #e3f2fd; padding: 15px; border-radius: 8px;">

#### 2. Schema Validation  
```yaml
Rules:
  - Required fields present
  - Field types correct
  - Enum values valid
  - Pattern matching
```

</div>

<div style="background-color: #fff3e0; padding: 15px; border-radius: 8px;">

#### 3. Synthesis Validation
```yaml
Tests:
  - Rules produce valid GUIDs
  - Identifiers are unique
  - Conditions match events
  - Tags extract correctly
```

</div>

<div style="background-color: #fce4ec; padding: 15px; border-radius: 8px;">

#### 4. Metric Validation
```yaml
Verification:
  - Queries are valid NRQL
  - Event types exist
  - Attributes referenced exist
  - Aggregations appropriate
```

</div>

</div>

#### 5. Relationship Validation
```yaml
Checks:
  - Source entities exist
  - Target entities exist
  - Relationship types valid
  - TTLs appropriate
```

### Testing Tools

#### Local Testing
```bash
# Run all validations
make validate

# Test specific entity type
make validate-entity TYPE=message-queue-cluster

# Test synthesis rules
make test-synthesis

# Test metric queries
make test-metrics
```

#### Integration Testing

<div style="background-color: #f5f5f5; border-radius: 8px; padding: 20px; margin: 20px 0;">

**Test Environment:**
- Dedicated test account
- Sample data generators
- All providers configured
- Monitoring enabled

**Test Process:**
1. Deploy definitions
2. Send test events
3. Verify entity creation
4. Check relationships
5. Validate metrics
6. Test dashboards

</div>

#### Load Testing
```yaml
Scenarios:
  - High partition count (10,000+)
  - Many topics (1,000+)
  - Large consumer groups (100+)
  - Rapid updates
  - Provider switching

Metrics:
  - Entity creation time
  - Synthesis latency
  - Query performance
  - Memory usage
  - Error rates
```

### Test Data Patterns

#### Provider Variations
```json
// Self-managed
{
  "eventType": "KafkaClusterSample",
  "clusterName": "prod-kafka",
  "provider": "SELF_MANAGED"
}

// AWS MSK Polling
{
  "eventType": "AwsMskClusterSample",
  "aws.kafka.clusterArn": "arn:aws:kafka:us-east-1:123:cluster/test",
  "provider.source": "cloudwatch"
}

// AWS MSK Streams
{
  "eventType": "MetricRaw",
  "aws.Namespace": "AWS/Kafka",
  "metricStreamName": "test-stream"
}
```

#### Boundary Testing
```json
{
  "cluster.activeControllerCount": 0,    // Unhealthy
  "cluster.offlinePartitionsCount": 10,  // Critical
  "broker.cpuPercent": 99.9,             // High load
  "consumer.lag": 1000000,               // Severe lag
  "topic": "very-long-topic-name-with-special-chars-!@#$%"
}
```

### Validation Checklist

<table>
<tr>
<td width="50%">

#### Pre-Deployment
- [ ] All test files present
- [ ] Syntax validation passes
- [ ] Schema validation passes  
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Documentation updated

</td>
<td width="50%">

#### Post-Deployment
- [ ] Entities appear in UI
- [ ] Metrics populate
- [ ] Relationships form
- [ ] Dashboards render
- [ ] No errors in logs
- [ ] Performance acceptable

</td>
</tr>
</table>

### Common Validation Errors

#### Synthesis Failures

<div style="background-color: #ffebee; border-radius: 8px; padding: 15px; margin: 10px 0;">

**Error:** "No matching synthesis rule"
- **Cause:** Event doesn't match conditions
- **Fix:** Check eventType and required attributes

</div>

<div style="background-color: #ffebee; border-radius: 8px; padding: 15px; margin: 10px 0;">

**Error:** "Invalid identifier"
- **Cause:** Identifier produces invalid GUID
- **Fix:** Ensure identifier components present

</div>

<div style="background-color: #ffebee; border-radius: 8px; padding: 15px; margin: 10px 0;">

**Error:** "Duplicate entity"
- **Cause:** Multiple rules match same event
- **Fix:** Make conditions more specific

</div>

#### Metric Query Failures

```yaml
Error: "Unknown event type"
Cause: FROM clause has wrong event
Fix: Verify event type spelling

Error: "Attribute not found"
Cause: SELECT references missing field
Fix: Check attribute names in test data

Error: "Invalid aggregation"
Cause: Wrong function for data type
Fix: Use appropriate aggregation
```

#### Relationship Failures

```yaml
Error: "Source entity not found"
Cause: Source GUID doesn't exist
Fix: Ensure source entity created first

Error: "Invalid relationship type"
Cause: Unknown relationship type
Fix: Use standard relationship types

Error: "Circular relationship"
Cause: Entity relates to itself
Fix: Check identifier construction
```

### Test Automation

#### CI/CD Pipeline
```yaml
stages:
  - validate:
      - Syntax check
      - Schema validation
      - Lint rules
      
  - test:
      - Unit tests
      - Synthesis tests
      - Metric tests
      
  - integration:
      - Deploy to test
      - Run test suite
      - Validate results
      
  - deploy:
      - Update S3
      - Notify publisher
      - Monitor rollout
```

#### Automated Test Generation
```python
def generate_test_data(entity_type, provider):
    """Generate test data for entity type and provider"""
    base_event = get_base_event(entity_type, provider)
    variations = []
    
    # Normal case
    variations.append(base_event)
    
    # Edge cases
    for field in get_optional_fields(entity_type):
        # Missing field
        variant = copy.deepcopy(base_event)
        del variant[field]
        variations.append(variant)
        
        # Null field
        variant = copy.deepcopy(base_event)
        variant[field] = None
        variations.append(variant)
    
    return variations
```

### Debugging Test Failures

#### Enable Debug Logging
```yaml
Log Levels:
  - DEBUG: All processing details
  - TRACE: Include event contents
  - PROFILE: Add timing information
```

#### Common Debug Queries
```sql
-- Check if events arriving
SELECT count(*) FROM KafkaClusterSample
WHERE clusterName = 'test-cluster'
SINCE 10 minutes ago

-- Verify synthesis working
SELECT latest(entity.type), latest(entity.guid)
FROM KafkaClusterSample
WHERE clusterName = 'test-cluster'

-- Check relationship creation
SELECT count(*) FROM Relationship
WHERE source.type = 'MESSAGE_QUEUE_CLUSTER'
AND target.type = 'MESSAGE_QUEUE_BROKER'
SINCE 1 hour ago
```

#### Test Isolation
```yaml
Isolation Strategies:
  - Use unique cluster names
  - Separate test accounts
  - Time-based test data
  - Cleanup after tests
  - Tag test entities
```

---

## 13. Operational Excellence {#operational-excellence}

### Deployment Architecture

```yaml
Deployment Model:
  Platform: Kubernetes
  Pattern: Microservices
  Regions: Multi-region active-active
  Cells: Independent failure domains
```

#### Service Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: entity-synthesis-engine
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: synthesis-engine
        image: entity-synthesis:latest
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "4"
            memory: "8Gi"
```

### Monitoring Strategy

#### Key Metrics to Monitor

<div style="background-color: #e8f5e9; border-radius: 8px; padding: 20px; margin: 20px 0;">

**Entity Pipeline Metrics:**
- `entity.synthesis.rate`: Entities/second
- `entity.synthesis.latency`: Processing time
- `entity.synthesis.errors`: Failure rate
- `entity.kafka.lag`: Pipeline delay

**Kafka Entity Metrics:**
- `kafka.entities.total`: Total count
- `kafka.entities.active`: Active entities
- `kafka.entities.expired`: Cleanup rate
- `kafka.relationships.total`: Connection count

**Resource Metrics:**
- `cpu.utilization`: Service CPU
- `memory.usage`: Service memory
- `kafka.consumer.lag`: Queue depth
- `elasticsearch.indexing.rate`: Storage throughput

</div>

#### Health Monitoring

```yaml
Health Checks:
  Liveness:
    - Kafka connectivity
    - Elasticsearch connectivity
    - Memory pressure
    
  Readiness:
    - Synthesis rules loaded
    - Kafka consumers ready
    - Cache warmed
    
  Startup:
    - Configuration valid
    - Dependencies available
    - Initial state loaded
```

#### SLIs and SLOs

<table>
<tr>
<th width="40%">📊 Service Level Indicator</th>
<th width="30%">🎯 SLO</th>
<th width="30%">📏 Measurement</th>
</tr>
<tr>
<td>Entity Creation Latency</td>
<td>95% < 30 seconds</td>
<td>Time from event to entity</td>
</tr>
<tr>
<td>Entity Availability</td>
<td>99.9% uptime</td>
<td>API success rate</td>
</tr>
<tr>
<td>Relationship Accuracy</td>
<td>99% correct</td>
<td>Validation checks</td>
</tr>
</table>

**Service Level Objectives:**
- Monthly uptime: 99.9%
- Entity processing: 1M/hour
- API latency p99: <100ms
- Dashboard load time: <2s

### Capacity Planning

#### Sizing Guidelines

<div style="display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 20px; margin: 20px 0;">

<div style="background-color: #e8f5e9; padding: 15px; border-radius: 8px;">

**Small (Dev/Test)**
- Clusters: 1-5
- Brokers: 3-15
- Topics: 10-100
- Partitions: 100-1000

</div>

<div style="background-color: #fff3e0; padding: 15px; border-radius: 8px;">

**Medium**
- Clusters: 5-20
- Brokers: 15-100
- Topics: 100-1000
- Partitions: 1000-10000

</div>

<div style="background-color: #ffebee; padding: 15px; border-radius: 8px;">

**Large**
- Clusters: 20+
- Brokers: 100+
- Topics: 1000+
- Partitions: 10000+

</div>

</div>

#### Resource Requirements

```yaml
Per 1000 Entities/Hour:
  CPU: 0.5 cores
  Memory: 1 GB
  Network: 10 Mbps
  Storage: 100 MB/day

Elasticsearch Sizing:
  - 1 KB per entity document
  - 3x replication factor
  - 30-day retention
  - 20% overhead

Kafka Sizing:
  - 100 bytes per event
  - 7-day retention
  - 3x replication
  - Compression enabled
```

### Performance Tuning

#### Synthesis Optimization

```yaml
Batch Processing:
  - Batch size: 1000 events
  - Timeout: 5 seconds
  - Parallelism: CPU cores * 2

Caching Strategy:
  - Entity cache: 1 hour TTL
  - Rule cache: 24 hour TTL
  - Metric cache: 5 minute TTL
```

#### Query Optimization

```sql
-- Efficient cluster query
SELECT latest(activeControllerCount)
FROM KafkaClusterSample
WHERE entity.guid = 'xxx'  -- Use index
SINCE 5 minutes ago       -- Limit time range

-- Avoid expensive operations
-- Bad: SELECT count(*) FROM KafkaPartitionSample
-- Good: SELECT uniqueCount(partition) FROM KafkaTopicSample
```

#### Indexing Strategy

```yaml
Elasticsearch Optimization:
  - Sharding: 1 shard per 50GB
  - Replicas: 1-2 for redundancy
  - Refresh interval: 30s
  - Merge policy: Tiered
  
Index Templates:
  - Dynamic mapping: false
  - Field limits: 1000
  - Analyzed fields: Minimal
  - Doc values: Enabled
```

### Operational Procedures

#### Startup Sequence

```yaml
1. Configuration Loading:
   - Load from Zookeeper
   - Validate definitions
   - Initialize caches
   
2. Connection Establishment:
   - Kafka consumers
   - Elasticsearch clients
   - Redis connections
   
3. State Recovery:
   - Load checkpoint
   - Resume from offset
   - Warm caches
   
4. Processing Start:
   - Begin synthesis
   - Start metrics
   - Enable health checks
```

#### Shutdown Sequence

```yaml
1. Graceful Shutdown:
   - Stop accepting new work
   - Finish in-flight processing
   - Commit Kafka offsets
   
2. State Persistence:
   - Save checkpoints
   - Flush caches
   - Close connections
   
3. Cleanup:
   - Release resources
   - Log statistics
   - Signal completion
```

#### Maintenance Windows

<div style="background-color: #f0f8ff; border-radius: 8px; padding: 20px; margin: 20px 0;">

**Planned Maintenance:**
- Duration: 2 hours
- Frequency: Monthly
- Impact: Read-only mode

**Tasks:**
- Elasticsearch optimization
- Kafka rebalancing
- Definition updates
- Performance tuning

**Communication:**
- 1 week notice
- Status page updates
- Slack notifications

</div>

### Incident Response

#### Severity Levels

<table>
<tr>
<th width="20%">🚨 Level</th>
<th width="40%">📝 Description</th>
<th width="40%">⏱️ Response</th>
</tr>
<tr>
<td><b>SEV1</b></td>
<td>
Critical: Complete outage, data loss risk, security breach
</td>
<td>15 minutes</td>
</tr>
<tr>
<td><b>SEV2</b></td>
<td>
Major: Partial outage, performance degradation, feature unavailable
</td>
<td>30 minutes</td>
</tr>
<tr>
<td><b>SEV3</b></td>
<td>
Minor: Individual entity issues, delayed processing, UI problems
</td>
<td>2 hours</td>
</tr>
</table>

#### Runbook Examples

<div style="background-color: #ffebee; border-radius: 8px; padding: 20px; margin: 20px 0;">

**High Entity Lag:**
1. Check Kafka consumer lag
2. Verify synthesis engine health
3. Check Elasticsearch capacity
4. Scale if needed
5. Monitor recovery

**Synthesis Failures:**
1. Check error logs
2. Validate definitions
3. Verify event format
4. Test with sample data
5. Deploy fix

</div>

#### Post-Incident Process

```yaml
Timeline:
  - Immediate: Restore service
  - 24 hours: Initial report
  - 48 hours: Root cause analysis
  - 1 week: Action items
  - 2 weeks: Improvements deployed
  
Documentation:
  - Incident timeline
  - Impact assessment
  - Root cause
  - Remediation steps
  - Prevention measures
```

### Security Operations

#### Access Control

```yaml
Service Accounts:
  - Principle of least privilege
  - Rotate credentials quarterly
  - Audit access logs
  
API Keys:
  - Unique per service
  - Encrypted in transit
  - Monitor usage
  
Network Policies:
  - Restrict by namespace
  - Whitelist only required
  - Enable network encryption
```

#### Compliance

```yaml
Data Handling:
  - PII identification
  - Encryption at rest
  - Audit trails
  - Retention policies
  
Certifications:
  - SOC 2 Type II
  - ISO 27001
  - GDPR compliant
  - HIPAA ready
```

### Cost Optimization

#### Resource Efficiency

```yaml
Optimization Strategies:
  - Right-size instances
  - Use spot instances for batch
  - Implement auto-scaling
  - Archive old data
  
Cost Monitoring:
  - Tag all resources
  - Daily cost reports
  - Budget alerts
  - Quarterly reviews
```

#### Data Lifecycle

```yaml
Retention Policies:
  - Hot data: 7 days (SSD)
  - Warm data: 30 days (HDD)
  - Cold data: 90 days (Archive)
  - Compliance: 7 years (Glacier)
  
Optimization:
  - Compress old data
  - Downsample metrics
  - Aggregate summaries
  - Delete test data
```

---

## 14. Troubleshooting and Debugging {#troubleshooting}

### Common Issues and Solutions

#### Entities Not Appearing

**Symptoms:**
- Kafka metrics flowing but no entities in UI
- Entity count remains zero
- Search returns no results

**Diagnostic Steps:**
```sql
-- 1. Check if raw events arriving
SELECT count(*) FROM KafkaClusterSample 
WHERE clusterName IS NOT NULL 
SINCE 1 hour ago

-- 2. Verify synthesis is working
SELECT latest(entity.guid), latest(entity.type)
FROM KafkaClusterSample
WHERE clusterName = 'your-cluster'
SINCE 1 hour ago

-- 3. Check for synthesis errors
SELECT count(*) FROM NrIntegrationError
WHERE category = 'EntitySynthesis'
SINCE 1 hour ago
```

**Common Causes:**

<div style="background-color: #ffebee; border-radius: 8px; padding: 20px; margin: 20px 0;">

1. **Missing Required Attributes**
   ```yaml
   Required: eventType, clusterName, accountId
   Check: All present and non-null
   ```

2. **Wrong Event Type**
   ```yaml
   Expected: KafkaClusterSample
   Actual: KafkaCluster or kafka.cluster
   Fix: Ensure exact match
   ```

3. **Feature Flag Disabled**
   ```yaml
   Check: kafka-entities-enabled = true
   Location: Feature flag service
   ```

4. **Synthesis Rule Mismatch**
   ```yaml
   Verify: Conditions match your events
   Debug: Enable TRACE logging
   ```

</div>

#### Wrong Provider Type

**Symptoms:**
- AWS MSK showing as SELF_MANAGED
- Confluent showing as generic Kafka
- Missing provider-specific features

**Diagnostic Steps:**
```sql
-- Check provider tag
SELECT latest(provider), count(*)
FROM KafkaClusterSample, AwsMskClusterSample
WHERE entity.guid = 'your-entity-guid'
FACET eventType
```

**Solutions:**

<table>
<tr>
<th width="30%">🔧 Provider</th>
<th width="70%">✅ Required Conditions</th>
</tr>
<tr>
<td><b>AWS MSK Polling</b></td>
<td>

```yaml
- eventType: AwsMskClusterSample
- provider.source: cloudwatch
```

</td>
</tr>
<tr>
<td><b>AWS MSK Streams</b></td>
<td>

```yaml
- eventType: MetricRaw
- aws.Namespace: AWS/Kafka
- metricStreamName: present
```

</td>
</tr>
<tr>
<td><b>Confluent Cloud</b></td>
<td>

```yaml
- eventType: ConfluentCloudClusterSample
- confluent.kafka.cluster.id: present
```

</td>
</tr>
</table>

#### Missing Relationships

**Symptoms:**
- No lines in service map
- Topics not connected to clusters
- Applications not linked to Kafka

**Diagnostic Queries:**
```sql
-- Check if both entities exist
SELECT count(*) FROM Entity
WHERE type IN ('MESSAGE_QUEUE_CLUSTER', 'MESSAGE_QUEUE_TOPIC')
AND tags.kafka.cluster.name = 'your-cluster'

-- Verify relationship events
SELECT count(*) FROM Relationship
WHERE source.type = 'MESSAGE_QUEUE_CLUSTER'
AND target.type = 'MESSAGE_QUEUE_TOPIC'
SINCE 1 hour ago

-- Check APM spans
SELECT count(*) FROM Span
WHERE messaging.system = 'kafka'
AND span.kind IN ('producer', 'consumer')
SINCE 1 hour ago
```

**Common Issues:**

<div style="background-color: #f5f5f5; border-radius: 8px; padding: 20px; margin: 20px 0;">

1. **Identifier Mismatch**
   ```yaml
   Source: clusterName = "prod-kafka-01"
   Target: clusterName = "prod-kafka-1"
   Fix: Ensure exact match
   ```

2. **TTL Expiration**
   ```yaml
   Dynamic relationships: 15 minutes
   Check: Activity within TTL window
   ```

3. **Missing Span Attributes**
   ```yaml
   Required for APM:
     - messaging.system = "kafka"
     - messaging.destination.name or messaging.source.name
     - span.kind = "producer" or "consumer"
   ```

</div>

#### Metric Query Issues

**Symptoms:**
- Golden metrics show "No data"
- Dashboards empty
- Incorrect values

**Debug Process:**
```sql
-- 1. Test raw data exists
SELECT count(*) FROM YourEventType
SINCE 1 hour ago

-- 2. Check specific attributes
SELECT uniques(your.attribute.name)
FROM YourEventType
SINCE 1 hour ago

-- 3. Test the actual metric query
-- Remove entity.guid filter first
SELECT latest(`cluster.activeControllerCount`)
FROM KafkaClusterSample
WHERE clusterName = 'your-cluster'
```

**Common Fixes:**

<table>
<tr>
<th width="40%">❌ Issue</th>
<th width="60%">✅ Fix</th>
</tr>
<tr>
<td>Attribute Name Mismatch</td>
<td>

```yaml
Expected: cluster.activeControllerCount
Actual: activeControllerCount
Fix: Use correct attribute name
```

</td>
</tr>
<tr>
<td>Wrong Event Type</td>
<td>

```yaml
Query: FROM KafkaClusterSample
Actual: AwsMskClusterSample
Fix: Include all event types
```

</td>
</tr>
<tr>
<td>Time Range Issues</td>
<td>

```yaml
Default: SINCE 60 minutes ago
Sparse data: Extend to 24 hours
```

</td>
</tr>
</table>

#### High Cardinality Warnings

**Symptoms:**
- Too many partition entities
- Performance degradation
- Storage alerts

**Analysis:**
```sql
-- Count partition entities
SELECT uniqueCount(partition) as 'Partition Count'
FROM KafkaPartitionSample
WHERE clusterName = 'your-cluster'
SINCE 1 day ago

-- Identify high-partition topics
SELECT uniqueCount(partition) as 'Partitions'
FROM KafkaPartitionSample
WHERE clusterName = 'your-cluster'
FACET topic
SINCE 1 day ago
```

**Mitigation Strategies:**

<div style="background-color: #e3f2fd; border-radius: 8px; padding: 20px; margin: 20px 0;">

1. **Shorter TTL**
   ```yaml
   Partition TTL: 4 hours (vs 8 days)
   Reason: Reduce active entity count
   ```

2. **Sampling**
   ```yaml
   Sample rate: 10%
   Implementation: In integration config
   ```

3. **Disable Partition Entities**
   ```yaml
   Feature flag: kafka-partition-entities-enabled = false
   Impact: No partition-level monitoring
   ```

</div>

### Advanced Debugging

#### Enable Debug Logging

```yaml
Service Configuration:
  LOG_LEVEL: DEBUG
  LOG_FORMAT: json
  LOG_DESTINATION: stdout
  
Specific Components:
  SYNTHESIS_LOG_LEVEL: TRACE
  RELATIONSHIP_LOG_LEVEL: DEBUG
  METRICS_LOG_LEVEL: INFO
```

#### Trace Entity Creation

```json
// Debug log example
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "DEBUG",
  "service": "entity-synthesis-engine",
  "message": "Processing entity",
  "context": {
    "eventType": "KafkaClusterSample",
    "identifier": "prod-kafka",
    "guid": "MTIzNDU2Nzg...",
    "rules_evaluated": 3,
    "rules_matched": 1,
    "tags_extracted": 5
  }
}
```

#### Performance Profiling

```yaml
Profiling Metrics:
  - synthesis.rule.evaluation.time
  - entity.creation.time
  - relationship.discovery.time
  - metric.query.execution.time
  
Enable Profiling:
  ENABLE_PROFILING: true
  PROFILE_SAMPLE_RATE: 0.1
```

### Query Debugging Patterns

#### Entity Exploration
```sql
-- Find all Kafka entities
SELECT uniques(entity.type), count(*)
FROM Event
WHERE entity.type LIKE 'MESSAGE_QUEUE%'
FACET entity.type
SINCE 1 day ago

-- Inspect entity tags
SELECT latest(tags)
FROM Event
WHERE entity.guid = 'your-entity-guid'
SINCE 1 hour ago
```

#### Relationship Analysis
```sql
-- Trace relationship creation
SELECT timestamp, source.name, target.name, relationshipType
FROM Relationship
WHERE source.type = 'MESSAGE_QUEUE_CLUSTER'
OR target.type = 'MESSAGE_QUEUE_CLUSTER'
SINCE 1 hour ago
LIMIT 100
```

#### Metric Validation
```sql
-- Compare provider metrics
SELECT 
  latest(cluster.activeControllerCount) as 'nri-kafka',
  latest(aws.kafka.ActiveControllerCount) as 'aws-msk'
FROM KafkaClusterSample, AwsMskClusterSample
WHERE entity.guid = 'your-guid'
```

### Platform-Specific Debugging

#### Entity Pipeline Issues

<div style="background-color: #f5f5f5; border-radius: 8px; padding: 20px; margin: 20px 0;">

**Check Points:**
1. `entity-ingest`: Receipt confirmation
2. `entity-deduplicator`: Dedup window
3. `entity-merger`: State accumulation
4. `entity-synthesis-engine`: Rule application
5. `core-entity-indexer`: Storage

**Logs Location:**
- Kubernetes: `kubectl logs -n entity-platform`
- Centralized: Logs UI with service filter

</div>

#### Kafka Pipeline Health
```bash
# Check consumer lag
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --group entity-synthesis --describe

# Verify topic messages
kafka-console-consumer --bootstrap-server kafka:9092 \
  --topic entity-raw --from-beginning --max-messages 10
```

#### Elasticsearch Queries
```bash
# Check entity document
curl -X GET "elasticsearch:9200/entities/_doc/ENTITY_GUID"

# Search for entities
curl -X POST "elasticsearch:9200/entities/_search" -d '{
  "query": {
    "match": {
      "type": "MESSAGE_QUEUE_CLUSTER"
    }
  }
}'
```

### Emergency Procedures

#### Entity Cleanup
```yaml
When needed:
  - Incorrect entities created
  - Testing artifacts
  - Migration cleanup
  
Process:
  1. Identify entities to remove
  2. Stop synthesis for type
  3. Delete from Elasticsearch
  4. Clear caches
  5. Re-enable synthesis
```

#### Force Refresh
```yaml
Symptoms:
  - Stale data
  - Cache issues
  - Update not appearing
  
Steps:
  1. Clear Redis cache
  2. Restart synthesis engine
  3. Reindex from Kafka
  4. Verify updates
```

#### Rollback Procedures
```yaml
Definition Rollback:
  1. Identify problematic version
  2. Revert S3 to previous
  3. Update Zookeeper path
  4. Monitor republishing
  5. Verify entity health
```

---

## 15. Performance Optimization {#performance-optimization}

### Entity Pipeline Performance

#### Optimization Targets

<div style="background-color: #e8f5e9; border-radius: 8px; padding: 20px; margin: 20px 0;">

**Key Metrics:**
- Entity Creation Latency: <30 seconds
- Synthesis Throughput: >10k entities/second
- Query Response Time: <100ms p99
- Dashboard Load Time: <2 seconds

</div>

#### Batch Processing Optimization

```yaml
Optimal Batch Sizes:
  - Kafka consumption: 5000 events
  - Synthesis processing: 1000 entities
  - Elasticsearch indexing: 500 documents
  - Relationship creation: 2000 relationships

Tuning Parameters:
  kafka.consumer:
    max.poll.records: 5000
    fetch.min.bytes: 1048576
    fetch.max.wait.ms: 500
  
  synthesis.engine:
    batch.size: 1000
    batch.timeout: 5s
    parallelism: ${CPU_COUNT * 2}
  
  elasticsearch.bulk:
    actions: 500
    size: 10mb
    timeout: 30s
```

#### Memory Management

```yaml
JVM Settings:
  -Xmx8g
  -Xms8g
  -XX:MaxDirectMemorySize=2g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:G1ReservePercent=25

Memory Allocation:
  - Heap: 8GB
  - Direct: 2GB
  - OS Cache: 4GB
  - Total: 14GB per instance
```

### Query Performance

#### NRQL Optimization

```sql
-- Efficient: Use entity.guid index
SELECT latest(activeControllerCount)
FROM KafkaClusterSample
WHERE entity.guid = 'xxx'
SINCE 5 minutes ago

-- Inefficient: Full scan
SELECT latest(activeControllerCount)
FROM KafkaClusterSample
WHERE clusterName = 'prod'
SINCE 7 days ago

-- Optimized aggregation
SELECT sum(bytesInPerSec)
FROM KafkaBrokerSample
WHERE entity.guid IN ('xxx', 'yyy', 'zzz')
TIMESERIES 1 minute
SINCE 1 hour ago
```

#### Index Usage

```yaml
Indexed Fields:
  - entity.guid (primary key)
  - entity.type
  - accountId
  - timestamp
  - Golden tags

Query Patterns:
  - Always filter by entity.guid when possible
  - Use time constraints
  - Limit result sets
  - Avoid wildcards in WHERE clause
```

#### Caching Strategy

<table>
<tr>
<th width="30%">🏪 Cache Layer</th>
<th width="70%">⏱️ TTL Settings</th>
</tr>
<tr>
<td><b>CDN Cache</b></td>
<td>

- Dashboard JSON: 5 minutes
- Static assets: 24 hours

</td>
</tr>
<tr>
<td><b>API Cache</b></td>
<td>

- Entity details: 1 minute
- Metric queries: 30 seconds
- Relationship graphs: 5 minutes

</td>
</tr>
<tr>
<td><b>Application Cache</b></td>
<td>

- Synthesis rules: 24 hours
- Entity metadata: 1 hour
- Query results: 30 seconds

</td>
</tr>
<tr>
<td><b>Database Cache</b></td>
<td>

- Elasticsearch query: 60 seconds
- Field data: Unlimited

</td>
</tr>
</table>

### Elasticsearch Optimization

#### Index Settings

```json
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "refresh_interval": "30s",
    "index.translog.durability": "async",
    "index.translog.sync_interval": "30s",
    "index.merge.policy.segments_per_tier": 5,
    "index.merge.policy.max_merged_segment": "5gb"
  },
  "mappings": {
    "dynamic": false,
    "properties": {
      "entity.guid": {
        "type": "keyword",
        "index": true
      },
      "entity.type": {
        "type": "keyword",
        "index": true
      },
      "tags": {
        "type": "object",
        "enabled": false
      }
    }
  }
}
```

#### Shard Strategy

<div style="background-color: #f0f8ff; border-radius: 8px; padding: 20px; margin: 20px 0;">

**Shard Sizing:**
- Target size: 30-50 GB per shard
- Max documents: 2 billion per shard
- Calculation: Total data / 40 GB = shard count

**Shard Distribution:**
- Cluster entities: 5 shards
- Broker entities: 5 shards
- Topic entities: 10 shards
- Partition entities: 20 shards (high volume)

</div>

### Kafka Optimization

#### Topic Configuration

```yaml
Entity Topics:
  entity-raw:
    partitions: 100
    replication: 3
    retention.ms: 604800000  # 7 days
    compression.type: lz4
    min.insync.replicas: 2
    
  entity-synthesized:
    partitions: 50
    replication: 3
    retention.ms: 172800000  # 2 days
    compression.type: snappy
```

#### Consumer Optimization

```yaml
Consumer Settings:
  - enable.auto.commit: false
  - isolation.level: read_committed
  - max.poll.interval.ms: 300000
  - session.timeout.ms: 45000
  - heartbeat.interval.ms: 15000
  
Processing Strategy:
  - Manual offset management
  - Batch commit after processing
  - Parallel processing within batch
  - Circuit breaker for failures
```

### High Cardinality Optimization

#### Partition Entity Management

<div style="background-color: #ffebee; border-radius: 8px; padding: 20px; margin: 20px 0;">

**Problem:** 100k+ partition entities
**Impact:** Memory pressure, query slowdown

**Solutions:**
1. **Reduced TTL**
   - Standard: 8 days
   - Partitions: 4 hours

2. **Sampling**
   - Sample rate: 10%
   - Key partitions: 100%

3. **Aggregation**
   - Store summary at topic level
   - Query details on demand

4. **Sharding**
   - Separate index for partitions
   - Dedicated query path

</div>

#### Consumer Group Scaling

```yaml
Problem: Thousands of consumer groups
Impact: Relationship explosion

Optimization:
  1. Group filtering:
     - Ignore test groups
     - Focus on production
     
  2. Relationship batching:
     - Batch by topic
     - Single relationship per topic
     
  3. Summary entities:
     - Consumer group summaries
     - Aggregated metrics
```

### Dashboard Performance

#### Widget Optimization

```yaml
Query Guidelines:
  1. Time range:
     - Default: 1 hour
     - Maximum: 24 hours
     
  2. Data points:
     - Line charts: <1000 points
     - Tables: <100 rows
     - Heatmaps: <10000 cells
     
  3. Faceting:
     - Limit: 100 unique values
     - Use LIMIT clause
     
  4. Aggregation:
     - Pre-aggregate when possible
     - Use summary metrics
```

#### Loading Strategy

```yaml
Progressive Loading:
  1. Critical metrics first
  2. Above-fold widgets
  3. Secondary pages lazy
  4. Details on demand

Caching:
  - Widget results: 30 seconds
  - Dashboard structure: 5 minutes
  - User preferences: Session
```

### Scaling Strategies

#### Horizontal Scaling

<table>
<tr>
<th width="40%">🔧 Component</th>
<th width="30%">📈 Scaling Range</th>
<th width="30%">🎯 Triggers</th>
</tr>
<tr>
<td>entity-synthesis-engine</td>
<td>1-20 instances</td>
<td>CPU > 70%</td>
</tr>
<tr>
<td>core-entity-indexer</td>
<td>1-10 instances</td>
<td>Memory > 80%</td>
</tr>
<tr>
<td>ep-next-gen-api</td>
<td>5-50 instances</td>
<td>Queue depth > 10k</td>
</tr>
</table>

#### Vertical Scaling

```yaml
When to Scale Up:
  - Consistent CPU pressure
  - Memory constraints
  - Large batch processing
  
Instance Sizes:
  - Small: 2 CPU, 4GB RAM
  - Medium: 4 CPU, 8GB RAM
  - Large: 8 CPU, 16GB RAM
  - XLarge: 16 CPU, 32GB RAM
```

### Cost-Performance Balance

#### Resource Efficiency

```yaml
Optimization Areas:
  1. Right-sizing:
     - Monitor actual usage
     - Downsize overprovisioned
     
  2. Spot instances:
     - Batch processing
     - Non-critical workloads
     
  3. Reserved capacity:
     - Predictable workloads
     - Long-term savings
     
  4. Data lifecycle:
     - Archive old entities
     - Compress large fields
```

#### Performance Budget

<div style="background-color: #e8f5e9; border-radius: 8px; padding: 20px; margin: 20px 0;">

**Latency Budget:**
- API Gateway: 10ms
- Service Processing: 50ms
- Database Query: 30ms
- Network: 10ms
- Total: <100ms

**Resource Budget:**
- CPU per 1k entities/hour: 0.5 cores
- Memory per 1M entities: 1GB
- Storage per 1M entities: 1GB
- Network per 1M entities: 100MB

</div>

---

## 🎯 Next Steps

You've completed Part 3 of the Kafka Entity Platform documentation! You now understand:

- ✅ Configuration management and deployment strategies
- ✅ Comprehensive testing and validation approaches
- ✅ Operational excellence practices
- ✅ Troubleshooting techniques and common issues
- ✅ Performance optimization strategies

<div align="center">

### 📚 Continue Your Journey

<table>
<tr>
<td width="33%" align="center">

**[📘 Part 1: Fundamentals](KAFKA_ENTITY_FUNDAMENTALS.md)**

Review:
- Core concepts
- Platform basics
- Entity hierarchy
- Lifecycle management

</td>
<td width="33%" align="center">

**[📗 Part 2: Implementation](KAFKA_ENTITY_IMPLEMENTATION.md)**

Review:
- Synthesis engine
- Golden metrics
- Relationships
- Provider configs
- Dashboards

</td>
<td width="33%" align="center">

**[📕 Part 4: Advanced Topics](KAFKA_ENTITY_ADVANCED.md)**

Explore:
- Best practices
- Platform integration
- Security & compliance
- Future roadmap
- Complete glossary

</td>
</tr>
</table>

</div>

---

<div align="center">

### 🆘 Need Help?

[📖 Entity Platform Docs](#) • [💬 Community Forum](#) • [🐛 Report Issues](#) • [📧 Contact Support](#)

---

*Entity Platform for Kafka • Part 3 of 4 • Last Updated: January 2025*

</div>