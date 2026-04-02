# AWS Config Cost Optimization Workflow

## Overview
This workflow analyzes an **existing** AWS Config deployment to identify cost optimization opportunities. It correlates Config recorder data, rule evaluations, CloudTrail events, and service dependencies to provide actionable recommendations.

**This workflow can run from:**
- **Management account** — full org-wide visibility, required for Control Tower Config recorder changes, and access to organization trail
- **Delegated administrator account for AWS Config** — can view aggregated compliance data, rules, and resource configurations across the organization
- **Delegated administrator account for CloudTrail** — can query the organization trail for per-resource-type change frequency analysis

**Ask the customer which account they're running from** — this determines what actions and data are available.

| Capability | Management Account | Config Delegated Admin | CloudTrail Delegated Admin |
|---|---|---|---|
| View aggregated Config data | ✅ | ✅ | ❌ |
| Manage Config rules/conformance packs | ✅ | ✅ | ❌ |
| Modify Config recorder (via CT solution) | ✅ | ❌ | ❌ |
| Query organization trail | ✅ | ❌ | ✅ |
| View Cost Explorer org-wide | ✅ | ⚠️ Needs billing access | ❌ |
| Full optimization workflow | ✅ | ⚠️ Partial (no recorder changes, no trail) | ⚠️ Partial (trail only) |

**For the complete workflow** (analysis + recommendations + implementation), the management account is ideal. If split across delegated admins, coordinate between Config delegated admin (rules/compliance) and CloudTrail delegated admin (change frequency analysis).

## Step 1: Analyze Current Config Costs

Use Cost Explorer to understand current Config spend:
- Filter by Service = "AWS Config"
- Group by Usage Type to see CI recording vs rule evaluations vs conformance pack evaluations
- Group by Linked Account to identify top-spending accounts
- Compare month-over-month trends

**From delegated admin**: May need billing access or CUR data. Alternatively, use CloudWatch `ConfigurationItemsRecorded` metrics per account.

## Step 2: Identify Top CI Contributors

### Via CloudWatch Metrics
Query the `ConfigurationItemsRecorded` metric in CloudWatch, grouped by `ResourceType`, to find which resource types generate the most CIs.

### Via Config Aggregator (from delegated admin)
Use the Config aggregator to query resource counts and compliance across all accounts.

### Via CloudTrail (per resource ID)
For each top-contributing resource type, query CloudTrail for non-read-only events to see **which specific resource IDs** generate the most changes:
- Extract `resources[].ARN` or resource identifiers from `requestParameters`
- Count events per unique resource ID
- Rank by frequency

## Step 3: Per-Resource-Type Change Frequency Analysis (CloudTrail)

**This is required to generate continuous-vs-periodic recommendations.** You must query the organization trail to get actual daily change counts per resource type.

**Requires access to the organization trail** — run from management account or CloudTrail delegated admin account.

### How to analyze
For each top CI-contributing resource type identified in Step 2:

1. Query the organization trail for **7 days** of non-read-only events for that resource type's eventSource
2. Group by **day** and count:
   - **Total events per day** (= continuous CIs per day)
   - **Unique resource IDs per day** (= periodic CIs per day)
