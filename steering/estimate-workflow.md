# AWS Config Cost Estimation Workflow

## Overview
This workflow uses CloudTrail organization trail data to estimate AWS Config configuration item (CI) costs. It queries non-read-only management events from CloudTrail, maps them to AWS Config supported resource types, and calculates estimated monthly costs.

## Step 1: Verify CloudTrail Organization Trail

Run `aws cloudtrail describe-trails` and confirm an organization trail exists with `IsOrganizationTrail: true` and `IsMultiRegionTrail: true`. If not, warn the user that estimates will be limited to the current account/region.

## Step 2: Query CloudTrail for Write Events

Use the `lookup_events` tool to query CloudTrail for non-read-only management events.

### Default Time Range: 7 days
- **Default**: Query the **last 7 days** of CloudTrail data, then extrapolate to a monthly estimate (multiply by ~4.3).
- **Customer can request fewer days** (e.g., 1 day, 3 days) for a quick snapshot — warn that shorter periods are less accurate.
- **Customer can request more days** (e.g., 14, 30, 60, up to 90 days max) for higher accuracy — CloudTrail `LookupEvents` supports up to 90 days of history.
- **Always state the time range used** and the extrapolation factor in the output.

**Extrapolation formula**:
```
Estimated Monthly CIs = (CIs in queried period) × (30 / days_queried)
```

**Important**: CloudTrail `lookup_events` returns max 50 events per call and supports pagination. You must paginate through ALL results to get accurate counts.

Filter criteria:
- `ReadOnly = false` (write events only — these are the ones that trigger Config CIs)
- Focus on `eventSource` values that map to AWS Config supported resource types (see mapping below)

## Step 3: Map CloudTrail eventSource to AWS Config Resource Types

The following CloudTrail `eventSource` values map to AWS Config supported resource types. Only count events from these sources:

