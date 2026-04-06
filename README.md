# AWS Config Cost Estimator & Optimizer — Kiro Power

Estimate AWS Config recorder costs **before enabling it** and optimize **existing** Config deployments by analyzing CloudTrail organization trail data, service dependencies, duplicate rules, and recording frequency.

## How It Works

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          Kiro IDE / Power                                │
│                                                                          │
│  Two modes:                                                              │
│                                                                          │
│  ┌─────────────────────────────┐  ┌────────────────────────────────────┐ │
│  │  COST ESTIMATION            │  │  COST OPTIMIZATION                 │ │
│  │  (Config not yet enabled)   │  │  (Config already running)          │ │
│  │                             │  │                                    │ │
│  │  1. Verify org trail        │  │  1. Analyze current Config spend   │ │
│  │  2. Query CloudTrail        │  │  2. List Config aggregators →      │ │
│  │     (7 days default)        │  │     user picks one (WAIT)          │ │
│  │  3. Map eventSource →       │  │  3. Top 10 resource types from     │ │
│  │     Config resource types   │  │     aggregator (confirm with user) │ │
│  │  4. Continuous estimate:    │  │  4. CloudTrail deep-dive ALL top   │ │
│  │     total events × $0.003   │  │     10 → specific AWS:: types      │ │
│  │  5. Periodic estimate:      │  │  5. Per-resource-ID change freq    │ │
│  │     unique resources/day    │  │  6. 4× rule per resource ID        │ │
│  │     × 30 × $0.012           │  │  7. Dependencies + exclusions      │ │
│  └─────────────────────────────┘  │  8. Control Tower workaround       │ │
│                                    └────────────────────────────────────┘ │
│                                                                          │
│  Runs from: Management Account, Config Delegated Admin,                  │
│             or CloudTrail Delegated Admin                                 │
└──────────────────────────────────────────────────────────────────────────┘
```

## Cost Estimation — Example Output

### Continuous Recording
**Analysis period**: 7 days | **Extrapolation**: ×4.29 (30 ÷ 7 — scales partial data to a full month)

| Account ID | Region | Event Source | Events (7d) | Est. Monthly CIs | Est. Monthly Cost |
|---|---|---|---|---|---|
| 123456789012 | us-east-1 | ec2.amazonaws.com | 1,200 | 5,143 | $15.43 |
| 123456789012 | us-east-1 | s3.amazonaws.com | 350 | 1,500 | $4.50 |
| **TOTAL** | | | **1,550** | **6,643** | **$19.93** |

### Periodic Recording
**Sample day**: 2026-03-31 (user-selected)

> ⚠️ Weekly and monthly projections are based on a single day's data. Re-run on different days for higher accuracy.

| Account ID | Region | Event Source | Unique Resources (1d) | Est. Weekly CIs | Est. Monthly CIs | Est. Monthly Cost |
|---|---|---|---|---|---|---|
| 123456789012 | us-east-1 | ec2.amazonaws.com | 45 | 315 | 1,350 | $16.20 |
| 123456789012 | us-east-1 | s3.amazonaws.com | 20 | 140 | 600 | $7.20 |
| **TOTAL** | | | **65** | **455** | **1,950** | **$23.40** |

## Cost Optimization — Example Output

### The 4× Rule: Continuous vs Periodic per Resource Type
Periodic is cheaper when a resource changes **more than 4× per day** ($0.012 / $0.003 = 4).

| Resource Type | Avg Events/Day | Avg Unique Resources/Day | Change Ratio | Recommendation | Est. Monthly Savings |
|---|---|---|---|---|---|
| `AWS::EC2::Subnet` | 217 | 2 | 108.5× | ✅ Switch to periodic | $18.78 |
| `AWS::EC2::NetworkInterface` | 162 | 5 | 32.4× | ✅ Switch to periodic | $12.78 |
| `AWS::S3::Bucket` | 80 | 30 | 2.7× | ❌ Keep continuous | -$3.60 |

### Per-Resource-ID Detail (periodic candidates)
| Account ID | Region | Resource Type | Resource ID | Changes/Day | Periodic Saves? |
|---|---|---|---|---|---|
| 123456789012 | us-east-1 | `AWS::EC2::Subnet` | subnet-06e0134b | 108 | ✅ Yes (108× > 4×) |
| 123456789012 | us-east-1 | `AWS::EC2::Subnet` | subnet-007336a4 | 108 | ✅ Yes (108× > 4×) |
| 123456789012 | us-east-1 | `AWS::EC2::NetworkInterface` | eni-0abc123 | 50 | ✅ Yes (50× > 4×) |

### Top CI Contributors by Account and Region
| Account ID | Region | Resource Type | Monthly CIs | Monthly Cost | % of Total |
|---|---|---|---|---|---|
| 123456789012 | us-east-1 | `AWS::EC2::SecurityGroup` | 15,000 | $45.00 | 25% |
| 123456789012 | us-east-1 | `AWS::EC2::Instance` | 10,000 | $30.00 | 17% |
| 234567890123 | eu-west-1 | `AWS::EC2::NetworkInterface` | 8,000 | $24.00 | 13% |

### Indirect Relationships — Hidden CI Multiplier
Changes to some EC2/VPC resources generate **extra CIs** for related resources:

| Change to | Generates extra CIs for |
|---|---|
| `AWS::EC2::Instance` | `AWS::EC2::SecurityGroup`, `AWS::EC2::Subnet`, `AWS::EC2::VPC` |
| `AWS::EC2::SecurityGroup` | `AWS::EC2::VPC` |
| `AWS::EC2::RouteTable` | `AWS::EC2::Instance`, `AWS::EC2::NetworkInterface`, `AWS::EC2::Subnet`, `AWS::EC2::VPNGateway`, `AWS::EC2::VPC` |

Customers can disable indirect relationships **per account** via [AWS Support case](https://docs.aws.amazon.com/config/latest/developerguide/faq.html#config-recording).

### Dependency Impact Matrix
| Recommendation | Security Hub | Control Tower | Firewall Mgr | Backup Audit Mgr | SSM Compliance |
|---|---|---|---|---|---|
| Exclude ResourceCompliance | ⚠️ Stale | ✅ OK | ✅ OK | ✅ OK | ⚠️ No history |
| Switch EC2 to periodic | ✅ OK | ✅ OK | ❌ Breaks | ✅ OK | ⚠️ Delayed |
| Exclude Backup types | ✅ OK | ✅ OK | ✅ OK | ❌ Breaks | ✅ OK |

## Service Dependencies Checked

| Service | Why It Matters |
|---|---|
| AWS Security Hub CSPM | Config rules power security checks |
| AWS Control Tower | Blocks direct Config recorder changes — requires [workaround solution](https://github.com/aws-samples/aws-control-tower-config-customization) |
| AWS Firewall Manager | Requires **continuous** recording — periodic breaks it |
| AWS Backup Audit Manager | Requires Config resource tracking for compliance frameworks |
| AWS Systems Manager Compliance | Uses Config for patch/association compliance history |
| AWS Audit Manager | Captures Config evaluations as audit evidence |
| AWS Trusted Advisor | Some checks powered by Config managed rules |
| Indirect Relationships | EC2/VPC resources generate extra CIs — can be disabled per account via [Support case](https://docs.aws.amazon.com/config/latest/developerguide/faq.html#config-recording) |

## Installation

In Kiro IDE → Powers panel → **"Add power from GitHub"** → enter:
```
https://github.com/rodbren/power-aws-config-cost-estimator
```

## Prerequisites

- **CloudTrail Organization Trail** with management events enabled
- AWS credentials with `cloudtrail:LookupEvents`, `cloudtrail:DescribeTrails`, `config:DescribeConfigRules` permissions
- For optimization: access to management account or delegated admin account

## MCP Servers Included

| Server | Purpose |
|---|---|
| `aws-documentation` | Search AWS docs for Config pricing, supported resources, best practices |
| `aws-core` | Execute AWS CLI commands to query CloudTrail and Config |
| `aws-knowledge` | Retrieve AWS knowledge base content for guidance |

## Usage

### Cost Estimation (Config not yet enabled)
- "estimate my AWS Config costs"
- "how much would it cost to enable Config recorder"
- "forecast Config costs for 30 days"

### Cost Optimization (Config already running)
- "optimize my AWS Config costs"
- "find duplicate Config rules"
- "which resource types should I switch to periodic"
- "what can I safely exclude from Config recording"
- "analyze Config cost drivers"

### Adjustable Parameters

| Parameter | Default | Options |
|---|---|---|
| Estimation window | 7 days | 1–90 days |
| Periodic sample day | Yesterday | Any specific day |
| Optimization analysis | 7 days of CloudTrail | 1–90 days |
| Account scope | Org-wide | Specific accounts |

## Project Structure

```
power-aws-config-cost-estimator/
├── POWER.md                              # Metadata, onboarding, steering mapping
├── mcp.json                              # MCP server configuration
├── README.md                             # This file
├── CHANGELOG.md                          # Version history
└── steering/
    ├── estimate-workflow.md              # Cost estimation workflow + 130+ eventSource mappings
    └── optimization-workflow.md          # Cost optimization: dependencies, duplicates,
                                          #   change frequency, exclusions, Control Tower