3. Calculate the **average daily change ratio**:
```
change_ratio = avg_total_events_per_day / avg_unique_resources_per_day
```
4. Apply the **4× rule**:
   - If `change_ratio > 4` → **recommend periodic** (saves money because each resource changes >4×/day, and periodic only charges once per resource per day)
   - If `change_ratio ≤ 4` → **recommend staying continuous** (fewer total CIs than periodic's flat per-resource-per-day charge)
   - If `change_ratio` is close to 4 (between 3–5) → **flag as borderline**, recommend monitoring before switching

5. Calculate the actual savings:
```
current_continuous_cost = avg_total_events_per_day × 30 × $0.003
proposed_periodic_cost  = avg_unique_resources_per_day × 30 × $0.012
monthly_savings         = current_continuous_cost - proposed_periodic_cost
```

### Example analysis output per resource type

**Always use full `AWS::Service::ResourceType` names**, not generic service names.

| Resource Type | Avg Events/Day | Avg Unique Resources/Day | Change Ratio | Recommendation | Est. Monthly Savings |
|---|---|---|---|---|---|
| `AWS::EC2::SecurityGroup` | 500 | 45 | 11.1× | ✅ Switch to periodic | $33.30 |
| `AWS::S3::Bucket` | 80 | 30 | 2.7× | ❌ Keep continuous | -$3.60 (periodic costs more) |
| `AWS::Lambda::Function` | 200 | 50 | 4.0× | ⚠️ Borderline | $0.00 (break-even) |
| `AWS::IAM::Role` | 150 | 120 | 1.25× | ❌ Keep continuous | -$32.40 (periodic costs more) |

### Important: Query the right days
- Query **at least 7 days** for a representative sample
- Include both weekdays and weekends
- Avoid maintenance windows or deployment days that skew results
- Let the customer choose the analysis period if they want more/fewer days

## Step 4: Evaluate Service Dependencies

**CRITICAL**: Before excluding any resource type or switching to periodic recording, check ALL service dependencies. Ask the customer which of these services they use.

| Dependent Service | Config Requirement | Impact of Excluding Resources or Switching to Periodic |
|---|---|---|
| **AWS Security Hub CSPM** | Requires Config rules for security checks. Many are change-triggered. | Excluding resource types breaks Security Hub controls that evaluate those types |
| **AWS Control Tower** | Enables Config on all enrolled accounts. **Blocks direct changes to Config recorder.** | Cannot modify Config recorder directly — causes Landing Zone drift. Must use the [Config Customization solution](https://github.com/aws-samples/aws-control-tower-config-customization) deployed in the management account |
| **AWS Firewall Manager** | Requires **continuous recording** | Switching to periodic **will break** Firewall Manager |
| **AWS Backup Audit Manager** | Requires Config resource tracking to evaluate backup compliance frameworks and generate audit reports | Excluding Backup resource types (`AWS::Backup::*`) breaks compliance frameworks and audit reports |
| **AWS Systems Manager Compliance** | Uses Config for compliance history and change tracking of patch compliance and State Manager associations | Excluding EC2/SSM resource types breaks compliance history, change tracking, and fleet-wide reporting |
| **AWS Audit Manager** | Captures Config rule evaluations as audit evidence | Removing rules or excluding resource types impacts audit evidence collection |
| **AWS Trusted Advisor** | Some checks powered by Config managed rules (Business/Enterprise Support) | Disabling those managed rules removes corresponding Trusted Advisor recommendations |
| **AWS Organizations** | Uses Config for multi-account aggregation | Aggregator views may show gaps if resource types are excluded |

**Always present these dependencies** when recommending exclusions or recording frequency changes.

## Step 5: Control Tower — How to Properly Customize Config Recorder

**AWS Control Tower blocks direct changes to the Config recorder.** Modifying it directly causes Landing Zone drift and can block account management operations.

**This step requires management account access.** If running from delegated admin, the customer must coordinate with the management account owner.

### The Workaround Solution
Use the [aws-control-tower-config-customization](https://github.com/aws-samples/aws-control-tower-config-customization) solution:

1. Deploy the CloudFormation template in the **management account's home region**
2. It uses `AWSControlTowerExecution` role to update Config recorder in member accounts
3. Hooks into Control Tower lifecycle events (`UpdateLandingZone`, `CreateManagedAccount`, `UpdateManagedAccount`) so changes persist
4. Supports:
   - **Exclusion mode**: Exclude specific resource types from recording
   - **Inclusion mode**: Record only specific resource types
   - **Recording frequency**: Switch between continuous and daily per resource type
   - **Account targeting**: Exclude/include specific accounts (e.g., keep prod on continuous, switch dev to periodic)

### Key parameters:
- `AccountSelectionMode`: EXCLUSION or INCLUSION for account targeting
- `ExcludedAccounts`: Always exclude Log Archive and Audit accounts
- `ConfigRecorderStrategy`: EXCLUSION or INCLUSION for resource types
- `ConfigRecorderExcludedResourceTypes`: Comma-separated resource types to exclude
- `ConfigRecorderDefaultRecordingFrequency`: Continuous or Daily
- `ConfigRecorderDailyResourceTypes`: Resource types to record daily instead of continuous
- `ConfigRecorderDailyGlobalResourceTypes`: Global resource types to record daily in home region

### Important warnings:
- **Always exclude Log Archive and Audit accounts** from changes
- Test in non-production accounts first
- Changes persist across Landing Zone updates and new account enrollments

## Step 6: ResourceCompliance Exclusion — Stale Compliance Risk

`AWS::Config::ResourceCompliance` is often the **#1 CI generator**. However, excluding it has important side effects:

### What happens when you exclude ResourceCompliance
- Config rules (detective controls) will show **stale compliance results** — the last known state before exclusion, never updating again
- The Config console timeline view for compliance history stops working
- Security Hub findings based on those rules may become outdated
- This affects ALL detective rules, not just rules for the excluded resource type

### Mitigation
- Use **CloudTrail** as a free alternative for compliance history. Query `PutEvaluations` events:
```sql
SELECT eventTime, awsRegion, recipientAccountId,
  element_at(additionalEventData, 'configRuleName') as configRuleName,
  json_extract_scalar(json_array_get(element_at(requestParameters,'evaluations'),0),'$.complianceType') as Compliance,
  json_extract_scalar(json_array_get(element_at(requestParameters,'evaluations'),0),'$.complianceResourceType') as ResourceType,
  json_extract_scalar(json_array_get(element_at(requestParameters,'evaluations'),0),'$.complianceResourceId') as ResourceName
FROM $EDS_ID
WHERE eventName='PutEvaluations'
  AND eventTime > '2026-01-01 00:00:00'
  AND eventTime < '2026-02-01 00:00:00'
  AND json_extract_scalar(json_array_get(element_at(requestParameters,'evaluations'),0),'$.complianceType') IN ('COMPLIANT','NON_COMPLIANT')
```
- Ensure Security Hub is still getting findings from other sources
- Document that compliance timeline is no longer real-time

## Step 7: Identify Duplicate Rules

**This analysis can be done from the delegated admin account** using `describe-config-rules`.

Duplicate Config rules are a common cost driver from multiple sources:
- AWS Config conformance packs
- AWS Security Hub CSPM standards
- AWS Control Tower controls
- Individually created rules

### How to detect duplicates
Rules are duplicates if they have identical `SourceIdentifier`, `InputParameters`, and `Scope`.

### Naming patterns by source
- Conformance packs: `rule-name-conformance-pack-a1b2c3d4e`
- Security Hub: `securityhub-rule-name-a1b2c3`
- Control Tower: `AWSControlTower_AWS-GR_RULE_NAME`

### Resolution options
1. Conformance packs overlapping with Security Hub → remove from conformance pack
2. Security Hub standards overlap → disable individual controls
3. Control Tower controls overlap → disable from Control Tower console (management account)
4. Use custom conformance packs instead of sample templates

**Reference**: [Discover duplicate AWS Config rules](https://aws.amazon.com/blogs/security/discover-duplicate-aws-config-rules-for-streamlined-compliance/)

## Step 8: Resource Exclusion Candidates

### Safe to exclude (generally)
- `AWS::Config::ResourceCompliance` (with stale compliance mitigation — see Step 6)
- Resource types not relevant to compliance framework
- Ephemeral/temporary resources

### Risky to exclude
- Resource types required by Security Hub controls
- Resources monitored by Firewall Manager
- `AWS::Backup::*` types if using Backup Audit Manager
- EC2/SSM types if using Systems Manager Compliance
- IAM types if you have IAM compliance rules

### Before excluding, always:
1. Check which Config rules evaluate that resource type
2. Check Security Hub control dependencies
3. Check Backup Audit Manager framework references
4. Check SSM Compliance tracking
5. Verify with security/compliance team
6. Document the decision and rationale

## Step 9: Additional Optimizations

### Global resource recording
Record global resources (IAM) in **one region only**. Disable `IncludeGlobalResourceTypes` in all other regions.

### API call consolidation
Batch tag operations (10 tags in 1 call vs 10 calls = 10× fewer CIs).

### Custom rules
Prefer CloudFormation Guard over Lambda. Narrow scope to specific resource types.

### S3 delivery channel
Set lifecycle policies on Config S3 bucket. Review if you need both snapshots and history files.

### Rule evaluation hygiene
Avoid frequent `DeleteEvaluationResults` / `StartConfigRulesEvaluation` — each generates new CIs. When deleting rules with many resources: stop compliance recording → delete rules → restart.

### Indirect Relationships — Hidden CI Multiplier

Indirect relationships generate **extra CIs** when related resources change. This is a significant hidden cost driver, especially for EC2/VPC resources.

#### How it works
- **Direct** (A→B): Change to security group → CI for security group + CI for EC2 instance
- **Indirect** (B→A): Change to EC2 instance → CI for EC2 instance + CI for security group

#### Full indirect relationship CI generation table
Changes to these resource types generate **additional CIs** for the related types:

| Change to this resource type | Generates extra CIs for |
|---|---|
| `AWS::EC2::RouteTable` | `AWS::EC2::Instance`, `AWS::EC2::NetworkInterface`, `AWS::EC2::Subnet`, `AWS::EC2::VPNGateway`, `AWS::EC2::VPC` |
| `AWS::EC2::EIP` | `AWS::EC2::Instance`, `AWS::EC2::NetworkInterface` |
| `AWS::EC2::Instance` | `AWS::EC2::SecurityGroup`, `AWS::EC2::Subnet`, `AWS::EC2::VPC` |
| `AWS::EC2::NetworkInterface` | `AWS::EC2::SecurityGroup`, `AWS::EC2::Subnet`, `AWS::EC2::VPC` |
| `AWS::EC2::NetworkACL` | `AWS::EC2::Subnet`, `AWS::EC2::VPC` |
| `AWS::EC2::VPNConnection` | `AWS::EC2::VPNGateway`, `AWS::EC2::CustomerGateway` |
| `AWS::EC2::InternetGateway` | `AWS::EC2::VPC` |
| `AWS::EC2::SecurityGroup` | `AWS::EC2::VPC` |
| `AWS::EC2::Subnet` | `AWS::EC2::VPC` |
| `AWS::EC2::VPNGateway` | `AWS::EC2::VPC` |

#### Services that depend on indirect relationships
- **`ec2-security-group-attached-to-eni`** Config managed rule — needs indirect relationships to check if non-default security groups are attached to ENIs
- **AWS Firewall Manager** Usage Audit Security Group policy — uses indirect relationships to determine when a security group was last used
- **Default VPC resources** — without indirect relationships, default resources (security groups, NACLs, route tables) created with a VPC take up to 12 hours to be recorded

#### How to disable indirect relationships
Customers can disable indirect relationships **per account** via an AWS Support case:
1. Open a **Technical** support case
2. Service: **AWS Config**, Category: **Other**
3. Subject: **Disable Indirect Relationship**
4. Description: confirm you've read the FAQ, list regions, and account IDs
5. For multiple accounts, attach a CSV with account IDs and regions
6. Can be submitted from individual account or management account for bulk

**Before disabling**, verify:
- No Config rules depend on indirect relationships (e.g., `ec2-security-group-attached-to-eni`)
- Firewall Manager Usage Audit policies are not in use
- Acceptable to wait up to 12 hours for default resource recording

**Reference**: [Indirect Relationships in AWS Config FAQ](https://docs.aws.amazon.com/config/latest/developerguide/faq.html#config-recording)

## Step 10: Present Optimization Report

### Current State Summary
| Metric | Value |
|---|---|
| Monthly Config spend | $X,XXX |
| Total CIs recorded/month | X,XXX |
| Total rule evaluations/month | X,XXX |
| Top CI resource type | AWS::XXX::YYY |
| Config rules count | XX |
| Conformance packs count | XX |
| Duplicate rules detected | XX |
| Control Tower managed | Yes/No |
| Running from | Management / Config Delegated Admin / CT Delegated Admin |

### Recording Frequency Recommendations (from CloudTrail analysis)

**Always use full AWS Config resource type names** (e.g., `AWS::EC2::Instance`, `AWS::EC2::SecurityGroup`, `AWS::S3::Bucket`) — never generic service names like "EC2" or "S3".

#### Top CI Contributors by Account and Region
| Account ID | Region | Resource Type | Monthly CIs | Monthly Cost | % of Total |
|---|---|---|---|---|---|
| 123456789012 | us-east-1 | `AWS::EC2::SecurityGroup` | 15,000 | $45.00 | 25% |
| 123456789012 | us-east-1 | `AWS::EC2::Instance` | 10,000 | $30.00 | 17% |
| 234567890123 | eu-west-1 | `AWS::EC2::NetworkInterface` | 8,000 | $24.00 | 13% |
| 123456789012 | us-east-1 | `AWS::Config::ResourceCompliance` | 7,000 | $21.00 | 12% |
| 345678901234 | us-west-2 | `AWS::EC2::VPC` | 5,000 | $15.00 | 8% |

#### Continuous vs Periodic Analysis per Resource Type
| Resource Type | Avg Events/Day | Avg Unique Resources/Day | Change Ratio | Recommendation | Est. Monthly Savings | Dependencies to Check |
|---|---|---|---|---|---|---|
| `AWS::EC2::SecurityGroup` | 500 | 45 | 11.1× | ✅ Periodic | $33.30 | Firewall Manager, indirect relationships |
| `AWS::S3::Bucket` | 80 | 30 | 2.7× | ❌ Keep continuous | -$3.60 | None |
| `AWS::Lambda::Function` | 200 | 50 | 4.0× | ⚠️ Borderline | $0.00 | None |
| `AWS::IAM::Role` | 150 | 120 | 1.25× | ❌ Keep continuous | -$32.40 | Global resource — record in 1 region only |

### Other Recommendations (ranked by savings)
| # | Recommendation | Est. Savings/mo | Risk | Requires Mgmt Account |
|---|---|---|---|---|
| 1 | Exclude `ResourceCompliance` | $XXX | Low (stale compliance) | Yes |
| 2 | Remove X duplicate rules | $XXX | Low | No |
| 3 | Exclude type Z | $XXX | Medium | Yes |
| 4 | Global resources in 1 region | $XXX | Low | Yes |

### Dependency Impact Matrix
| Recommendation | Security Hub | Control Tower | Firewall Mgr | Backup Audit Mgr | SSM Compliance | Audit Mgr | Trusted Advisor |
|---|---|---|---|---|---|---|---|
| Exclude ResourceCompliance | ⚠️ Stale | ✅ OK | ✅ OK | ✅ OK | ⚠️ No history | ⚠️ Less evidence | ✅ OK |
| Switch EC2 to periodic | ✅ OK | ✅ OK | ❌ Breaks | ✅ OK | ⚠️ Delayed | ✅ OK | ✅ OK |
| Exclude Backup types | ✅ OK | ✅ OK | ✅ OK | ❌ Breaks | ✅ OK | ⚠️ Less evidence | ✅ OK |
| Exclude EC2/SSM types | ⚠️ Breaks | ✅ OK | ❌ Breaks | ✅ OK | ❌ Breaks | ⚠️ Less evidence | ⚠️ Fewer |

### For Control Tower environments
If recommendations require Config recorder changes, provide the [Control Tower Config Customization solution](https://github.com/aws-samples/aws-control-tower-config-customization) deployment steps with specific parameters. **Must be deployed from management account.**

## References

- [AWS Config Cost Optimization Best Practices](https://aws-samples.github.io/cloud-operations-best-practices/docs/guides/AWS%20Config/AWS%20Config%20Cost%20Optimization/)
- [Cost Optimization Recommendations for AWS Config](https://aws.amazon.com/blogs/mt/cost-optimization-recommendations-for-aws-config/)
- [Discover Duplicate AWS Config Rules](https://aws.amazon.com/blogs/security/discover-duplicate-aws-config-rules-for-streamlined-compliance/)
- [Customize AWS Config in Control Tower](https://aws.amazon.com/blogs/mt/customize-aws-config-resource-tracking-in-aws-control-tower-environment/)
- [Control Tower Config Customization (GitHub)](https://github.com/aws-samples/aws-control-tower-config-customization)
- [AWS Config Service Integrations](https://docs.aws.amazon.com/config/latest/developerguide/service-integrations.html)
- [AWS Config Delegated Administrator](https://docs.aws.amazon.com/config/latest/developerguide/aggregated-register-delegated-administrator.html)
- [AWS Backup Audit Manager](https://docs.aws.amazon.com/aws-backup/latest/devguide/aws-backup-audit-manager.html)
- [AWS Systems Manager Compliance](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-compliance.html)
- [Managing Recording Frequency](https://docs.aws.amazon.com/config/latest/developerguide/managing-recorder_console-change-recording-frequency.html)
- [AWS Config Pricing](https://aws.amazon.com/config/pricing/)