| CloudTrail eventSource | AWS Config Service Coverage |
|---|---|
| `appstream.amazonaws.com` | AppStream |
| `appflow.amazonaws.com` | AppFlow |
| `app-integrations.amazonaws.com` | AppIntegrations |
| `apigateway.amazonaws.com` | API Gateway |
| `athena.amazonaws.com` | Athena |
| `bedrock.amazonaws.com` | Bedrock |
| `cloudfront.amazonaws.com` | CloudFront |
| `monitoring.amazonaws.com` | CloudWatch |
| `logs.amazonaws.com` | CloudWatch Logs |
| `rum.amazonaws.com` | CloudWatch RUM |
| `evidently.amazonaws.com` | CloudWatch Evidently |
| `internetmonitor.amazonaws.com` | Internet Monitor |
| `codeguru-reviewer.amazonaws.com` | CodeGuru Reviewer |
| `codeguru-profiler.amazonaws.com` | CodeGuru Profiler |
| `cognito-idp.amazonaws.com` | Cognito |
| `cognito-identity.amazonaws.com` | Cognito |
| `comprehend.amazonaws.com` | Comprehend |
| `connect.amazonaws.com` | Connect |
| `detective.amazonaws.com` | Detective |
| `dynamodb.amazonaws.com` | DynamoDB |
| `ec2.amazonaws.com` | EC2, EBS, VPC, Subnets, Security Groups, NAT Gateways, Transit Gateways, etc. |
| `imagebuilder.amazonaws.com` | EC2 Image Builder |
| `ecr.amazonaws.com` | ECR |
| `ecs.amazonaws.com` | ECS |
| `elasticfilesystem.amazonaws.com` | EFS |
| `eks.amazonaws.com` | EKS |
| `elasticmapreduce.amazonaws.com` | EMR |
| `emr-serverless.amazonaws.com` | EMR Serverless |
| `events.amazonaws.com` | EventBridge |
| `schemas.amazonaws.com` | EventBridge Schemas |
| `scheduler.amazonaws.com` | EventBridge Scheduler |
| `forecast.amazonaws.com` | Forecast |
| `frauddetector.amazonaws.com` | Fraud Detector |
| `gamelift.amazonaws.com` | GameLift |
| `guardduty.amazonaws.com` | GuardDuty |
| `inspector2.amazonaws.com` | Inspector |
| `ivs.amazonaws.com` | IVS |
| `cassandra.amazonaws.com` | Keyspaces |
| `geo.amazonaws.com` | Location Service |
| `es.amazonaws.com` | OpenSearch (legacy) |
| `opensearch.amazonaws.com` | OpenSearch |
| `aoss.amazonaws.com` | OpenSearch Serverless |
| `personalize.amazonaws.com` | Personalize |
| `pinpoint.amazonaws.com` | Pinpoint |
| `qldb.amazonaws.com` | QLDB |
| `kendra.amazonaws.com` | Kendra |
| `kinesis.amazonaws.com` | Kinesis |
| `kinesisanalytics.amazonaws.com` | Kinesis Analytics |
| `firehose.amazonaws.com` | Data Firehose |
| `kinesisvideo.amazonaws.com` | Kinesis Video |
| `lex.amazonaws.com` | Lex |
| `models.lex.amazonaws.com` | Lex V2 |
| `lightsail.amazonaws.com` | Lightsail |
| `lookoutmetrics.amazonaws.com` | Lookout for Metrics |
| `lookoutvision.amazonaws.com` | Lookout for Vision |
| `macie2.amazonaws.com` | Macie |
| `grafana.amazonaws.com` | Managed Grafana |
| `aps.amazonaws.com` | Managed Prometheus |
| `memory-db.amazonaws.com` | MemoryDB |
| `mq.amazonaws.com` | Amazon MQ |
| `kafka.amazonaws.com` | MSK |
| `kafkaconnect.amazonaws.com` | MSK Connect |
| `qbusiness.amazonaws.com` | Q Business |
| `quicksight.amazonaws.com` | QuickSight |
| `redshift.amazonaws.com` | Redshift |
| `rds.amazonaws.com` | RDS |
| `route53.amazonaws.com` | Route 53 |
| `route53resolver.amazonaws.com` | Route 53 Resolver |
| `route53-recovery-readiness.amazonaws.com` | Route 53 ARC |
| `route53-recovery-control-config.amazonaws.com` | Route 53 ARC |
| `sagemaker.amazonaws.com` | SageMaker |
| `ses.amazonaws.com` | SES |
| `sns.amazonaws.com` | SNS |
| `sqs.amazonaws.com` | SQS |
| `s3.amazonaws.com` | S3 |
| `verifiedpermissions.amazonaws.com` | Verified Permissions |
| `workspaces.amazonaws.com` | WorkSpaces |
| `amplify.amazonaws.com` | Amplify |
| `appconfig.amazonaws.com` | AppConfig |
| `apprunner.amazonaws.com` | App Runner |
| `appmesh.amazonaws.com` | App Mesh |
| `appsync.amazonaws.com` | AppSync |
| `auditmanager.amazonaws.com` | Audit Manager |
| `autoscaling.amazonaws.com` | Auto Scaling |
| `b2bi.amazonaws.com` | B2B Data Interchange |
| `backup.amazonaws.com` | Backup |
| `backup-gateway.amazonaws.com` | Backup Gateway |
| `batch.amazonaws.com` | Batch |
| `bcm-data-exports.amazonaws.com` | Billing/Cost Management |
| `budgets.amazonaws.com` | Budgets |
| `acm.amazonaws.com` | Certificate Manager |
| `cleanrooms.amazonaws.com` | Clean Rooms |
| `cleanrooms-ml.amazonaws.com` | Clean Rooms ML |
| `cloudformation.amazonaws.com` | CloudFormation |
| `cloudtrail.amazonaws.com` | CloudTrail |
| `cloud9.amazonaws.com` | Cloud9 |
| `servicediscovery.amazonaws.com` | Cloud Map |
| `codeartifact.amazonaws.com` | CodeArtifact |
| `codebuild.amazonaws.com` | CodeBuild |
| `codedeploy.amazonaws.com` | CodeDeploy |
| `codepipeline.amazonaws.com` | CodePipeline |
| `config.amazonaws.com` | Config |
| `ce.amazonaws.com` | Cost Explorer |
| `dms.amazonaws.com` | DMS |
| `datasync.amazonaws.com` | DataSync |
| `deadline.amazonaws.com` | Deadline Cloud |
| `devicefarm.amazonaws.com` | Device Farm |
| `elasticbeanstalk.amazonaws.com` | Elastic Beanstalk |
| `entityresolution.amazonaws.com` | Entity Resolution |
| `fis.amazonaws.com` | Fault Injection Service |
| `globalaccelerator.amazonaws.com` | Global Accelerator |
| `glue.amazonaws.com` | Glue |
| `databrew.amazonaws.com` | Glue DataBrew |
| `groundstation.amazonaws.com` | Ground Station |
| `healthlake.amazonaws.com` | HealthLake |
| `iam.amazonaws.com` | IAM |
| `access-analyzer.amazonaws.com` | IAM Access Analyzer |
| `rolesanywhere.amazonaws.com` | IAM Roles Anywhere |
| `iot.amazonaws.com` | IoT |
| `iot-wireless.amazonaws.com` | IoT Wireless |
| `iotdeviceadvisor.amazonaws.com` | IoT Device Advisor |
| `iotanalytics.amazonaws.com` | IoT Analytics |
| `iotevents.amazonaws.com` | IoT Events |
| `iottwinmaker.amazonaws.com` | IoT TwinMaker |
| `iotsitewise.amazonaws.com` | IoT SiteWise |
| `greengrass.amazonaws.com` | Greengrass V2 |
| `kms.amazonaws.com` | KMS |
| `lambda.amazonaws.com` | Lambda |
| `m2.amazonaws.com` | Mainframe Modernization |
| `network-firewall.amazonaws.com` | Network Firewall |
| `networkmanager.amazonaws.com` | Network Manager |
| `organizations.amazonaws.com` | Organizations |
| `omics.amazonaws.com` | HealthOmics |
| `panorama.amazonaws.com` | Panorama |
| `acm-pca.amazonaws.com` | Private CA |
| `resiliencehub.amazonaws.com` | Resilience Hub |
| `resource-explorer-2.amazonaws.com` | Resource Explorer |
| `resource-groups.amazonaws.com` | Resource Groups |
| `robomaker.amazonaws.com` | RoboMaker |
| `signer.amazonaws.com` | Signer |
| `secretsmanager.amazonaws.com` | Secrets Manager |
| `securityhub.amazonaws.com` | Security Hub |
| `servicecatalog.amazonaws.com` | Service Catalog |
| `shield.amazonaws.com` | Shield |
| `states.amazonaws.com` | Step Functions |
| `ssm.amazonaws.com` | Systems Manager |
| `transfer.amazonaws.com` | Transfer Family |
| `wafv2.amazonaws.com` | WAF V2 |
| `waf.amazonaws.com` | WAF Classic |
| `waf-regional.amazonaws.com` | WAF Regional |
| `xray.amazonaws.com` | X-Ray |
| `elasticloadbalancing.amazonaws.com` | ELB / ALB / NLB / GLB |
| `mediaconnect.amazonaws.com` | MediaConnect |
| `mediapackage.amazonaws.com` | MediaPackage |
| `mediapackagev2.amazonaws.com` | MediaPackage V2 |
| `mediatailor.amazonaws.com` | MediaTailor |

