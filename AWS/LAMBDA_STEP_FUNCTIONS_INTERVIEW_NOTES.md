# AWS Lambda & Step Functions — Interview Notes

## 1. AWS Lambda

### What is AWS Lambda?

AWS Lambda is a **serverless, event-driven compute service**.

You provide the code and configuration, while AWS manages:
- Servers
- OS maintenance
- Scaling
- Availability
- Runtime infrastructure

Conceptually:

```text
Event / Request
      |
      v
API Gateway / S3 / SQS / EventBridge
      |
      v
   AWS Lambda
      |
      v
Database / S3 / AWS Service
```

Example:

```text
User
  |
  v
API Gateway
  |
  v
Lambda
  |
  v
DynamoDB
```

---

## 2. Key Lambda Features

### Serverless

No EC2 instance or OS management is required.

### Event Driven

Lambda can be triggered by services such as:
- API Gateway
- S3
- SQS
- SNS
- EventBridge
- DynamoDB Streams
- CloudWatch events

### Automatic Scaling

Lambda automatically creates more concurrent executions when request volume increases.

### Pay Per Use

You pay primarily based on:
- Number of requests
- Execution duration
- Allocated resources

### Stateless Design

Each invocation should normally be treated independently.

Persistent state should be stored externally, for example:
- DynamoDB
- S3
- RDS
- ElastiCache

### IAM Integration

Lambda uses IAM execution roles to control which AWS resources it can access.

### VPC Integration

Lambda can be configured to access private resources inside a VPC, such as:
- RDS
- ElastiCache
- Internal services

### Versions and Aliases

Useful for:
- Blue/green deployments
- Canary deployments
- Controlled releases

### Concurrency Controls

Lambda supports:
- Reserved concurrency
- Provisioned concurrency

Provisioned concurrency can help reduce cold-start latency.

### Monitoring

Integrates with:
- CloudWatch Logs
- CloudWatch Metrics
- AWS X-Ray

---

## 3. Maximum Lambda Runtime

A single Lambda invocation can run for a maximum of:

```text
15 minutes = 900 seconds
```

If a workload needs to run longer, consider:
- ECS
- EKS
- EC2
- AWS Batch
- Step Functions to coordinate multiple tasks

---

## 4. Why Use Lambda Instead of EC2?

Lambda is well suited for:
- Short-lived workloads
- Event-driven processing
- Bursty workloads
- APIs
- Automation
- File processing
- Queue consumers
- Scheduled tasks

Example:

```text
S3 Upload
   |
   v
Lambda
   |
   +--> Validate file
   +--> Resize image
   +--> Store metadata
```

This avoids keeping an EC2 server running continuously just to wait for events.

---

## 5. Why Can't We Use Lambda for Everything Instead of EC2?

Lambda and EC2 solve different workload requirements.

### 1. Lambda is for Short-Lived Execution

Lambda functions are not designed to run permanently.

```text
Request
   |
Lambda starts
   |
Performs task
   |
Returns result
   |
Execution ends
```

A continuously running Java, Python, or middleware service may be better suited to:
- EC2
- ECS
- EKS

### 2. Lambda is Stateless

You should not depend on a particular execution environment remaining available.

Bad design:

```text
Lambda memory
   |
Store session
   |
Expect next request on same environment
```

Better design:

```text
Lambda
   |
   v
DynamoDB / S3 / ElastiCache
```

### 3. Persistent Connections Can Be a Poor Fit

Applications that need:
- Long-lived TCP connections
- Persistent workers
- Long processing
- Streaming daemons
- Complex background services

are generally better suited to EC2/ECS/EKS.

### 4. Cold Starts

When Lambda creates a new execution environment, initialization introduces additional latency.

```text
Request
   |
Create runtime
   |
Load application
   |
Execute function
```

This is called a **cold start**.

Provisioned concurrency can reduce cold-start impact.

### 5. Continuous High Utilization

Lambda is attractive for unpredictable and bursty workloads.

```text
Bursty / Event Driven
        ↓
      Lambda
```

For continuously busy workloads:

```text
Continuous / Predictable
        ↓
EC2 / ECS / EKS may be more suitable
```

Cost should always be evaluated for the specific workload.

### 6. Limited Infrastructure Control

