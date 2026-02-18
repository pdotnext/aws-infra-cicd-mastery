## Architecture (Week 1 – Foundation)

Week 1 focuses on building a clean foundational infrastructure architecture with proper separation of concerns.
The infrastructure is divided into three independent CloudFormation stacks:

### 1. Networking Stack

- VPC (custom CIDR)
- Internet Gateway
- Multi-AZ Public Subnets
- Route Table with default route to IGW
- Cross-stack exports for VPC and Subnet IDs

Design considerations:

- Region-agnostic AZ selection using `!GetAZs`
- Multi-AZ readiness from day one
- Deterministic routing configuration
- Exported values for compute layer reuse

---

### 2. Identity Stack

- IAM Role with least privilege permissions
- Instance Profile for EC2/ASG attachment
- Explicit export of Instance Profile name
- Separation of role trust policy and inline permissions

Security considerations:

- IMDSv2 enforcement in compute layer
- Principle of least privilege
- No long-lived credentials
- Role-based instance access

---

### 3. Compute Stack

- Launch Template with:
  - Dynamic SSM-based AMI resolution
  - IMDSv2 enforcement
  - Explicit network interface configuration
  - Security Group attachment
- Auto Scaling Group:
  - Multi-AZ deployment
  - Rolling update strategy
  - UpdatePolicy for controlled instance replacement
  - Change Set driven deployment review

Design considerations:

- Avoid single EC2 instance fragility
- Support safe rolling updates
- Explicit LaunchTemplate version tracking
- Separation of control plane (CloudFormation) and data plane (ASG runtime behavior)

### Architecture Diagram (Week 1)

                          Internet
                              |
                       +----------------+
                       | InternetGateway|
                       +----------------+
                              |
                     -------------------------
                     |                       |
            +----------------+       +----------------+
            | PublicSubnetA  |       | PublicSubnetB  |
            |  (AZ-A)        |       |  (AZ-B)        |
            +----------------+       +----------------+
                     |                       |
                     |                       |
                 ---------------------------------
                 |       Auto Scaling Group      |
                 |  Min:1  Max:2  Multi-AZ      |
                 ---------------------------------
                               |
                       +------------------+
                       |  Launch Template |
                       |------------------|
                       | - SSM AMI        |
                       | - IMDSv2         |
                       | - SecurityGroup  |
                       | - InstanceProfile|
                       +------------------+
                               |
                        +----------------+
                        |  IAM Role      |
                        |  (Least Priv)  |
                        +----------------+

## Deployment Order

The infrastructure is deployed in a strict dependency-aware order to ensure cross-stack imports resolve correctly.

1. **Networking Stack**
   - Creates VPC, Internet Gateway, Route Table, and Public Subnets.
   - Exports VPC ID and Subnet IDs for downstream stacks.
   - Forms the foundational network layer.

2. **Identity Stack**
   - Creates IAM Role and Instance Profile.
   - Exports Instance Profile name.
   - Required by the compute layer for secure instance access.

3. **Compute Stack**
   - Creates Launch Template.
   - Deploys Auto Scaling Group across multiple AZs.
   - Imports networking and identity stack exports.

Deployment must follow this order because the compute stack relies on exported values from both networking and identity stacks.

## Cleanup Order

Stacks must be deleted in reverse dependency order:

1. Compute Stack
2. Identity Stack
3. Networking Stack

CloudFormation prevents deletion of stacks that export values currently in use by other stacks.

## Health Model Overview (ALB vs ASG)

The system uses two independent health mechanisms that interact during deployments and scaling events:

### Application Load Balancer (ALB)

- Acts as ingress for the application.
- Distributes traffic only to _healthy_ targets.
- Performs HTTP health checks based on:
  - Interval
  - Timeout
  - HealthyThreshold
  - UnhealthyThreshold
- Controls traffic routing decisions.

### Auto Scaling Group (ASG)

- Manages instance lifecycle (launch, terminate, replace).
- Can use:
  - `EC2` health checks (instance state only), or
  - `ELB` health checks (inherits ALB health status).