## Step 4: Aggregate and Count Events

### For Continuous Recording Estimate
Group the non-read-only events by:
1. **Account ID** (`recipientAccountId`)
2. **Region** (`awsRegion`)
3. **Event Source** (`eventSource`)

The **total event count** per group represents the estimated number of CIs (every change = 1 CI).

### For Periodic Recording Estimate
Periodic recording charges **1 CI per unique resource per day**, regardless of how many times that resource changed. To estimate this:

1. **Ask the user which day** they want to use for the daily sample. Default to **yesterday** if not specified. The user may want to pick a typical business day vs. a weekend or maintenance window day for more representative results.
2. Query **that single day** of CloudTrail data
3. Extract the **resource ID** from each event. CloudTrail events contain resource identifiers in:
   - `resources[].ARN` — the primary source for resource IDs
   - `requestParameters` — contains resource-specific identifiers (e.g., `instanceId`, `bucketName`, `functionName`)
   - `responseElements` — for create events where the resource ID is returned
4. Count **distinct resource IDs per eventSource per account per region** for that single day
5. Present the results at multiple time horizons:
   - **Daily**: unique resource count (actual data)
   - **Weekly**: daily count × 7 (forecasted)
   - **Monthly**: daily count × 30 (forecasted)

**Important**: The periodic estimate must be based on **unique resource IDs**, not total events. A single EC2 instance modified 50 times in a day = 1 CI under periodic, but 50 CIs under continuous.

**Disclaimer**: Always include this disclaimer for periodic estimates:
> ⚠️ **Forecasting disclaimer**: The weekly and monthly periodic estimates are projections based on a single day's data (YYYY-MM-DD). Actual costs will vary depending on daily resource activity patterns. For higher accuracy, the user can re-run this analysis on different days (e.g., weekday vs. weekend, peak vs. off-peak) and compare results. The day chosen significantly impacts the forecast — a maintenance day will show higher activity than a typical day.

## Step 5: Calculate Estimated Cost

Use the following AWS Config pricing tiers (US East - N. Virginia, public regions):

### Configuration Items (Continuous Recording)
- **$0.003 per configuration item** recorded