With EC2, you control:
- Operating system
- Packages
- Runtime
- Kernel configuration
- Agents
- Drivers
- Storage
- Networking

Lambda abstracts most of this.

That is an advantage operationally but a limitation if the workload requires low-level control.

### 7. Specialized Workloads

Examples that may be better suited to EC2/EKS:
- GPU workloads
- Large-memory applications
- Specialized drivers
- Stateful middleware
- Custom system software

---

## 6. Lambda vs EC2

| Lambda | EC2 |
|---|---|
| Serverless | Server based |
| Event driven | Long-running workloads |
| No OS management | Full OS control |
| Automatic scaling | Scaling configured by user |
| Pay per execution | Pay while instance is running |
| Stateless design | Can support stateful processes |
| Maximum invocation 15 minutes | Can run continuously |
| Best for bursty workloads | Good for steady workloads |
| Less infrastructure control | Full infrastructure control |

---

## 7. Good Lambda Use Cases

### API Backend

```text
API Gateway
   |
   v
Lambda
   |
   v
DynamoDB
```

### File Processing

```text
S3 Upload
   |
   v
Lambda
   |
   v
Process File
```

### Queue Processing

```text
SQS
 |
 v
Lambda
 |
 v
Process Message
```

### Scheduled Automation

```text
EventBridge Schedule
        |
        v
      Lambda
```

### SRE / Platform Automation

```text
CloudWatch / EventBridge
          |
          v
        Lambda
          |
          +--> Tag resources
          +--> Stop unused instances
          +--> Perform compliance checks
          +--> Trigger notifications
```

---

# 8. Lambda Interview Q&A

## Q1. What is AWS Lambda?

AWS Lambda is a serverless, event-driven compute service where AWS manages the underlying infrastructure and we provide the function code and trigger.

## Q2. What can trigger Lambda?

Common triggers include:
- API Gateway
- S3
- SQS
- SNS
- EventBridge
- DynamoDB Streams
- CloudWatch events

## Q3. What is the maximum Lambda execution time?

A single invocation can run for a maximum of **15 minutes (900 seconds)**.

## Q4. Why use Lambda instead of EC2?

Use Lambda for short-lived, stateless, event-driven, and bursty workloads because it removes server management and scales automatically.

## Q5. Why not use Lambda for every application?

Lambda is not ideal for long-running workloads, persistent connections, stateful processing, specialized compute, or applications requiring full OS control.

## Q6. What is a Lambda cold start?

A cold start occurs when AWS creates a new execution environment before running the function. This adds initialization latency.

## Q7. How do you reduce cold starts?

Options include:
- Keep deployment package small
- Reduce initialization work
- Reuse connections where appropriate
- Use Provisioned Concurrency for latency-sensitive functions

## Q8. How does Lambda access private RDS?

Configure Lambda for the appropriate VPC, private subnets, and Security Groups so it can reach the database privately.

## Q9. Is Lambda stateful?

Lambda should be designed as stateless. Persistent state should be stored in services such as DynamoDB, S3, RDS, or ElastiCache.

## Q10. How does Lambda scale?

Lambda automatically increases concurrent executions as event/request volume increases, subject to account and function concurrency controls.

---


## 8A. How EC2 or an Application Connects to Lambda

An EC2 instance or application does **not connect directly to a Lambda runtime using a server IP**.

It normally invokes Lambda through an AWS service/API.

### Option 1 — Direct Lambda Invocation

An EC2-hosted application can invoke Lambda through the AWS SDK / Lambda Invoke API.

```text
EC2 / Application
      |
      | AWS SDK / HTTPS
      v
Lambda Invoke API
      |
      v
Lambda Function
```

The EC2 instance uses an IAM instance role with:

```text
lambda:InvokeFunction
```

This is useful for internal service-to-service invocation.

---

### Option 2 — API Gateway

For an HTTP/REST API:

```text
Application / Client
      |
      | HTTPS
      v
API Gateway
      |
      v
Lambda
```

API Gateway can provide:

- Authentication / authorization
- Throttling
- Rate limiting
- Request validation
- API routing

Use this when Lambda is the backend of an application API.

---

### Option 3 — Asynchronous Invocation through SQS / SNS / EventBridge

For decoupled processing:

```text
Application
    |
    v
   SQS
    |
    v
 Lambda
```

or:

```text
Application
    |
EventBridge
    |
    v
 Lambda
```

This is useful when the application should not wait for the Lambda execution to complete.

---

### Option 4 — Private EC2 to Lambda through a VPC Endpoint

If EC2 is in a private subnet and you do not want the Lambda API call to use a NAT Gateway:

```text
Private EC2
     |
     v
Lambda Interface VPC Endpoint
Private IP / ENI
     |
 AWS PrivateLink
     |
     v
Lambda Service
     |
     v
Lambda Function
```

Without the endpoint:

```text
Private EC2
     |
NAT Gateway
     |
Lambda Public Service Endpoint
     |
Lambda
```

With an Interface VPC Endpoint:

```text
Private EC2
     |
vpce-Lambda
     |
PrivateLink
     |
Lambda Service
```

No NAT or Internet Gateway is required for the Lambda API call.

---

## 8B. Important VPC Direction Distinction

A common interview trap is:

> “Do I put Lambda inside my VPC so EC2 can invoke it?”

No.

For EC2 to invoke Lambda:

```text
EC2 -> Lambda
= Lambda Invoke API / API Gateway / Event Service
```

Lambda VPC configuration is mainly for the opposite direction:

```text
Lambda -> Private RDS / Redis / EC2 / Internal Service
= Lambda configured for VPC access
```

Example:

```text
Lambda
  |
VPC networking
  |
Private Subnet Resources
  |
RDS / ElastiCache / Internal Service
```

So remember:

```text
EC2 -> Lambda
does NOT require Lambda to be VPC-enabled

Lambda -> Private VPC Resource
does require appropriate Lambda VPC configuration
```

---

## 8C. IAM Roles Involved in Lambda Invocation

There are three different IAM concepts to distinguish.

### 1. Caller IAM Role

For direct EC2-to-Lambda invocation:

```text
EC2
 |
IAM Instance Role
 |
lambda:InvokeFunction
 |
 v
Lambda
```

Example permission:

```json
{
  "Effect": "Allow",
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:ap-south-1:123456789012:function:process-order"
}
```

The EC2 role trust policy allows EC2 to assume the role:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "ec2.amazonaws.com"
  },
  "Action": "sts:AssumeRole"
}
```

Easy memory:

> Caller role = Who can invoke the Lambda?

---

### 2. Lambda Execution Role

Once the function starts, Lambda assumes its own execution role.

```text
Lambda Function
      |
Lambda Execution Role
      |
      +--> CloudWatch Logs
      +--> S3
      +--> DynamoDB
      +--> Secrets Manager
      +--> Other AWS APIs
```

Trust policy:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "lambda.amazonaws.com"
  },
  "Action": "sts:AssumeRole"
}
```

Easy memory:

> Lambda execution role = What can the Lambda do after it runs?

---

### 3. Lambda Resource-Based Policy

Used when another AWS service or another AWS account invokes the function.

Example with API Gateway:

```text
API Gateway
     |
Lambda Resource Policy
     |
     v
Lambda
```

Conceptually the policy allows:

```text
Principal: apigateway.amazonaws.com
Action: lambda:InvokeFunction
```

Easy memory:

> Lambda resource policy = Which AWS service/account is allowed to invoke this function?

---

## 8D. IAM by Invocation Pattern

### Direct EC2 -> Lambda

```text
EC2
 |
EC2 IAM Role
lambda:InvokeFunction
 |
 v
Lambda
 |
Lambda Execution Role
 |
AWS Services
```

Required:

- EC2 role with `lambda:InvokeFunction`
- Lambda execution role for whatever the function itself needs to access

---

### EC2 -> API Gateway -> Lambda

```text
EC2 / Client
     |
     v
API Gateway
     |
Lambda Resource Policy
     |
     v
Lambda
     |
Lambda Execution Role
```

Potential controls:

- Client/API authentication
- API Gateway authorization
- Lambda resource-based permission allowing API Gateway
- Lambda execution role

---

### EC2 -> SQS -> Lambda

```text
EC2
 |
EC2 IAM Role
sqs:SendMessage
 |
 v
SQS
 |
 v
Lambda
```

EC2 role needs:

```text
sqs:SendMessage
```

Lambda execution role typically needs permissions required for the SQS event-source integration, such as:

```text
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

---

### Application -> EventBridge -> Lambda

```text
Application
 |
IAM Role
events:PutEvents
 |
 v
EventBridge
 |
Lambda Resource Policy
 |
 v
Lambda
```

The application role publishes the event.

Lambda's resource-based policy allows EventBridge to invoke the function.

---

### EC2 -> Lambda through Interface VPC Endpoint

```text
EC2
 |
EC2 IAM Role
lambda:InvokeFunction
 |
Lambda Interface Endpoint
 |
PrivateLink
 |
Lambda
```

The IAM role requirement does not fundamentally change.

You may additionally apply a **VPC Endpoint Policy** to restrict which Lambda functions can be accessed through that endpoint.

Conceptually:

```text
EC2 IAM Role
      +
VPC Endpoint Policy
      +
Lambda Permissions
```

---

## 8E. Interview Q&A — Lambda Connectivity and IAM

### Q11. How can an EC2 instance invoke Lambda?

It can invoke Lambda directly using the AWS SDK / Lambda Invoke API. The EC2 instance profile role needs `lambda:InvokeFunction` permission on the target Lambda.

### Q12. Does Lambda need to be inside the VPC for EC2 to invoke it?

No. EC2 invokes the Lambda service API. Lambda VPC configuration is mainly required when the Lambda function itself needs to access private VPC resources such as RDS, Redis, or internal services.

### Q13. How can a private EC2 invoke Lambda without NAT?

Create an Interface VPC Endpoint for the Lambda service. The EC2 instance then accesses the Lambda service privately through AWS PrivateLink.

### Q14. What IAM role does EC2 need?

The EC2 instance profile role needs permission such as:

```text
lambda:InvokeFunction
```

for the specific target Lambda ARN.

### Q15. What is the Lambda execution role?

It is the role assumed by the Lambda service while executing the function. It controls what the function can access after invocation, such as S3, DynamoDB, Secrets Manager, or CloudWatch.

### Q16. What is a Lambda resource-based policy?

It defines which AWS services, accounts, or principals are allowed to invoke the Lambda. For example, API Gateway or EventBridge is commonly granted invoke permission through the Lambda resource policy.

### Q17. Caller role vs Lambda execution role?

```text
Caller Role
= Who can invoke Lambda?

Lambda Execution Role
= What can Lambda do after invocation?

Lambda Resource Policy
= Which service/account can invoke the function?
```

### Q18. When would you use API Gateway instead of direct Lambda invocation?

Use API Gateway when the Lambda is exposed as an HTTP/API backend and you need authentication, throttling, routing, request validation, or API management.

### Q19. When would you use SQS instead of direct invocation?

Use SQS for asynchronous, decoupled processing when the caller does not need to wait for Lambda to finish and you want buffering/retry behavior.

---


# 9. AWS Step Functions

## What is AWS Step Functions?

AWS Step Functions is a **serverless workflow orchestration service**.

It coordinates multiple tasks and AWS services in a defined sequence using a **state machine**.

Example:

```text
API Request
   |
   v
Step Functions
   |
   +--> Lambda: Validate
   |
   +--> Lambda: Process Payment
   |
   +--> DynamoDB: Update Order
   |
   +--> SNS: Send Notification
```

---

## 10. Why Use Step Functions?

Without Step Functions, a single Lambda can become overloaded with:
- Sequencing logic
- Retry logic
- Wait logic
- Error handling
- Branching
- Multiple service calls

Instead:

```text
Step Functions
   |
   +-- Validate
   +-- Process
   +-- Retry
   +-- Update DB
   +-- Notify
```

Key principle:

> Lambda executes a task. Step Functions coordinates multiple tasks.

---

## 11. Step Functions Features

### Sequencing

Run tasks one after another.

```text
Task A
  |
  v
Task B
  |
  v
Task C
```

### Conditional Branching

```text
Payment Successful?
      /       \
    Yes       No
     |         |
Update Order  Retry / Fail
```

### Retry

Automatically retry failed tasks.

### Catch / Error Handling

Route failed tasks to a fallback or error-handling path.

### Wait States

Pause workflow execution for a defined period.

### Parallel Execution

Run multiple branches simultaneously.

```text
        Start
          |
      -----------
      |         |
   Task A     Task B
      |         |
      -----------
          |
         Next