```

## References

- [AWS Config Pricing](https://aws.amazon.com/config/pricing/)
- [AWS Config Supported Resource Types](https://docs.aws.amazon.com/config/latest/developerguide/resource-config-reference.html)
- [AWS Config Cost Optimization Best Practices](https://aws-samples.github.io/cloud-operations-best-practices/docs/guides/AWS%20Config/AWS%20Config%20Cost%20Optimization/)
- [Cost Optimization Recommendations for AWS Config](https://aws.amazon.com/blogs/mt/cost-optimization-recommendations-for-aws-config/)
- [Discover Duplicate AWS Config Rules](https://aws.amazon.com/blogs/security/discover-duplicate-aws-config-rules-for-streamlined-compliance/)
- [Customize AWS Config in Control Tower](https://aws.amazon.com/blogs/mt/customize-aws-config-resource-tracking-in-aws-control-tower-environment/)
- [AWS Config Service Integrations](https://docs.aws.amazon.com/config/latest/developerguide/service-integrations.html)
- [AWS Backup Audit Manager](https://docs.aws.amazon.com/aws-backup/latest/devguide/aws-backup-audit-manager.html)
- [AWS Systems Manager Compliance](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-compliance.html)
- [Kiro Powers Documentation](https://kiro.dev/docs/powers/create/)

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and updates.

**To update**: Remove and re-add the power from the Kiro IDE Powers panel.

## License

MIT
