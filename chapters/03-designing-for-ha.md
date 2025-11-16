# Chapter 4: Designing for High Availability

In Part I, you learned that **the cloud is a new computer** that requires rethinking architecture from first principles. One of the most fundamental shifts is how we approach reliability.

**Traditional mindset**: Build highly reliable components to prevent failures
**Cloud-native mindset**: Assume everything fails constantly; design the system to survive anyway

This chapter shows you how to design high-availability systems that remain operational even when individual components, entire data centers, or even whole geographic regions fail.

## Understanding High Availability in the Cloud

### What is High Availability?

High availability (HA) means your system remains operational and accessible to users even when components fail. It's measured as a percentage of uptime:

| Availability | Downtime per Year | Downtime per Month | Downtime per Week |
|--------------|-------------------|---------------------|-------------------|
| 99% (Two nines) | 3.65 days | 7.31 hours | 1.68 hours |
| 99.9% (Three nines) | 8.76 hours | 43.83 minutes | 10.08 minutes |
| 99.99% (Four nines) | 52.60 minutes | 4.38 minutes | 1.01 minutes |
| 99.999% (Five nines) | 5.26 minutes | 26.30 seconds | 6.05 seconds |
| 99.9999% (Six nines) | 31.56 seconds | 2.63 seconds | 0.61 seconds |

### The Cloud Changes the HA Game

**Traditional approach to 99.99% availability**:
- Buy enterprise-grade hardware with redundant everything
- Implement hardware RAID
- Dual power supplies, backup generators
- Expensive maintenance contracts
- **Cost**: $100,000+ per server

**Cloud approach to 99.99%+ availability**:
- Use cheap commodity instances (they WILL fail)
- Deploy across multiple availability zones
- Automatic health checks and replacement
- Load balancers distribute traffic
- **Cost**: $500-5,000/month, no upfront investment

**The paradox**: Cheaper, less reliable components + better architecture = higher reliability and lower cost.

### Cloud-Native HA: More Options, Different Priorities

The cloud fundamentally changes HA strategy in two critical ways:

#### 1. HA Through PaaS: Let the Cloud Provider Handle Redundancy

**Traditional HA**: You build and manage redundancy yourself
- Configure database replication manually
- Set up load balancer clusters
- Manage failover scripts
- Monitor and replace failed components
- Complex, expensive, error-prone

**Cloud HA**: Use PaaS services with built-in redundancy
- **RDS Multi-AZ**: Automatic failover across availability zones (you just enable it)
- **DynamoDB**: Automatic replication across 3 AZs (no configuration needed)
- **S3**: 99.999999999% (11 nines) durability automatically
- **ELB**: Distributed across multiple AZs by default
- **Lambda**: AWS handles all infrastructure HA

**The insight**: Cloud infrastructure is **natively multi-replica**. By using managed PaaS services, you avoid the complexity and cost of building HA yourself.

**Example comparison**:

```
Traditional approach (self-managed HA):
┌─────────────────────────────────────────────────┐
│ Your responsibilities:                          │
│ - Configure PostgreSQL streaming replication   │
│ - Set up keepalived for automatic failover     │
│ - Monitor replication lag                      │
│ - Handle split-brain scenarios                 │
│ - Test failover procedures regularly           │
│ - Manage backup replication                    │
│                                                 │
│ Cost: 2-3 engineers × weeks of work            │
│ Ongoing: Constant monitoring and maintenance   │
└─────────────────────────────────────────────────┘

Cloud PaaS approach (RDS Multi-AZ):
┌─────────────────────────────────────────────────┐
│ Your responsibilities:                          │
│ - Set MultiAZ: true in CloudFormation          │
│                                                 │
│ AWS handles:                                    │
│ - Automatic synchronous replication            │
│ - Automatic failover (1-2 minutes)             │
│ - Automatic backup to standby                  │
│ - Continuous monitoring                        │
│ - DNS update on failover                       │
│                                                 │
│ Cost: One line of configuration                │
│ Ongoing: Zero - AWS manages everything         │
└─────────────────────────────────────────────────┘
```

**Cost savings**: Not just money - saved engineering time, reduced operational complexity, fewer human errors.

#### 2. Data Durability > Service Availability

This is the most important insight for cloud-native HA:

**Critical principle**: Prioritize data persistence over service uptime, because **services can be recreated from code in minutes, but lost data is gone forever**.

**Why this changes everything**:

In the cloud, your entire infrastructure is defined as code. If a region fails:

```python
# Your entire application infrastructure
$ terraform apply -var="region=us-west-2"

# 5-10 minutes later: identical environment running
```

**But you CANNOT recreate lost data from code.**

**The new HA hierarchy**:

```
Priority 1: DATA DURABILITY (most critical)
├─ Database backups across regions
├─ S3 cross-region replication
├─ Point-in-time recovery enabled
└─ Can tolerate: ZERO data loss

Priority 2: DATA AVAILABILITY (very important)
├─ Database replicas for read scaling
├─ Cache layers (Redis/ElastiCache)
├─ Can tolerate: Seconds of replication lag

Priority 3: SERVICE AVAILABILITY (important, but recoverable)
├─ Multi-AZ deployments
├─ Auto-scaling groups
├─ Load balancers
└─ Can tolerate: Minutes of downtime (infrastructure can be recreated)
```

**Reality check: Do you really need sub-second RTO?**

Most businesses claim they need "99.999% availability" but when you analyze actual requirements:

| Business Type | Claimed Need | Actual Tolerance | Reality |
|--------------|--------------|------------------|---------|
| E-commerce | "Zero downtime" | 5-10 minutes | Black Friday is 1 day/year, can tolerate brief maintenance windows other times |
| SaaS Application | "Always available" | 2-5 minutes | Users will retry, support team can communicate during rare outages |
| Internal Tools | "Business critical" | 15-30 minutes | Employees can wait, work on other tasks |
| Social Media | "Real-time required" | 1-2 minutes | Users accustomed to "refresh if error" |

**The key insight**: If your business can tolerate 2-5 minutes of downtime, you don't need expensive active-active multi-region architecture. You just need:

1. **Strong data durability** (so no data loss)
2. **Infrastructure as Code** (so you can recreate services quickly)
3. **Multi-AZ deployment** (so single AZ failures don't affect you)

**Example: Practical HA strategy**

Instead of this expensive approach:
```
❌ Active-Active Multi-Region (expensive, complex)
├─ Full infrastructure in 3 regions
├─ DynamoDB Global Tables
├─ Cross-region data synchronization
├─ Complex failover logic
└─ Cost: $15,000/month for small app
```

Use this cost-effective approach:
```
✓ Multi-AZ + Fast Recovery (practical, sufficient)
├─ Primary region: Multi-AZ deployment
├─ Data: Continuous backups to S3 (cross-region)
├─ Infrastructure: All defined in Terraform/CloudFormation
├─ Recovery process:
│   1. Detect region failure (1 minute)
│   2. Terraform apply to new region (5 minutes)
│   3. Restore database from backup (3 minutes)
│   4. Update DNS to new region (2 minutes)
│   Total: ~11 minutes RTO
│   Data loss: <1 minute (RPO)
└─ Cost: $2,000/month for same app

Savings: $13,000/month (87% cost reduction)
Trade-off: 11 minutes RTO vs near-zero RTO
Question: Is 11-minute RTO acceptable for your business? (Usually: YES)
```

**When to choose expensive HA**:

Use active-active multi-region ONLY if:
- [ ] Your business genuinely loses $10,000+ per minute of downtime
- [ ] You have regulatory requirements for continuous operation
- [ ] You serve global users requiring <100ms latency everywhere
- [ ] You've confirmed 5-minute RTO is genuinely unacceptable

For everyone else: **Multi-AZ + Infrastructure as Code + Strong data backups** is sufficient and 87% cheaper.

### The Pillars of Cloud HA

High availability in the cloud rests on four pillars:

1. **Redundancy**: No single points of failure
2. **Geographic Distribution**: Spread across availability zones and regions
3. **Automated Recovery**: Self-healing systems
4. **Load Distribution**: Spread traffic intelligently

Let's explore each in detail with practical AWS implementations.

---

## Multi-Region Architecture

### Why Multi-Region?

A single AWS region contains multiple availability zones (isolated data centers). But what if an entire region fails?

**Real incidents**:
- AWS US-EAST-1 outage (2017): Major services down for hours
- Azure South Central US (2018): Cooling failure took down entire region
- GCP us-east1 (2019): Network connectivity loss

**Single-region risk**: One region failure = complete outage
**Multi-region benefit**: One region failure = automatic failover to another region

### Multi-Region Architecture Patterns

#### Pattern 1: Active-Passive (Disaster Recovery)

Primary region serves all traffic; secondary region is on standby.

```
┌─────────────────────────────────────────────────────────────┐
│                         Route 53                            │
│              (Health checks primary region)                 │
└──────────────────┬──────────────────────────┬───────────────┘
                   │                          │
                   ▼                          ▼
         ┌──────────────────┐       ┌──────────────────┐
         │  US-EAST-1       │       │  US-WEST-2       │
         │  (PRIMARY)       │       │  (STANDBY)       │
         │                  │       │                  │
         │  • Active traffic│       │  • No traffic    │
         │  • RDS primary   │──────▶│  • RDS replica   │
         │  • S3 + replica  │       │  • S3 replica    │
         └──────────────────┘       └──────────────────┘
              Normal state

         ┌──────────────────┐       ┌──────────────────┐
         │  US-EAST-1       │       │  US-WEST-2       │
         │  (FAILED) ❌     │       │  (ACTIVE) ✓      │
         │                  │       │                  │
         │  • Down          │       │  • All traffic   │
         │  • RDS down      │       │  • RDS promoted  │
         │  • Unreachable   │       │  • Serving users │
         └──────────────────┘       └──────────────────┘
              Failover state
```

**Characteristics**:
- **RTO (Recovery Time Objective)**: 5-30 minutes
- **RPO (Recovery Point Objective)**: 1-5 minutes (replication lag)
- **Cost**: Medium (standby resources + data transfer)
- **Complexity**: Low to medium

**When to use**:
- Cost-sensitive applications
- Can tolerate brief downtime
- Disaster recovery requirement

**AWS Implementation**:

```yaml
# CloudFormation template for active-passive multi-region

# PRIMARY REGION: us-east-1
Resources:
  # Application servers
  WebServerAutoScaling:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      MinSize: 3
      MaxSize: 10
      DesiredCapacity: 3
      VPCZoneIdentifier:
        - !Ref SubnetAZ1
        - !Ref SubnetAZ2
        - !Ref SubnetAZ3
      HealthCheckType: ELB
      HealthCheckGracePeriod: 300

  # Primary database
  PrimaryDatabase:
    Type: AWS::RDS::DBInstance
    Properties:
      Engine: postgres
      MultiAZ: true  # HA within region
      BackupRetentionPeriod: 7
      # Enable cross-region read replica
      ReadReplicaIdentifiers:
        - arn:aws:rds:us-west-2:123456789012:db:standby-db

  # S3 with cross-region replication
  PrimaryBucket:
    Type: AWS::S3::Bucket
    Properties:
      ReplicationConfiguration:
        Role: !GetAtt ReplicationRole.Arn
        Rules:
          - Status: Enabled
            Destination:
              Bucket: arn:aws:s3:::standby-bucket-us-west-2
              ReplicationTime:
                Status: Enabled
                Time:
                  Minutes: 15
            Priority: 1

# SECONDARY REGION: us-west-2 (deployed separately)
# - EC2 Auto Scaling Group (MinSize: 0, ready to scale)
# - RDS Read Replica (can be promoted to primary)
# - S3 replica bucket

# Route 53 health check and failover
HealthCheck:
  Type: AWS::Route53::HealthCheck
  Properties:
    Type: HTTPS
    ResourcePath: /health
    FullyQualifiedDomainName: api.example.com
    Port: 443
    RequestInterval: 30
    FailureThreshold: 3

DNSRecord:
  Type: AWS::Route53::RecordSet
  Properties:
    Name: api.example.com
    Type: A
    SetIdentifier: Primary
    Failover: PRIMARY
    HealthCheckId: !Ref HealthCheck
    AliasTarget:
      HostedZoneId: !GetAtt LoadBalancer.CanonicalHostedZoneID
      DNSName: !GetAtt LoadBalancer.DNSName

# Standby DNS record (deployed in us-west-2)
StandbyDNSRecord:
  Type: AWS::Route53::RecordSet
  Properties:
    Name: api.example.com
    Type: A
    SetIdentifier: Secondary
    Failover: SECONDARY
    # No health check - accepts traffic only when primary fails
    AliasTarget:
      HostedZoneId: !GetAtt StandbyLoadBalancer.CanonicalHostedZoneID
      DNSName: !GetAtt StandbyLoadBalancer.DNSName
```

**Failover process**:
1. Route 53 detects primary region health check failures (90 seconds)
2. DNS automatically routes to secondary region
3. Auto Scaling in secondary region scales from 0 to desired capacity (5 minutes)
4. RDS read replica promoted to primary (2-5 minutes)
5. Application fully operational in secondary region

**Total failover time**: 5-10 minutes

#### Pattern 2: Active-Active (True Multi-Region)

Both regions actively serve traffic simultaneously.

```
                    ┌─────────────────────┐
                    │     Route 53        │
                    │  (Latency-based     │
                    │   or Geolocation)   │
                    └──────┬──────┬───────┘
                           │      │
                ┌──────────┘      └──────────┐
                │                            │
                ▼                            ▼
    ┌────────────────────┐      ┌────────────────────┐
    │   US-EAST-1        │      │   EU-WEST-1        │
    │   (50% traffic)    │      │   (50% traffic)    │
    │                    │      │                    │
    │  • Active traffic  │      │  • Active traffic  │
    │  • RDS with        │◄────►│  • RDS with        │
    │    bi-directional  │      │    bi-directional  │
    │    replication     │      │    replication     │
    │  • DynamoDB Global │◄────►│  • DynamoDB Global │
    │    Tables          │      │    Tables          │
    └────────────────────┘      └────────────────────┘
```

**Characteristics**:
- **RTO**: Near-zero (automatic)
- **RPO**: Near-zero (synchronous or millisecond replication)
- **Cost**: High (full infrastructure in multiple regions)
- **Complexity**: High

**When to use**:
- Global user base
- Cannot tolerate any downtime
- Need lowest latency for all users
- Mission-critical applications

**AWS Implementation**:

```yaml
# Active-Active with DynamoDB Global Tables

# US-EAST-1 Configuration
Resources:
  # Application tier
  USEastWebServers:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      MinSize: 3
      MaxSize: 20
      DesiredCapacity: 5

  # Global Table (automatically replicates across regions)
  UsersTable:
    Type: AWS::DynamoDB::GlobalTable
    Properties:
      TableName: users
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: userId
          AttributeType: S
      KeySchema:
        - AttributeName: userId
          KeyType: HASH
      Replicas:
        - Region: us-east-1
          PointInTimeRecoverySpecification:
            PointInTimeRecoveryEnabled: true
        - Region: eu-west-1
          PointInTimeRecoverySpecification:
            PointInTimeRecoveryEnabled: true
        - Region: ap-southeast-1
          PointInTimeRecoverySpecification:
            PointInTimeRecoveryEnabled: true

  # Aurora Global Database (for relational data)
  GlobalAuroraCluster:
    Type: AWS::RDS::GlobalCluster
    Properties:
      GlobalClusterIdentifier: global-app-cluster
      Engine: aurora-postgresql
      EngineVersion: 14.6

  PrimaryCluster:
    Type: AWS::RDS::DBCluster
    Properties:
      Engine: aurora-postgresql
      GlobalClusterIdentifier: !Ref GlobalAuroraCluster
      DBClusterIdentifier: primary-cluster-us-east-1

  SecondaryCluster:
    Type: AWS::RDS::DBCluster
    Properties:
      Engine: aurora-postgresql
      GlobalClusterIdentifier: !Ref GlobalAuroraCluster
      DBClusterIdentifier: secondary-cluster-eu-west-1
      # Read-only, can be promoted if primary fails

# Route 53 Latency-Based Routing
DNSRecord:
  Type: AWS::Route53::RecordSet
  Properties:
    Name: api.example.com
    Type: A
    SetIdentifier: US-EAST-1
    Region: us-east-1
    AliasTarget:
      HostedZoneId: !GetAtt USLoadBalancer.CanonicalHostedZoneID
      DNSName: !GetAtt USLoadBalancer.DNSName

EUDNSRecord:
  Type: AWS::Route53::RecordSet
  Properties:
    Name: api.example.com
    Type: A
    SetIdentifier: EU-WEST-1
    Region: eu-west-1
    AliasTarget:
      HostedZoneId: !GetAtt EULoadBalancer.CanonicalHostedZoneID
      DNSName: !GetAtt EULoadBalancer.DNSName
```

**How it works**:
1. **User in New York** → Route 53 sends to US-EAST-1 (lower latency)
2. **User in London** → Route 53 sends to EU-WEST-1 (lower latency)
3. **DynamoDB Global Tables** → Writes in US replicate to EU in <1 second
4. **Aurora Global Database** → Writes in primary region replicate in <1 second
5. **If US-EAST-1 fails** → All traffic automatically routes to EU-WEST-1

#### Pattern 3: Multi-Region with Regional Affinity

Users "stick" to their home region, but can fail over to another region.

**Best for**: Applications with data sovereignty requirements or regional user bases

```python
# Python example: Regional user routing

import boto3
from geopy.distance import geodesic

# Define regions with their coordinates
REGIONS = {
    'us-east-1': {'lat': 38.13, 'lon': -78.45, 'endpoint': 'https://us-east-api.example.com'},
    'eu-west-1': {'lat': 53.35, 'lon': -6.26, 'endpoint': 'https://eu-west-api.example.com'},
    'ap-southeast-1': {'lat': 1.29, 'lon': 103.85, 'endpoint': 'https://ap-se-api.example.com'}
}

def get_nearest_region(user_lat, user_lon):
    """Find nearest healthy region for user"""
    user_location = (user_lat, user_lon)

    # Calculate distances to all regions
    distances = {}
    for region, config in REGIONS.items():
        region_location = (config['lat'], config['lon'])
        distance = geodesic(user_location, region_location).kilometers

        # Check region health
        if is_region_healthy(region):
            distances[region] = distance

    # Return nearest healthy region
    nearest_region = min(distances, key=distances.get)
    return REGIONS[nearest_region]['endpoint']

def is_region_healthy(region):
    """Check if region is healthy via Route 53 health check"""
    route53 = boto3.client('route53')
    # Check health check status for region
    # (Implementation details omitted)
    return True  # Simplified
```

### Multi-Region Data Strategy

The hardest part of multi-region architecture is managing data consistency.

#### Strategy 1: Global Database (DynamoDB Global Tables)

**Best for**: NoSQL workloads requiring global consistency

```python
# Write to DynamoDB - automatically replicates globally

import boto3

dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
table = dynamodb.Table('users')  # Global Table

# Write in US-EAST-1
table.put_item(
    Item={
        'userId': 'user123',
        'email': 'user@example.com',
        'createdAt': '2024-01-15T10:30:00Z',
        'region': 'us-east-1'
    }
)

# Within 1 second, this data is available in:
# - eu-west-1
# - ap-southeast-1
# All regions can read and write independently
```

**Characteristics**:
- Multi-master (all regions can write)
- Eventual consistency (typically <1 second)
- Automatic conflict resolution (last-writer-wins)
- No additional code required

#### Strategy 2: Aurora Global Database

**Best for**: Relational workloads with primary region

```sql
-- Primary region (us-east-1): Read and Write
INSERT INTO orders (order_id, user_id, total)
VALUES ('ORD123', 'user456', 99.99);

-- Secondary region (eu-west-1): Read-only
SELECT * FROM orders WHERE user_id = 'user456';
-- Gets replicated data in <1 second

-- If primary region fails, promote secondary:
-- aws rds failover-global-cluster --global-cluster-id global-app-cluster
-- Secondary becomes primary in ~1 minute
```

**Characteristics**:
- Single-master (one primary region for writes)
- Up to 5 secondary regions (read-only)
- <1 second replication lag
- Can promote secondary to primary in failover

#### Strategy 3: S3 Cross-Region Replication

**Best for**: Static assets, backups, user uploads

```yaml
# S3 Bucket Configuration

SourceBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: app-assets-us-east-1
    VersioningConfiguration:
      Status: Enabled  # Required for replication
    ReplicationConfiguration:
      Role: !GetAtt ReplicationRole.Arn
      Rules:
        - Id: ReplicateToEU
          Status: Enabled
          Priority: 1
          Filter:
            Prefix: ''  # Replicate everything
          Destination:
            Bucket: arn:aws:s3:::app-assets-eu-west-1
            # Optional: Different storage class in destination
            StorageClass: STANDARD_IA
            ReplicationTime:
              Status: Enabled
              Time:
                Minutes: 15  # 99.99% replicated within 15 mins
```

**Upload in us-east-1**:
```bash
aws s3 cp image.jpg s3://app-assets-us-east-1/images/
```

**Available in eu-west-1** within seconds:
```bash
aws s3 ls s3://app-assets-eu-west-1/images/
# Shows: image.jpg
```

### Multi-Region Cost Optimization

Multi-region infrastructure is expensive. Here's how to optimize:

**1. Smart Traffic Distribution**

```python
# Route 53 weighted routing - send less traffic to expensive regions

# 70% to us-east-1 (cheapest)
{
    "Name": "api.example.com",
    "Type": "A",
    "SetIdentifier": "US-Primary",
    "Weight": 70,
    "AliasTarget": {
        "DNSName": "us-east-lb.amazonaws.com"
    }
}

# 20% to eu-west-1 (moderate cost)
{
    "Name": "api.example.com",
    "Type": "A",
    "SetIdentifier": "EU-Secondary",
    "Weight": 20,
    "AliasTarget": {
        "DNSName": "eu-west-lb.amazonaws.com"
    }
}

# 10% to ap-southeast-1 (expensive)
{
    "Name": "api.example.com",
    "Type": "A",
    "SetIdentifier": "APAC-Tertiary",
    "Weight": 10,
    "AliasTarget": {
        "DNSName": "apac-lb.amazonaws.com"
    }
}
```

**2. Different Instance Sizes per Region**

```yaml
# Primary region: Full capacity
USEastASG:
  MinSize: 10
  MaxSize: 50
  InstanceType: t3.large

# Secondary region: Reduced capacity
EUWestASG:
  MinSize: 3
  MaxSize: 20
  InstanceType: t3.medium  # Smaller instance
```

**3. On-Demand vs Reserved Instances**

```
Primary region (us-east-1):
- 10x Reserved Instances (1-year) → 35% discount
- Auto-scaling on On-Demand

Secondary regions:
- All On-Demand (don't pay for unused capacity)
```

---

## Load Balancing Strategies

Load balancers are critical for high availability - they distribute traffic and route around failures.

### Types of Load Balancers in AWS

#### 1. Application Load Balancer (ALB) - Layer 7

**Best for**: HTTP/HTTPS web applications, microservices

```yaml
ApplicationLoadBalancer:
  Type: AWS::ElasticLoadBalancingV2::LoadBalancer
  Properties:
    Name: web-app-alb
    Scheme: internet-facing
    Type: application
    Subnets:
      - !Ref PublicSubnetAZ1
      - !Ref PublicSubnetAZ2
      - !Ref PublicSubnetAZ3  # Spread across 3 AZs
    SecurityGroups:
      - !Ref ALBSecurityGroup

# HTTP Listener
HTTPListener:
  Type: AWS::ElasticLoadBalancingV2::Listener
  Properties:
    LoadBalancerArn: !Ref ApplicationLoadBalancer
    Port: 80
    Protocol: HTTP
    DefaultActions:
      # Redirect HTTP to HTTPS
      - Type: redirect
        RedirectConfig:
          Protocol: HTTPS
          Port: 443
          StatusCode: HTTP_301

# HTTPS Listener with routing rules
HTTPSListener:
  Type: AWS::ElasticLoadBalancingV2::Listener
  Properties:
    LoadBalancerArn: !Ref ApplicationLoadBalancer
    Port: 443
    Protocol: HTTPS
    Certificates:
      - CertificateArn: !Ref SSLCertificate
    DefaultActions:
      - Type: forward
        TargetGroupArn: !Ref WebTargetGroup

# Path-based routing
APIRule:
  Type: AWS::ElasticLoadBalancingV2::ListenerRule
  Properties:
    ListenerArn: !Ref HTTPSListener
    Priority: 1
    Conditions:
      - Field: path-pattern
        Values:
          - /api/*
    Actions:
      - Type: forward
        TargetGroupArn: !Ref APITargetGroup

# Host-based routing
AdminRule:
  Type: AWS::ElasticLoadBalancingV2::ListenerRule
  Properties:
    ListenerArn: !Ref HTTPSListener
    Priority: 2
    Conditions:
      - Field: host-header
        Values:
          - admin.example.com
    Actions:
      - Type: forward
        TargetGroupArn: !Ref AdminTargetGroup

# Target Group with health checks
WebTargetGroup:
  Type: AWS::ElasticLoadBalancingV2::TargetGroup
  Properties:
    Name: web-servers
    Port: 80
    Protocol: HTTP
    VpcId: !Ref VPC
    HealthCheckEnabled: true
    HealthCheckProtocol: HTTP
    HealthCheckPath: /health
    HealthCheckIntervalSeconds: 30
    HealthCheckTimeoutSeconds: 5
    HealthyThresholdCount: 2
    UnhealthyThresholdCount: 3
    TargetType: instance
    DeregistrationDelay: 30  # Drain connections for 30s before removing
```

**ALB Features**:
- **Path-based routing**: /api/* → API servers, /admin/* → Admin servers
- **Host-based routing**: api.example.com → API, www.example.com → Web
- **HTTP/2 and WebSocket support**
- **WAF integration**: Protect against attacks
- **Authentication**: Built-in OIDC/SAML auth

#### 2. Network Load Balancer (NLB) - Layer 4

**Best for**: TCP/UDP traffic, ultra-high performance, static IPs

```yaml
NetworkLoadBalancer:
  Type: AWS::ElasticLoadBalancingV2::LoadBalancer
  Properties:
    Name: app-nlb
    Type: network
    Scheme: internet-facing
    Subnets:
      - !Ref PublicSubnetAZ1
      - !Ref PublicSubnetAZ2
      - !Ref PublicSubnetAZ3
    # NLB automatically gets static IPs in each subnet

TCPListener:
  Type: AWS::ElasticLoadBalancingV2::Listener
  Properties:
    LoadBalancerArn: !Ref NetworkLoadBalancer
    Port: 3306  # MySQL
    Protocol: TCP
    DefaultActions:
      - Type: forward
        TargetGroupArn: !Ref DatabaseProxyTargetGroup

DatabaseProxyTargetGroup:
  Type: AWS::ElasticLoadBalancingV2::TargetGroup
  Properties:
    Port: 3306
    Protocol: TCP
    VpcId: !Ref VPC
    HealthCheckEnabled: true
    HealthCheckProtocol: TCP
    HealthCheckIntervalSeconds: 10
    HealthyThresholdCount: 2
    UnhealthyThresholdCount: 2
    # Preserve source IP
    TargetGroupAttributes:
      - Key: preserve_client_ip.enabled
        Value: true
```

**NLB Characteristics**:
- **Performance**: Millions of requests/second, ultra-low latency
- **Static IPs**: Can assign Elastic IPs (required for whitelisting)
- **Preserve source IP**: See real client IP
- **TLS termination**: Can terminate SSL/TLS

**When to use NLB**:
- Non-HTTP protocols (database connections, game servers, IoT)
- Need static IPs for firewall rules
- Extreme performance requirements (>100k req/s per instance)

#### 3. Gateway Load Balancer (GWLB)

**Best for**: Virtual appliances (firewalls, IDS/IPS, deep packet inspection)

```yaml
GatewayLoadBalancer:
  Type: AWS::ElasticLoadBalancingV2::LoadBalancer
  Properties:
    Type: gateway
    Subnets:
      - !Ref SecuritySubnetAZ1
      - !Ref SecuritySubnetAZ2

# Use case: Route all traffic through security appliances
GWLBEndpoint:
  Type: AWS::EC2::VPCEndpointService
  Properties:
    GatewayLoadBalancerArns:
      - !Ref GatewayLoadBalancer
    AcceptanceRequired: false
```

### Advanced Load Balancing Patterns

#### Pattern 1: Blue/Green Deployments

Zero-downtime deployments by switching traffic between two environments.

```yaml
# Blue environment (current production)
BlueTargetGroup:
  Type: AWS::ElasticLoadBalancingV2::TargetGroup
  Properties:
    Name: web-blue
    Port: 80
    Protocol: HTTP
    VpcId: !Ref VPC

# Green environment (new version)
GreenTargetGroup:
  Type: AWS::ElasticLoadBalancingV2::TargetGroup
  Properties:
    Name: web-green
    Port: 80
    Protocol: HTTP
    VpcId: !Ref VPC

# Initially, all traffic goes to Blue
Listener:
  Type: AWS::ElasticLoadBalancingV2::Listener
  Properties:
    LoadBalancerArn: !Ref ALB
    Port: 80
    Protocol: HTTP
    DefaultActions:
      - Type: forward
        TargetGroupArn: !Ref BlueTargetGroup

# Deployment process:
# 1. Deploy new version to Green target group
# 2. Test Green environment
# 3. Switch listener to Green (instant cutover)
# 4. Monitor for issues
# 5. If problems, switch back to Blue instantly
```

**Switch traffic to Green**:
```bash
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/web-alb/50dc6c495c0c9188/f2f7dc8efc522ab2 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-green/73e2d6bc24d8a067
```

**Rollback** (if issues detected):
```bash
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:listener/app/web-alb/50dc6c495c0c9188/f2f7dc8efc522ab2 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/web-blue/73e2d6bc24d8a067
```

**Result**: Instant rollback with zero downtime.

#### Pattern 2: Canary Deployments

Gradually shift traffic to new version while monitoring.

```yaml
Listener:
  Type: AWS::ElasticLoadBalancingV2::Listener
  Properties:
    LoadBalancerArn: !Ref ALB
    Port: 80
    Protocol: HTTP
    DefaultActions:
      # Forward to both target groups with weights
      - Type: forward
        ForwardConfig:
          TargetGroups:
            - TargetGroupArn: !Ref StableTargetGroup
              Weight: 95  # 95% to stable version
            - TargetGroupArn: !Ref CanaryTargetGroup
              Weight: 5   # 5% to new version
          TargetGroupStickinessConfig:
            Enabled: true
            DurationSeconds: 3600
```

**Deployment process**:
```
Phase 1: 95% stable, 5% canary   → Monitor for 1 hour
Phase 2: 80% stable, 20% canary  → Monitor for 1 hour
Phase 3: 50% stable, 50% canary  → Monitor for 1 hour
Phase 4: 0% stable, 100% canary  → Full rollout
```

**Automated canary with monitoring**:
```python
import boto3
import time

elb = boto3.client('elbv2')
cloudwatch = boto3.client('cloudwatch')

def canary_deployment(listener_arn, stable_tg, canary_tg):
    """Gradually shift traffic to canary with automatic rollback"""

    weights = [
        (95, 5),   # 5% canary
        (80, 20),  # 20% canary
        (50, 50),  # 50% canary
        (0, 100)   # 100% canary
    ]

    for stable_weight, canary_weight in weights:
        print(f"Shifting to {canary_weight}% canary...")

        # Update weights
        elb.modify_listener(
            ListenerArn=listener_arn,
            DefaultActions=[{
                'Type': 'forward',
                'ForwardConfig': {
                    'TargetGroups': [
                        {'TargetGroupArn': stable_tg, 'Weight': stable_weight},
                        {'TargetGroupArn': canary_tg, 'Weight': canary_weight}
                    ]
                }
            }]
        )

        # Wait and monitor
        time.sleep(3600)  # 1 hour

        # Check error rate
        error_rate = get_error_rate(canary_tg)

        if error_rate > 0.05:  # 5% error threshold
            print(f"Error rate too high ({error_rate}), rolling back!")
            rollback(listener_arn, stable_tg)
            return False

        print(f"Phase successful, error rate: {error_rate}")

    print("Canary deployment complete!")
    return True

def get_error_rate(target_group_arn):
    """Get error rate from CloudWatch"""
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/ApplicationELB',
        MetricName='HTTPCode_Target_5XX_Count',
        Dimensions=[{'Name': 'TargetGroup', 'Value': target_group_arn}],
        StartTime=datetime.now() - timedelta(minutes=15),
        EndTime=datetime.now(),
        Period=900,
        Statistics=['Sum']
    )
    # Calculate error rate (simplified)
    return 0.02  # Example: 2% error rate

def rollback(listener_arn, stable_tg):
    """Instant rollback to stable version"""
    elb.modify_listener(
        ListenerArn=listener_arn,
        DefaultActions=[{
            'Type': 'forward',
            'TargetGroupArn': stable_tg
        }]
    )
```

#### Pattern 3: Sticky Sessions

Keep user sessions on the same server (when needed).

```yaml
TargetGroup:
  Type: AWS::ElasticLoadBalancingV2::TargetGroup
  Properties:
    Name: web-sticky
    Port: 80
    Protocol: HTTP
    VpcId: !Ref VPC
    # Enable stickiness
    TargetGroupAttributes:
      - Key: stickiness.enabled
        Value: true
      - Key: stickiness.type
        Value: lb_cookie  # ALB-generated cookie
      - Key: stickiness.lb_cookie.duration_seconds
        Value: 86400  # 24 hours
```

**When to use**:
- Legacy applications with server-side sessions
- WebSocket connections
- File upload sessions

**Better alternative**: Use stateless architecture with external session storage (Redis/DynamoDB)

### Load Balancer Health Checks

Health checks are crucial - they determine which instances receive traffic.

```yaml
TargetGroup:
  Type: AWS::ElasticLoadBalancingV2::TargetGroup
  Properties:
    HealthCheckEnabled: true
    HealthCheckProtocol: HTTP
    HealthCheckPath: /health
    HealthCheckIntervalSeconds: 30    # Check every 30 seconds
    HealthCheckTimeoutSeconds: 5      # Timeout after 5 seconds
    HealthyThresholdCount: 2          # 2 successes = healthy
    UnhealthyThresholdCount: 3        # 3 failures = unhealthy
    Matcher:
      HttpCode: 200  # Accept only 200 OK
```

**Implement a health check endpoint**:

```python
from flask import Flask, jsonify
import psycopg2

app = Flask(__name__)

@app.route('/health')
def health_check():
    """
    Health check endpoint - verifies application is fully functional
    Returns 200 if healthy, 500 if unhealthy
    """
    try:
        # Check database connection
        conn = psycopg2.connect(
            host=os.environ['DB_HOST'],
            database=os.environ['DB_NAME'],
            user=os.environ['DB_USER'],
            password=os.environ['DB_PASSWORD']
        )
        cursor = conn.cursor()
        cursor.execute('SELECT 1')
        cursor.close()
        conn.close()

        # Check Redis connection
        redis_client = redis.Redis(host=os.environ['REDIS_HOST'])
        redis_client.ping()

        # All checks passed
        return jsonify({
            'status': 'healthy',
            'database': 'ok',
            'cache': 'ok'
        }), 200

    except Exception as e:
        # Health check failed - LB will remove this instance
        return jsonify({
            'status': 'unhealthy',
            'error': str(e)
        }), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

**What happens when health check fails**:
1. Instance fails 3 consecutive checks (90 seconds)
2. Load balancer marks instance unhealthy
3. Load balancer stops sending traffic to instance
4. Instance remains in target group but receives no traffic
5. Health checks continue
6. When 2 consecutive checks pass, instance is marked healthy again
7. Load balancer resumes sending traffic

---

## Database Replication and Failover

Databases are often the most critical component - and the hardest to make highly available.

### RDS Multi-AZ: High Availability Within a Region

**How Multi-AZ works**:

```
┌──────────────────────────────────────────────────────────┐
│                      Application                         │
│              (connects to single endpoint)               │
└────────────────────┬─────────────────────────────────────┘
                     │
                     ▼
         ┌────────────────────────┐
         │   RDS Endpoint         │
         │  mydb.123.us-east-1    │
         │  .rds.amazonaws.com    │
         └───────────┬────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
┌──────────────┐          ┌──────────────┐
│  us-east-1a  │          │  us-east-1b  │
│   PRIMARY    │          │  STANDBY     │
│   ┌──────┐   │          │   ┌──────┐   │
│   │ DB   │   │──Sync───▶│   │ DB   │   │
│   │      │   │ Repl     │   │      │   │
│   └──────┘   │          │   └──────┘   │
│   Active     │          │   Passive    │
└──────────────┘          └──────────────┘

Normal operation: Primary serves all traffic
```

**Failover scenario**:

```
Primary AZ fails ❌

┌──────────────┐          ┌──────────────┐
│  us-east-1a  │          │  us-east-1b  │
│   PRIMARY    │          │  STANDBY     │
│   ┌──────┐   │          │   ┌──────┐   │
│   │  ❌  │   │          │   │ DB   │   │
│   │      │   │          │   │      │   │
│   └──────┘   │          │   └──────┘   │
│   FAILED     │          │  Promoting... │
└──────────────┘          └──────────────┘

DNS automatically updates to point to standby (1-2 minutes)

┌──────────────┐          ┌──────────────┐
│  us-east-1a  │          │  us-east-1b  │
│   (Failed)   │          │  PRIMARY ✓   │
│              │          │   ┌──────┐   │
│              │          │   │ DB   │   │
│              │◀─Sync────│   │      │   │
│              │  Repl    │   └──────┘   │
│              │          │   Active     │
└──────────────┘          └──────────────┘

New standby created in us-east-1a
```

**Configure Multi-AZ**:

```yaml
Database:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceIdentifier: myapp-db
    Engine: postgres
    EngineVersion: 14.7
    DBInstanceClass: db.t3.medium
    AllocatedStorage: 100
    StorageType: gp3
    MultiAZ: true  # ← This enables automatic failover
    BackupRetentionPeriod: 7
    PreferredBackupWindow: "03:00-04:00"
    PreferredMaintenanceWindow: "mon:04:00-mon:05:00"
    PubliclyAccessible: false
    VPCSecurityGroups:
      - !Ref DatabaseSecurityGroup
    DBSubnetGroupName: !Ref DBSubnetGroup

DBSubnetGroup:
  Type: AWS::RDS::DBSubnetGroup
  Properties:
    DBSubnetGroupDescription: Subnets for RDS
    SubnetIds:
      - !Ref PrivateSubnetAZ1  # us-east-1a
      - !Ref PrivateSubnetAZ2  # us-east-1b
      - !Ref PrivateSubnetAZ3  # us-east-1c
```

**Characteristics**:
- **RTO**: 1-2 minutes (automatic)
- **RPO**: Zero (synchronous replication)
- **Cost**: ~2x single-AZ (you pay for both instances)
- **Performance impact**: Minimal (synchronous writes add ~1-2ms latency)

### Aurora: Cloud-Native Database

Aurora is AWS's cloud-native database - built from the ground up for the cloud.

**Aurora architecture**:

```
                    Application
                        │
                        ▼
            ┌───────────────────────┐
            │   Aurora Endpoint     │
            └───────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
    ┌────────┐      ┌────────┐      ┌────────┐
    │Writer  │      │Reader  │      │Reader  │
    │Instance│      │Instance│      │Instance│
    │(Master)│      │(Replica│      │(Replica│
    └────┬───┘      └────┬───┘      └────┬───┘
         │               │               │
         └───────────────┼───────────────┘
                         │
         ┌───────────────┴───────────────┐
         │      Aurora Storage Layer     │
         │    (6 copies across 3 AZs)    │
         │                               │
         │  ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐│
         │  │  │ │  │ │  │ │  │ │  │ │  ││
         │  └──┘ └──┘ └──┘ └──┘ └──┘ └──┘│
         │  AZ1    AZ1   AZ2   AZ2  AZ3 AZ3│
         └───────────────────────────────┘
```

**Key differences from regular RDS**:
- **Storage**: Automatically distributes data across 6 copies in 3 AZs
- **Failover**: <30 seconds (vs 1-2 minutes for RDS Multi-AZ)
- **Read replicas**: Up to 15 low-latency replicas
- **Storage auto-scaling**: Grows automatically from 10GB to 128TB
- **Backtrack**: Rewind database to any point in time (no restore needed)

**Create Aurora cluster**:

```yaml
AuroraCluster:
  Type: AWS::RDS::DBCluster
  Properties:
    Engine: aurora-postgresql
    EngineVersion: 14.6
    DatabaseName: myapp
    MasterUsername: admin
    MasterUserPassword: !Ref DBPassword
    BackupRetentionPeriod: 7
    PreferredBackupWindow: "03:00-04:00"
    PreferredMaintenanceWindow: "mon:04:00-mon:05:00"
    VpcSecurityGroupIds:
      - !Ref DatabaseSecurityGroup
    DBSubnetGroupName: !Ref DBSubnetGroup
    EnableCloudwatchLogsExports:
      - postgresql
    # Enable backtrack (PostgreSQL doesn't support this, MySQL does)
    # BacktrackWindow: 72  # 72 hours for MySQL

# Writer instance
WriterInstance:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceClass: db.r6g.large
    Engine: aurora-postgresql
    DBClusterIdentifier: !Ref AuroraCluster
    PubliclyAccessible: false

# Read replicas (auto-scaling)
ReaderInstance1:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceClass: db.r6g.large
    Engine: aurora-postgresql
    DBClusterIdentifier: !Ref AuroraCluster
    PubliclyAccessible: false

ReaderInstance2:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceClass: db.r6g.large
    Engine: aurora-postgresql
    DBClusterIdentifier: !Ref AuroraCluster
    PubliclyAccessible: false
```

**Connect to Aurora**:

```python
import psycopg2

# Writer endpoint - for writes
writer_conn = psycopg2.connect(
    host='myapp-cluster.cluster-123.us-east-1.rds.amazonaws.com',
    database='myapp',
    user='admin',
    password='password'
)

# Reader endpoint - for reads (load-balanced across replicas)
reader_conn = psycopg2.connect(
    host='myapp-cluster.cluster-ro-123.us-east-1.rds.amazonaws.com',
    database='myapp',
    user='admin',
    password='password'
)

# Write to writer
with writer_conn.cursor() as cursor:
    cursor.execute("INSERT INTO orders (user_id, total) VALUES (%s, %s)",
                   (123, 99.99))
    writer_conn.commit()

# Read from reader (less load on writer)
with reader_conn.cursor() as cursor:
    cursor.execute("SELECT * FROM orders WHERE user_id = %s", (123,))
    orders = cursor.fetchall()
```

### Read Replicas for Scaling

Offload read traffic from primary database.

```yaml
# Primary database
PrimaryDB:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceIdentifier: primary-db
    Engine: postgres
    MultiAZ: true
    BackupRetentionPeriod: 7  # Required for read replicas

# Read replica in same region
ReadReplica1:
  Type: AWS::RDS::DBInstance
  Properties:
    SourceDBInstanceIdentifier: !Ref PrimaryDB
    DBInstanceClass: db.t3.medium
    PubliclyAccessible: false

# Read replica in different region (cross-region)
CrossRegionReplica:
  Type: AWS::RDS::DBInstance
  Properties:
    SourceDBInstanceIdentifier: !GetAtt PrimaryDB.Arn
    Region: us-west-2
    DBInstanceClass: db.t3.medium
```

**Use read replicas in application**:

```python
import psycopg2
import random

# Connection pool
WRITER_HOST = 'primary-db.123.us-east-1.rds.amazonaws.com'
READER_HOSTS = [
    'replica1.123.us-east-1.rds.amazonaws.com',
    'replica2.123.us-east-1.rds.amazonaws.com',
    'replica3.123.us-east-1.rds.amazonaws.com'
]

def get_write_connection():
    """Always use primary for writes"""
    return psycopg2.connect(host=WRITER_HOST, database='myapp',
                            user='admin', password='password')

def get_read_connection():
    """Load balance reads across replicas"""
    reader_host = random.choice(READER_HOSTS)
    return psycopg2.connect(host=reader_host, database='myapp',
                           user='admin', password='password')

# Write operation
def create_order(user_id, total):
    conn = get_write_connection()
    with conn.cursor() as cursor:
        cursor.execute("INSERT INTO orders (user_id, total) VALUES (%s, %s)",
                      (user_id, total))
        conn.commit()
    conn.close()

# Read operation
def get_user_orders(user_id):
    conn = get_read_connection()  # Uses replica
    with conn.cursor() as cursor:
        cursor.execute("SELECT * FROM orders WHERE user_id = %s", (user_id,))
        orders = cursor.fetchall()
    conn.close()
    return orders
```

**Replication lag considerations**:

```python
def get_orders_after_write(user_id, order_id):
    """
    After a write, you may want to read from primary to avoid replication lag
    """
    # Just wrote order_id to primary
    create_order(user_id, 99.99)

    # Read from primary to ensure we see the new order
    # (replicas may be 1-5 seconds behind)
    conn = get_write_connection()  # Read from primary
    with conn.cursor() as cursor:
        cursor.execute("SELECT * FROM orders WHERE order_id = %s", (order_id,))
        order = cursor.fetchone()
    conn.close()
    return order
```

### DynamoDB: Serverless Database with Built-in HA

DynamoDB is fully managed and automatically highly available.

```yaml
UsersTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: users
    BillingMode: PAY_PER_REQUEST  # Auto-scales
    AttributeDefinitions:
      - AttributeName: userId
        AttributeType: S
      - AttributeName: email
        AttributeType: S
    KeySchema:
      - AttributeName: userId
        KeyType: HASH
    GlobalSecondaryIndexes:
      - IndexName: EmailIndex
        KeySchema:
          - AttributeName: email
            KeyType: HASH
        Projection:
          ProjectionType: ALL
    # Automatic backups
    PointInTimeRecoverySpecification:
      PointInTimeRecoveryEnabled: true
    # Multi-region (Global Tables)
    StreamSpecification:
      StreamViewType: NEW_AND_OLD_IMAGES
    SSESpecification:
      SSEEnabled: true
```

**DynamoDB automatically**:
- Replicates across 3 availability zones
- Auto-scales to handle any traffic
- Provides 99.99% availability SLA
- Takes continuous backups
- Handles hardware failures transparently

**No configuration needed for HA** - it's built-in!

---

## Summary: High Availability Patterns

### The Cloud HA Toolkit

**Geographic Distribution**:
- **Multi-AZ**: Deploy across availability zones (same region)
- **Multi-Region**: Deploy across geographic regions (global)
- **Global Load Balancing**: Route 53 latency/geolocation routing

**Load Distribution**:
- **ALB**: HTTP/HTTPS applications, path routing, host routing
- **NLB**: TCP/UDP, ultra-high performance, static IPs
- **Target Groups**: Health checks, connection draining

**Database HA**:
- **RDS Multi-AZ**: 1-2 minute failover, zero data loss
- **Aurora**: <30 second failover, 15 read replicas, auto-scaling storage
- **DynamoDB**: Built-in multi-AZ, auto-scaling, global tables for multi-region

**Deployment Patterns**:
- **Blue/Green**: Instant cutover with instant rollback
- **Canary**: Gradual rollout with monitoring
- **Rolling**: Update instances one by one

### Cost vs Availability Trade-offs

```
┌─────────────────┬─────────────┬──────────────┬───────────────┐
│ Pattern         │ Availability│ Cost/Month   │ Complexity    │
├─────────────────┼─────────────┼──────────────┼───────────────┤
│ Single AZ       │ 99.5%       │ $500         │ Low           │
│ Multi-AZ        │ 99.95%      │ $800         │ Low           │
│ Multi-Region    │ 99.99%      │ $2,500       │ Medium        │
│ Active-Active   │ 99.999%     │ $5,000+      │ High          │
└─────────────────┴─────────────┴──────────────┴───────────────┘
```

**Choose based on business requirements**:
- **E-commerce during holiday**: Active-Active (can't afford downtime)
- **SaaS application**: Multi-Region Active-Passive (global users, cost-conscious)
- **Internal tools**: Multi-AZ (regional users, moderate budget)
- **Development environment**: Single-AZ (cost optimization)

### Key Takeaways

1. **Embrace Failure**: Design assuming everything fails
2. **Redundancy at Every Layer**: No single points of failure
3. **Geographic Distribution**: Multi-AZ minimum, Multi-Region for critical apps
4. **Automated Recovery**: Health checks + auto-scaling = self-healing
5. **Test Failover**: Regularly test DR procedures (Chaos Engineering)
6. **Monitor Everything**: You can't fix what you don't measure
7. **Balance Cost vs Availability**: Not everything needs five nines

### Checklist: HA Architecture Review

- [ ] Application servers deployed across ≥3 availability zones
- [ ] Load balancer distributes traffic with health checks
- [ ] Database configured with Multi-AZ or Aurora
- [ ] Auto Scaling configured (scale up AND down)
- [ ] Health check endpoints implemented
- [ ] Deployment strategy defined (blue/green or canary)
- [ ] Multi-region failover tested (for critical apps)
- [ ] Monitoring and alerting configured
- [ ] Disaster recovery plan documented and tested
- [ ] Cost optimization reviewed (right-sized instances, Reserved Instances for baseline)

## Next Steps

You now understand how to design highly available systems in the cloud. Chapter 5 will cover **Monitoring and Observability** - because you can't operate what you can't see. You'll learn how to implement comprehensive monitoring, alerting, and troubleshooting for cloud-native applications.