### Configuration Items (Periodic Recording)
- **$0.012 per configuration item** recorded (1 CI per resource per day instead of per-change)

### Config Rules (if applicable)
- First 100,000 evaluations: **$0.001 per evaluation**
- Next 400,000 evaluations (100,001–500,000): **$0.001 per evaluation**
- Over 500,000 evaluations: **$0.001 per evaluation**

### Conformance Pack Evaluations (if applicable)
- First 100,000 evaluations: **$0.001 per evaluation**
- Next 400,000 evaluations: **$0.001 per evaluation**
- Over 500,000 evaluations: **$0.001 per evaluation**

### Cost Formula
```
--- CONTINUOUS ---
Extrapolation Factor = 30 / days_queried  (default: 30 / 7 = ~4.29)
Estimated Monthly CIs = TotalEvents_in_period × Extrapolation Factor
Estimated Monthly Cost = Estimated Monthly CIs × $0.003

--- PERIODIC ---
(Use 1 day of data — user chooses the day, default: yesterday)
Unique Resources Per Day = COUNT(DISTINCT resource_id) per eventSource/account/region
Estimated Weekly CIs  = Unique Resources Per Day × 7
Estimated Monthly CIs = Unique Resources Per Day × 30
Estimated Weekly Cost  = Estimated Weekly CIs × $0.012
Estimated Monthly Cost = Estimated Monthly CIs × $0.012
```

**Always present both continuous and periodic estimates** so the customer can compare.
**Always show the extrapolation factor, days queried, and for periodic: the specific day used, unique resource count, and the forecasting disclaimer.**

## Step 6: Present Results

Format the output as two tables:

### Continuous Recording Estimate
**Analysis period**: X days (YYYY-MM-DD to YYYY-MM-DD) | **Extrapolation factor**: 30/X = Y.YY

| Account ID | Region | Event Source | Events (period) | Est. Monthly CIs | Est. Monthly Cost |
|---|---|---|---|---|---|
| 123456789012 | us-east-1 | ec2.amazonaws.com | 1,163 | 4,984 | $14.95 |
| ... | ... | ... | ... | ... | ... |
| **TOTAL** | | | **X** | **Y** | **$A.AA** |

### Periodic Recording Estimate
**Sample day**: YYYY-MM-DD (user-selected) | **Unique resources per day, projected to weekly and monthly**

> ⚠️ **Forecasting disclaimer**: Weekly and monthly projections are based on a single day's data. Actual costs vary by daily activity. Re-run on different days for higher accuracy.

| Account ID | Region | Event Source | Unique Resources (1 day) | Est. Weekly CIs | Est. Monthly CIs | Est. Weekly Cost | Est. Monthly Cost |
|---|---|---|---|---|---|---|---|
| 123456789012 | us-east-1 | ec2.amazonaws.com | 45 | 315 | 1,350 | $3.78 | $16.20 |
| ... | ... | ... | ... | ... | ... | ... | ... |
| **TOTAL** | | | **X** | **Y** | **Z** | **$A.AA** | **$B.BB** |

Then provide:
1. **Total estimated monthly cost** for continuous recording
2. **Total estimated monthly cost** for periodic recording
3. **Top 5 services by CI count** — these are the cost drivers
4. **Caveats and limitations** (see below)

## Caveats to Always Mention

1. **Approximation only**: Not every CloudTrail write event creates a Config CI. Some API calls modify sub-resources that don't map 1:1 to Config resource types.
2. **Missing resource types**: Some Config resource types have no CloudTrail equivalent (e.g., `AWS::Config::ResourceCompliance`, `AWS::Config::ConfigurationRecorder`).
3. **CloudTrail LookupEvents limitation**: Only returns the last 90 days of management events and has API rate limits. For larger analysis, recommend CloudTrail Lake or Athena.
4. **Config rules and conformance packs**: This estimate covers CI recording costs only. Config rules and conformance pack evaluations are billed separately.
5. **Regional pricing**: Pricing may vary by region. Always verify at https://aws.amazon.com/config/pricing/
6. **Periodic vs continuous**: Periodic recording generates 1 CI per **unique resource** per day — a resource modified 100 times = 1 CI. Continuous recording generates 1 CI per change — 100 modifications = 100 CIs. Periodic is cheaper for frequently-changing resources; continuous is cheaper for rarely-changing resources.
7. **Organization-wide**: If using an organization trail, results include all member accounts. Per-account breakdowns help identify which accounts drive the most cost.
8. **Use the AWS Pricing Calculator** for official estimates: https://calculator.aws/#/createCalculator/Config
