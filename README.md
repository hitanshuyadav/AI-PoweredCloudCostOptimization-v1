# 🤖 AI-Powered Cloud Cost Optimization

An AWS-based cloud cost optimization system that automatically collects cloud resource information, cost data, and CloudWatch metrics, analyzes potential optimization opportunities, and uses **Amazon Bedrock** to generate human-readable recommendations.

The system uses AWS managed services to create a serverless cost-analysis workflow.

---

## 🎯 Project Objective

Cloud resources can generate unnecessary costs because of:

* Underutilized EC2 instances
* Unattached EBS volumes
* Unused Elastic IPs
* Over-provisioned resources
* Unnecessary snapshots
* Poor resource utilization
* Unexpected service costs

This project aims to automatically identify such opportunities and provide actionable recommendations.

The system follows:

```text
Collect → Analyze → Recommend → Explain → Store → Notify
```

---

# 🏗️ Architecture

```text
                         Amazon EventBridge
                                │
                                ▼
                       Collection Workflow
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
       Resource Collector   Cost Collector   Metrics Collector
            Lambda              Lambda            Lambda
              │                 │                 │
              │                 │                 │
              ▼                 ▼                 ▼
          AWS APIs        Cost Explorer       CloudWatch
              │                 │                 │
              └─────────────────┼─────────────────┘
                                │
                                ▼
                           DynamoDB
                                │
                                ▼
                  Recommendation Lambda
                                │
                       ┌────────┴────────┐
                       │                 │
                       ▼                 ▼
                  Rule Engine       Cost Analysis
                       │                 │
                       └────────┬────────┘
                                ▼
                         Amazon Bedrock
                                │
                                ▼
                    AI-generated explanation
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
                DynamoDB                  Amazon SNS
             Recommendation                 │
                History                     ▼
                                         Email
```

---

# ☁️ AWS Services Used

| AWS Service            | Purpose                                           |
| ---------------------- | ------------------------------------------------- |
| **AWS Lambda**         | Runs collection and recommendation logic          |
| **Amazon EventBridge** | Triggers the optimization workflow                |
| **Amazon EC2 APIs**    | Collects EC2 resource information                 |
| **Amazon EBS APIs**    | Collects EBS information                          |
| **Amazon RDS APIs**    | Collects RDS information                          |
| **Amazon S3 APIs**     | Collects S3 information                           |
| **AWS Cost Explorer**  | Retrieves cloud cost information                  |
| **Amazon CloudWatch**  | Retrieves resource utilization metrics            |
| **Amazon DynamoDB**    | Stores collected data and recommendation history  |
| **Amazon Bedrock**     | Generates AI-powered explanations                 |
| **Amazon SNS**         | Sends recommendations through notifications/email |
| **AWS IAM**            | Controls permissions between services             |

---

# 🔄 How It Works

## 1. EventBridge

Amazon EventBridge starts the cost optimization workflow on a schedule.

For example:

```text
Every day
    ↓
EventBridge
    ↓
Start cost analysis
```

This allows the system to periodically analyze the AWS environment without requiring a continuously running server.

---

# 2. Resource Collector Lambda

The Resource Collector Lambda discovers resources using AWS APIs through **Boto3**.

It can collect information such as:

```text
EC2
├── Instance ID
├── Instance Type
├── State
├── Region
├── Availability Zone
└── Tags

EBS
├── Volume ID
├── Size
├── State
└── Attachment

RDS
├── DB Identifier
├── Instance Class
├── Engine
└── Status

S3
└── Bucket information
```

The Lambda uses an **IAM execution role** to access the required AWS APIs.

No AWS access keys are hard-coded into the application.

---

# 3. Cost Collector Lambda

The Cost Collector retrieves AWS spending information using the **AWS Cost Explorer API**.

The collected information can include:

```text
Service
Region
Date
Usage
Cost
```

Example:

```text
EC2          $60.20
RDS          $42.10
S3           $12.50
NAT Gateway  $31.80
EBS           $8.20
```