- Uses `HealthCheckGracePeriod` to prevent premature termination during startup.

### Important Insight

When `HealthCheckType = ELB`, a timing misalignment between:

- Application startup time
- Health check thresholds
- Timeout values
- Grace period

can introduce race conditions that may lead to cascading instance replacement.

Proper tuning of these parameters is critical for production stability.

## Rolling Update Strategy

### Launch Template Version Tracking

The Auto Scaling Group explicitly references:
`Version: !GetAtt LaunchTemplate.LatestVersionNumber`

This ensures that any change to the Launch Template (e.g., AMI updates, instance type changes, network configuration updates) is reflected in the ASG during stack updates.

Without explicit version tracking, the ASG could continue using an outdated Launch Template version, introducing configuration drift between desired state and runtime state.

---

### UpdatePolicy (AutoScalingRollingUpdate)

The ASG uses an `UpdatePolicy` to control how instance replacement occurs during stack updates.

Key parameters:

- `MinInstancesInService`
- `MaxBatchSize`
- `PauseTime`
- `WaitOnResourceSignals`

This enables:

- Controlled rolling updates
- Zero or minimal downtime deployments
- Safe replacement of instances when Launch Template versions change

Without an UpdatePolicy, CloudFormation would update the ASG configuration without orchestrating safe instance replacement.

## Change Set Governance

Production deployments should use explicit Change Sets instead of directly executing `aws cloudformation deploy`.

Change Sets provide:

- **Visibility** — Clear identification of `Modify`, `Add`, `Remove`, and `Replace` actions.
- **Impact Analysis** — Detection of resource replacement risks (e.g., ASG replacement, subnet changes).
- **Intent Verification** — Validation that the proposed changes align with the intended design update.

This is especially important for high-risk changes such as:

- Subnet modifications (`VPCZoneIdentifier`)
- Launch Template updates
- IAM modifications
- Resource replacement actions

Change Sets allow infrastructure changes to be reviewed before execution, reducing the risk of unintended destructive updates in production environments.

## Lessons Learned (Week 1)

At the beginning of this journey, my focus was primarily on writing CloudFormation templates correctly.

By the end of Week 1, my perspective shifted significantly.

### Mindset Shift

- From writing YAML → to understanding lifecycle behavior.
- From deploying resources → to analyzing production impact.
- From isolated stacks → to layered architecture design.
- From successful creation → to safe update orchestration.
- From configuration → to governance and change review discipline.

### Key Technical Insights

- ALB and ASG have independent health systems that must be tuned carefully.
- `HealthCheckType = ELB` introduces orchestration coupling and potential race conditions.
- Timeout values can trigger cascading failures under load if misconfigured.
- LaunchTemplate version tracking prevents configuration drift.
- UpdatePolicy enables safe rolling deployments.
- Change Sets are essential for production governance.

This week introduced production-level nuances that are rarely covered in introductory courses.

## Week 2 Plan (Preview)

Week 2 will expand the foundation into traffic management, deployment automation, and infrastructure governance.

### Phase 1 – Load Balancing & Traffic Control

- Integrate Application Load Balancer (ALB)
- Configure Target Groups
- Implement secure traffic flow (ALB → ASG)
- Explore blue/green deployment patterns
- Model traffic shifting behavior and failure scenarios

### Phase 2 – CI/CD for Infrastructure

- Introduce deployment automation (Makefile or shell orchestration)
- Implement change-set-driven pipeline workflow
- Integrate with GitHub Actions (or AWS CodePipeline)
- Enforce branch protection and review requirements
- Introduce approval gates for production deployment

### Phase 3 – Infrastructure Validation & Security Controls

- Add `cfn-lint` validation
- Introduce CloudFormation Guard rules
- Enforce tagging standards
- Implement static security scanning for templates
- Formalize deployment governance checks

The objective of Week 2 is to transition from infrastructure creation to infrastructure automation and governance.
