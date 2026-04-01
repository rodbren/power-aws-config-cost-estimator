# AWS Config Cost Estimator — Kiro Power

Estimate AWS Config recorder costs **before enabling it** by analyzing CloudTrail organization trail write events against AWS Config's supported resource types.

## How It Works

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Kiro IDE / Power                             │
│                                                                     │
│  1. Activate power with keywords                                    │
│     ("estimate config costs", "config pricing", etc.)               │
│                                                                     │
│  2. Onboarding: Verify CloudTrail Organization Trail exists         │
│                                                                     │
│  3. Query CloudTrail (default: last 7 days)                         │
│     ┌──────────────────────────────────┐                            │
│     │  CloudTrail Organization Trail   │                            │
│     │  ┌────────────────────────────┐  │                            │
│     │  │ Filter:                    │  │                            │
│     │  │  • readOnly = false        │  │                            │
│     │  │  • managementEvent = true  │  │                            │
│     │  └────────────────────────────┘  │                            │
│     └──────────────┬───────────────────┘                            │
│                    │                                                │
│                    ▼                                                │
│  4. Map eventSource → AWS Config Supported Resource Types           │
│     ┌──────────────────────────────────────────────────┐            │
│     │ ec2.amazonaws.com        → EC2, EBS, VPC, ...    │            │
│     │ s3.amazonaws.com         → S3 Buckets, ...       │            │
│     │ lambda.amazonaws.com     → Lambda Functions       │            │
│     │ iam.amazonaws.com        → IAM Users, Roles, ... │            │
│     │ rds.amazonaws.com        → RDS Instances, ...    │            │
│     │ ... 130+ eventSource mappings                    │            │
│     └──────────────┬───────────────────────────────────┘            │
│                    │                                                │
│                    ▼                                                │
│  5. Aggregate by Account / Region / Service                         │
│                                                                     │
│  6. Extrapolate to monthly estimate (CIs × 30/days_queried)        │
│                                                                     │
│  7. Calculate costs using AWS Config pricing                        │
│     ┌──────────────────────────────────────────────┐                │
│     │ Continuous: $0.003 per CI                    │                │
│     │ Periodic:   $0.012 per CI (1/resource/day)   │                │
│     └──────────────────────────────────────────────┘                │
│                                                                     │
│  8. Output: Cost breakdown table + top cost drivers                 │
└─────────────────────────────────────────────────────────────────────┘
```

## Example Output

| Account ID | Region | Event Source | CIs (7d) | Est. Monthly CIs | Cost (Continuous) | Cost (Periodic) |
|---|---|---|---|---|---|---|
| 123456789012 | us-east-1 | ec2.amazonaws.com | 1,200 | 5,143 | $15.43 | — |
| 123456789012 | us-east-1 | s3.amazonaws.com | 350 | 1,500 | $4.50 | — |
| 123456789012 | us-east-1 | iam.amazonaws.com | 200 | 857 | $2.57 | — |
| **TOTAL** | | | **1,750** | **7,500** | **$22.50** | **$36.00** |

## Installation

In Kiro IDE:
1. Open the **Powers panel**
2. Click **"Add power from GitHub"**
3. Enter: `https://github.com/rodbren/power-aws-config-cost-estimator`

## Prerequisites

- **CloudTrail Organization Trail** with management events enabled
- AWS credentials with `cloudtrail:LookupEvents` and `cloudtrail:DescribeTrails` permissions

## MCP Servers Included

| Server | Purpose |
|---|---|
| `aws-documentation` | Search AWS docs for Config pricing and supported resource types |
| `aws-core` | Execute AWS CLI commands to query CloudTrail data |
| `aws-knowledge` | Retrieve AWS knowledge base content for guidance |

## Usage

Activate the power by mentioning any of these keywords in a conversation:
- "aws config", "config cost", "config estimator", "config pricing"
- "configuration items", "config recorder", "cloudtrail"

### Adjustable Time Range

| Command | Days | Accuracy |
|---|---|---|
| "quick estimate" | 7 (default) | Good |
| "use 1 day" | 1 | Low — snapshot only |
| "use 30 days" | 30 | High |
| "use 90 days" | 90 | Highest (max CloudTrail LookupEvents supports) |

## Caveats

- **Approximation**: Not every CloudTrail write event creates a Config CI
- **Missing types**: Some Config resource types (e.g., `AWS::Config::ResourceCompliance`) have no CloudTrail equivalent
- **CI recording only**: Config rules and conformance pack evaluations are billed separately
- **Regional pricing**: Verify current pricing at [aws.amazon.com/config/pricing](https://aws.amazon.com/config/pricing/)

## Project Structure

```
power-aws-config-cost-estimator/
├── POWER.md                          # Power metadata, onboarding, steering mapping
├── mcp.json                          # MCP server configuration
├── README.md                         # This file
└── steering/
    └── estimate-workflow.md          # Full estimation workflow + eventSource mapping
```

## References

- [AWS Config Pricing](https://aws.amazon.com/config/pricing/)
- [AWS Config Supported Resource Types](https://docs.aws.amazon.com/config/latest/developerguide/resource-config-reference.html)
- [Estimating AWS Config Costs Using CloudTrail (AWS Blog)](https://aws.amazon.com/blogs/mt/estimating-aws-config-recorder-costs-and-usage-using-aws-cloudtrail/)
- [Kiro Powers Documentation](https://kiro.dev/docs/powers/create/)

## License

MIT