This information is stored in DynamoDB for further analysis.

---

# 4. CloudWatch Metrics Collector

The Metrics Collector Lambda retrieves resource utilization metrics from Amazon CloudWatch.

For example, for EC2:

```text
CPUUtilization
NetworkIn
NetworkOut
DiskReadOps
DiskWriteOps
```

Example:

```text
Instance:
i-123456789

Average CPU:
4.2%

Maximum CPU:
12.4%

Network:
Low
```

Historical metrics can be used to avoid making recommendations from a single data point.

> **Note:** EC2 memory utilization generally requires the CloudWatch Agent or another mechanism that publishes memory metrics.

---

# 5. DynamoDB

DynamoDB acts as the central data store.

The system stores information collected from different AWS services.

### Resource information

```text
Resource ID
Resource Type
Region
Configuration
Tags
Status
```

### Cost information

```text
Service
Date
Region
Cost
Usage
```

### CloudWatch metrics

```text
Resource ID
Metric
Average
Maximum
Timestamp
Period
```

### Recommendations

```text
Recommendation ID
Resource ID
Recommendation Type
Current Cost
Potential Saving
Reason
AI Explanation
Status
Timestamp
```

---

# 🧠 6. Recommendation Engine

The Recommendation Lambda reads the collected data from DynamoDB.

The initial recommendation logic is **rule-based**.

For example:

```text
IF

EC2 CPU utilization < 10%
AND
instance has been running continuously
AND
sufficient historical data exists

THEN

Flag the instance as potentially underutilized.
```

The system can then estimate potential savings.

Example:

```text
Current instance:
t3.large

Estimated current cost:
$60/month

Potential smaller instance:
t3.medium

Estimated cost:
$30/month

Potential saving:
$30/month
```

The rule engine handles the deterministic part of the analysis.

---

# 🤖 7. Amazon Bedrock

Amazon Bedrock provides the AI layer.

The recommendation engine sends structured information to Bedrock.

Example:

```text
Resource:
i-123456789

Type:
t3.large

Average CPU:
4.2%

Maximum CPU:
12%

Runtime:
24/7

Monthly Cost:
$60

Potential Saving:
$30/month

Detected Issue:
Potential underutilization
```

Bedrock converts this information into a human-readable recommendation.

Example:

```text
EC2 Cost Optimization Recommendation

The instance i-123456789 appears to be
underutilized based on its recent utilization.

The instance has maintained an average CPU
utilization of approximately 4.2%.

Recommendation:
Evaluate whether the workload can run on a
smaller instance type.

Estimated potential saving:
$30/month.

Before making the change, verify memory
utilization and application performance during
peak workloads.
```

### Why use rules + AI?

The system does not depend entirely on an LLM for infrastructure decisions.

```text
AWS Data
   ↓
Rule Engine
   ↓
Detect optimization opportunity
   ↓
Calculate estimated saving
   ↓
Amazon Bedrock
   ↓
Generate human-readable explanation
```

This makes the recommendation process more predictable while still benefiting from AI.

---

# 📢 8. Amazon SNS

After Bedrock generates the recommendation, the result is sent to Amazon SNS.

```text
Recommendation Lambda
        │
        ▼
    Amazon Bedrock
        │
        ▼
Human-readable recommendation
        │
        ▼
       SNS
        │
        ▼
      Email
```

Example notification:

```text
AWS Cost Optimization Recommendation

Resource:
i-123456789

Potential Saving:
$30/month

Recommendation:
Evaluate downsizing the EC2 instance.

Reason:
The instance has maintained low utilization
during the analyzed period.

Before changing the instance, verify memory
and peak workload requirements.
```

---

# 🔐 Security

The system uses **AWS IAM** to control communication between Lambda and other AWS services.

Lambda functions use IAM execution roles with only the permissions required for their tasks.

For example, collection functions can have read-only permissions such as:

```text
ec2:DescribeI
```
