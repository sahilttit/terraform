# Paydoh Terraform Infrastructure

Reusable Terraform codebase for Paydoh networking, security, observability, and connectivity in AWS.

## Scope Implemented

- VPC CIDR: `/16`
- Availability Zones:
  - `ap-south-1a`
  - `ap-south-1b`
- Total 6 subnets:
  - Public (2): ALB, NAT Gateway
  - Private App (2): Microservices (EC2/Docker), AI Chatbot
  - Private DB (2): RDS, MongoDB, Redis
- VPC Flow Logs enabled for all subnets to CloudWatch log group: `/prod/vpc`
- Security detections:
  - DB egress/data exfiltration threshold (`>500 MB`, high severity intent)
  - Port scan activity threshold (`>=20` events in `60s`, high severity intent)
- Network security controls:
  - Security Groups (includes SSH port `22` rule via allowed CIDRs)
  - Network ACLs as secondary defense layer
- Connectivity:
  - AWS Transit Gateway (for bank integration path)
  - AWS Direct Connect Gateway + optional private VIF (when connection ID is supplied)

## Directory Structure

```text
.
|-- main.tf
|-- providers.tf
|-- versions.tf
|-- variables.tf
|-- outputs.tf
|-- README.md
|-- modules
|   |-- network
|   |   |-- main.tf
|   |   |-- variables.tf
|   |   `-- outputs.tf
|   |-- security
|   |   |-- main.tf
|   |   |-- variables.tf
|   |   `-- outputs.tf
|   |-- observability
|   |   |-- main.tf
|   |   |-- variables.tf
|   |   `-- outputs.tf
|   `-- connectivity
|       |-- main.tf
|       |-- variables.tf
|       `-- outputs.tf
`-- env
    |-- prodtfavrs
    |   `-- terraform.tfvars
    `-- stagingtfvars
        `-- terraform.tfvars
```

> Note: Folder name `prodtfavrs` is kept as requested.

## Naming Convention

Resources are tagged/named in this pattern:

- `Env(prod/staging)-paydoh-servicename-availabilityzone` (where AZ-scoped)
- Examples:
  - `prod-paydoh-public-ap-south-1a`
  - `staging-paydoh-database-ap-south-1b`

## Prerequisites

- Terraform `>= 1.5.0`
- AWS credentials configured and valid for target account
- AWS CLI recommended for credential verification

Check credentials:

```powershell
aws sts get-caller-identity
```

## How to Run

From project root:

1. Initialize:

```powershell
terraform init
```

2. Format (optional):

```powershell
terraform fmt -recursive
```

3. Validate:

```powershell
terraform validate
```

4. Plan for Prod:

```powershell
terraform plan -var-file="env/prodtfavrs/terraform.tfvars"
```

5. Plan for Staging:

```powershell
terraform plan -var-file="env/stagingtfvars/terraform.tfvars"
```

6. Apply (example for Prod):

```powershell
terraform apply -var-file="env/prodtfavrs/terraform.tfvars"
```

## Important Inputs

Set in env tfvars:

- `vpc_cidr` (`/16`)
- `availability_zones` (2 AZs)
- `public_subnet_cidrs` (2 values)
- `private_app_subnet_cidrs` (2 values)
- `private_db_subnet_cidrs` (2 values)
- `allowed_ssh_cidrs`
- `enable_transit_gateway`
- `enable_direct_connect`
- `dx_connection_id` (required only to create private VIF)

## Direct Connect Notes

- This code creates:
  - DX Gateway
  - DX Gateway association to TGW
  - Optional private VIF
- The physical DX circuit speed (for example `1 Gbps`) is provisioned outside this Terraform stack.

## Security/Monitoring Notes

- Flow logs are enabled per-subnet and sent to `/prod/vpc`.
- CloudWatch metric filters/alarms are included for:
  - DB subnet exfiltration pattern
  - Port scan-like activity volume
- Tune thresholds as needed for production signal quality.

## PostgreSQL RDS (Current Standard)

- Engine is fixed to PostgreSQL in this stack.
- Multi-AZ is enabled (`primary` + synchronous standby in-region).
- Storage encryption uses a dedicated KMS key.
- Automated backups retain 7 days.
- RDS is private-only (`publicly_accessible = false`) in private DB subnets.
- Security group access is restricted to `sg-app-ec2` on `5432/3306` as configured.
- CloudWatch log export is enabled for `postgresql` logs.

## DR: Cross-Region Read Replica (ap-south-2)

- Controlled by `enable_rds_cross_region_replica`.
- When enabled, a PostgreSQL read replica is created in DR region (`ap-south-2`) using provider alias `aws.dr`.

### Promotion Procedure (Failover Runbook)

1. Confirm primary region outage and freeze writes at application layer.
2. Promote DR replica in `ap-south-2`:
   - AWS CLI: `aws rds promote-read-replica --db-instance-identifier <dr-replica-id> --region ap-south-2`
3. Wait until replica status becomes `available` and role is standalone primary.
4. Update application/database endpoint configuration to new primary endpoint.
5. Validate read/write transactions from app tier.
6. After primary region recovery, rebuild replication topology from new primary.

## Next Recommended Enhancements

- Add SNS notifications for CloudWatch alarms
- Add tighter NACL rules based on exact subnet CIDRs
- Add application modules (ALB, ASG/ECS, RDS, Redis, MongoDB)