```

### Workflow State

Step Functions maintains workflow state between tasks.

### Visual Workflow

The workflow can be visualized, which helps with troubleshooting and operations.

### AWS Service Integration

Step Functions can integrate directly with services such as:
- Lambda
- ECS
- SQS
- SNS
- DynamoDB
- API Gateway
- AWS Batch
- Glue

You do not always need Lambda between Step Functions and an AWS service.

---

## 12. Standard vs Express Workflows

### Standard Workflows

Best for:
- Durable workflows
- Long-running processes
- Auditable business processes
- Workflows requiring detailed execution history

Quick memory:

```text
Standard = Durable + Long Running
```

### Express Workflows

Best for:
- High-volume workloads
- Short-duration workflows
- Event processing
- High-throughput execution

Quick memory:

```text
Express = High Volume + Short Running
```

---

## 13. Step Functions Failure Handling

Example:

```text
Call Task
   |
   +-- Success -> Next Step
   |
   +-- Failure
          |
          v
        Retry
          |
     Still fails?
          |
          v
        Catch
          |
          v
   Error Handler
```

Step Functions supports:
- Retry
- Backoff
- Catch
- Fallback paths
- Timeout handling

---

## 14. Lambda + Step Functions Example

Suppose an order workflow has multiple steps.

```text
Customer
   |
   v
API Gateway
   |
   v
Step Functions
   |
   +--> Lambda: Validate Order
   |
   +--> Lambda: Check Inventory
   |
   +--> Lambda: Process Payment
   |
   +--> DynamoDB: Save Order
   |
   +--> SNS: Send Confirmation
```

If payment fails:

```text
Process Payment
      |
      X
      |
    Retry
      |
      X
      |
Compensation / Failure Handler
```

---

# 15. Step Functions Interview Q&A

## Q1. What is AWS Step Functions?

AWS Step Functions is a serverless workflow orchestration service used to coordinate multiple tasks and AWS services using a state machine.

## Q2. Why use Step Functions instead of one large Lambda?

A large Lambda becomes difficult to maintain when it includes sequencing, retry, wait, branching, and error-handling logic. Step Functions separates workflow logic from task execution.

## Q3. Standard vs Express Workflows?

**Standard**
- Durable
- Long-running
- Detailed execution history

**Express**
- High-volume
- Short-duration
- High-throughput event processing

## Q4. How does Step Functions handle failures?

It supports:
- Retry
- Backoff
- Catch
- Timeout
- Fallback paths

## Q5. Can Step Functions call services other than Lambda?

Yes. It can directly integrate with services including:
- ECS
- SQS
- SNS
- DynamoDB
- API Gateway
- AWS Batch
- Glue

## Q6. What is a state machine?

A state machine defines the sequence of workflow states, such as tasks, choices, waits, parallel execution, success, and failure.

## Q7. Can Step Functions run tasks in parallel?

Yes. Parallel states allow multiple branches to execute concurrently.

## Q8. Can Step Functions wait for a period?

Yes. A Wait state can pause execution based on time or timestamp.

## Q9. Where would you use Step Functions?

Typical use cases:
- Order processing
- Payment workflows
- Data pipelines
- Batch orchestration
- Multi-step automation
- Approval workflows

## Q10. Lambda vs Step Functions?

```text
Lambda
= performs one unit of work

Step Functions
= orchestrates multiple units of work
```

---

# 16. Interview-Ready Summary

### Lambda

> “AWS Lambda is a serverless, event-driven compute service. AWS manages the underlying servers, scaling, and runtime infrastructure while we provide the function code and trigger. It is best suited for short-lived, stateless, event-driven, and bursty workloads. It is not a replacement for EC2 in every scenario because long-running services, persistent connections, stateful workloads, specialized compute, and full OS-control requirements may be better suited to EC2, ECS, or EKS.”

### Step Functions

> “AWS Step Functions is a serverless orchestration service used to coordinate multiple tasks and AWS services. It provides sequencing, retries, branching, waits, parallel execution, and error handling. Lambda performs individual tasks, while Step Functions manages the overall workflow.”
