# Blume Application Terraform Configuration

This repository contains the Terraform configuration for deploying the network infrastructure for the **blume-application** in a modular and environment-specific manner. The setup creates a VPC with public, private, and database subnets, along with necessary networking components, in the AWS `us-east-1` region for the `dev` environment.

## Project Structure

```
blume-terraform/
├── modules/
│   └── vpc/
│       ├── main.tf          # VPC, subnets, single NAT Gateway, security groups, etc.
│       ├── variables.tf     # Input variables for the VPC module
│       ├── outputs.tf       # Outputs for VPC and subnet IDs
├── environments/
│   └── dev/
│       ├── main.tf          # Calls the VPC module for the dev environment
│       ├── variables.tf     # Declares variables for the dev environment
│       ├── terraform.tfvars # Variable values for the dev environment
│       ├── outputs.tf       # Environment-specific outputs
│       ├── provider.tf      # AWS provider configuration
```

### Components
- **VPC Module (`modules/vpc/`)**:
  - **VPC**: `blume-vpc-dev` (CIDR: 10.0.0.0/16)
  - **Public Subnets**: `blume-public-subnet-a/b/c-dev` (10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24)
  - **Private Subnets**: `blume-private-subnet-a/b/c-dev` (10.0.11.0/24, 10.0.12.0/24, 10.0.13.0/24)
  - **Database Subnets**: `blume-database-subnet-a/b/c-dev` (10.0.21.0/24, 10.0.22.0/24, 10.0.23.0/24)
  - **Gateways**:
    - **Internet Gateway**: `blume-igw-dev` for public subnets
    - **NAT Gateway**: Single NAT Gateway in `blume-public-subnet-a-dev` for private subnets
  - **Route Tables**:
    - Public route table with route to Internet Gateway
    - Single private route table with route to NAT Gateway
    - Database route table (local routing only)
  - **Security Groups**:
    - `blume-public-sg-dev`: Allows HTTP/HTTPS ingress
    - `blume-private-sg-dev`: Allows HTTP from public SG
    - `blume-database-sg-dev`: Allows MySQL (3306) from private SG
  - **RDS Subnet Group**: `blume-database-subnet-group-dev` for future RDS instances
  - **VPC Flow Logs**: Logs to CloudWatch (`/aws/vpc/blume-vpc-flow-logs-dev`)

- **Dev Environment (`environments/dev/`)**:
  - Configures the VPC module with `dev`-specific variables
  - No EC2 instances or S3 buckets are created

## Prerequisites

- **Terraform**: Version 1.5 or higher
- **AWS CLI**: Configured with credentials (`aws configure`)
- **AWS Permissions**:
  - `ec2:*` (VPC, subnets, NAT Gateway, EIP)
  - `iam:*` (VPC Flow Logs role)
  - `logs:*` (CloudWatch Logs for VPC Flow Logs)
- **AWS Region**: `us-east-1` (configurable in `terraform.tfvars`)

## Setup Instructions

1. **Clone the Repository**:
   ```bash
   git clone <repository-url>
   cd blume-terraform
   ```

2. **Configure AWS Credentials**:
   - Set up AWS credentials using one of the following:
     ```bash
     aws configure
     ```
     Or set environment variables:
     ```bash
     export AWS_ACCESS_KEY_ID=<your-access-key>
     export AWS_SECRET_ACCESS_KEY=<your-secret-key>
     export AWS_DEFAULT_REGION=us-east-1
     ```

3. **Navigate to the `dev` Environment**:
   ```bash
   cd environments/dev
   ```

4. **Initialize Terraform**:
   ```bash
   terraform init
   ```

5. **Review the Plan**:
   ```bash
   terraform plan
   ```
   This shows the resources to be created (VPC, subnets, NAT Gateway, etc.).

6. **Apply the Configuration**:
   ```bash
   terraform apply
   ```
   Confirm with `yes`.

## Outputs

Run `terraform output` to view:
- `vpc_id`: ID of the VPC
- `public_subnet_ids`: IDs of public subnets
- `private_subnet_ids`: IDs of private subnets
- `database_subnet_ids`: IDs of database subnets
- `public_sg_id`: Public Security Group ID
- `private_sg_id`: Private Security Group ID
- `database_sg_id`: Database Security Group ID
- `database_subnet_group_name`: RDS subnet group name

## Destroying the Infrastructure

To remove all resources:
1. Navigate to `environments/dev`:
   ```bash
   cd blume-terraform/environments/dev
   ```
2. Run:
   ```bash
   terraform destroy
   ```
   Confirm with `yes`.
3. Verify in the AWS Console (VPC section) that resources are deleted.

## Troubleshooting

- **Error: `AddressLimitExceeded`**:
  - Cause: AWS EIP limit (default: 5) exceeded.
  - Fix:
    - Release unused EIPs:
      ```bash
      aws ec2 describe-addresses --region us-east-1
      aws ec2 release-address --allocation-id <allocation-id> --region us-east-1
      ```
    - Or request an EIP limit increase via AWS Service Quotas.
- **State Conflicts**:
  - If partial deployment occurred, destroy and reapply:
    ```bash
    terraform destroy
    terraform apply
    ```
  - Backup state file before destroying:
    ```bash
    cp terraform.tfstate terraform.tfstate.backup
    ```

## Notes

- **Single NAT Gateway**: The configuration uses one NAT Gateway to minimize EIP usage and costs. For production, consider multiple NAT Gateways for high availability.
- **No EC2 or S3**: The setup excludes EC2 instances and S3 buckets, focusing on VPC infrastructure.
- **Extensibility**: Add resources like RDS (in `database_subnet_ids`) or ALB (in `public_subnet_ids`) by extending `environments/dev/main.tf`.