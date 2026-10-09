# ShoppersMart WordPress Infrastructure

Infrastructure-as-code for a highly available WordPress ecommerce site on AWS. CloudFormation provisions the network, application tier, database tier, edge security, shared storage, cache, backup plan, and monitoring. GitHub Actions validates and deploys the template using AWS IAM OIDC.

## Project status

The deployed `ecommerce-vpc` stack currently has the VPC, WordPress Auto Scaling group, ALB, CloudFront/WAF, EFS, RDS MySQL primary and cross-AZ read replica, and CloudWatch dashboard. Its public site is `https://d1s4xet1yd06um.cloudfront.net`.

The repository template also includes ElastiCache Redis object caching and AWS Backup. These additions are represented by the available CloudFormation change set `wordpress-redis-backup-readiness`; that change set has not been executed. Redis and AWS Backup resources described below are therefore planned until a stack update completes.

## Architecture

```text
Viewer (HTTPS)
    |
    v
CloudFront distribution ---- AWS WAF (managed Common/XSS and SQLi rules)
    |
    | HTTP origin connection; source restricted to CloudFront origin-facing IP ranges
    v
Application Load Balancer (public subnets)
    |
    v
WordPress EC2 Auto Scaling group (private server subnets, two AZs)
    |             |               |
    |             |               +-- Encrypted EFS shared wp-content
    |             +-- Private TLS Redis object cache (planned)
    |
    +-- MySQL primary (us-east-1b) -- asynchronous replication --> Read replica (us-east-1a)
             |
             +-- AWS Backup daily recovery points (planned)
```

### Network and security

- VPC CIDR: `10.0.0.0/16`.
- Two public subnets host the internet-facing ALB. Private server and database subnets are spread across two Availability Zones.
- Application and database instances have no public IP addresses. Private route tables have no general Internet/NAT route; an S3 gateway endpoint and a Secrets Manager interface endpoint provide required private service access.
- Security groups allow HTTP from the ALB to WordPress, MySQL from WordPress to RDS, NFS from WordPress to EFS, and (in the pending template) Redis TLS from WordPress to Redis.
- EC2 instance metadata uses IMDSv2. Application database credentials and the Redis authentication token are retrieved from Secrets Manager at startup.

### Web delivery and application

- CloudFront is the viewer-facing HTTPS endpoint and redirects HTTP viewers to HTTPS. The default CloudFront certificate is used.
- Dynamic requests are not cached by CloudFront; all methods WordPress needs, cookies, query strings, and required forwarding headers are passed to the ALB.
- The ALB serves as the CloudFront origin over HTTP. The ALB security group is restricted to the AWS-managed CloudFront origin-facing prefix list.
- AWS WAF has a CloudFront-scoped web ACL with `AWSManagedRulesCommonRuleSet` (including XSS protection) and `AWSManagedRulesSQLiRuleSet`. `SizeRestrictions_BODY` in the common group is configured to count rather than block.
- The EC2 Auto Scaling group has desired/minimum capacity 2, maximum capacity 3, and CPU target tracking at 70%.
- Encrypted EFS is mounted by instances and shared at `wp-content`, so uploads and plugins are available across instances.

### Database and caching

- RDS MySQL is private and encrypted. The primary is in `us-east-1b`; its read replica is in `us-east-1a`.
- HyperDB directs writes to the primary and prefers the read replica for read-only queries. RDS read replication is asynchronous. HyperDB directs reads after writes in the same request to the primary; a later request may briefly observe replica lag.
- The template changes the primary from RDS-managed master-password management to a static Secrets Manager copy so that the primary supports a read replica. Password changes and rotation must be coordinated manually between the primary and copied secret.
- The pending Redis design uses a two-node ElastiCache replication group across private server subnets, with at-rest encryption, in-transit TLS, an auth token stored in Secrets Manager, Multi-AZ, and automatic failover.
- WordPress's Redis Object Cache drop-in is installed into shared `wp-content`; `WP_CACHE` and the Redis host, port, TLS scheme, password, and key prefix are set by instance bootstrap.

### Backups and monitoring

- The pending AWS Backup plan stores backups in a new vault and runs daily at 05:00 UTC with 30-day retention.
- Backup selections cover the primary RDS instance, EFS file system, and EC2 instances tagged by the WordPress Auto Scaling group. The read replica is not a separate backup target.
- The `Ecommerce-CMS` CloudWatch dashboard displays EC2/Auto Scaling CPU utilization, primary RDS CPU utilization, and ALB request count.
- Configure AWS Backup notifications/monitoring and test restores; a configured backup plan alone does not prove recoverability.

## Repository contents

| Path | Purpose |
| --- | --- |
| `template.yaml` | Complete CloudFormation stack and outputs. |
| `.github/workflows/deploy-cloudformation.yml` | Template validation and deployment on pushes to `main`. |
| `README.md` | Project architecture, prerequisites, deployment, and operational notes. |

## Deployment prerequisites

### AWS account

The workflow is configured for AWS account `525391386782`, region `us-east-1`, and stack name `ecommerce-vpc`. It uses:

- An IAM OIDC provider for `token.actions.githubusercontent.com`.
- `EcommerceGitHubActionsDeployRole`, trusted for this repository's `main` branch and allowed to assume the CloudFormation execution role.
- `EcommerceCloudFormationExecutionRole`, with permissions for the resources in the template, including EC2, RDS, IAM, EFS, ELB, Auto Scaling, CloudFront, WAF, Secrets Manager, ElastiCache, and AWS Backup. It must be able to create the stack's backup service role and pass it to `backup.amazonaws.com`.
- A running RDS primary/master secret when updating this existing stack. The workflow reuses the `DatabaseMasterSecretArn` stack parameter; if it is absent during initial setup, the workflow looks up the RDS master secret.

The execution role policy is account-specific and is not provisioned by this template. Review its resource scopes and permissions against the CloudFormation resources before deploying.

### WordPress AMI and GitHub settings

The `WORDPRESS_AMI_ID` GitHub repository Actions secret must identify a compatible AMI. For the Redis-enabled template, the AMI must include:

- WordPress, Apache, PHP, AWS CLI, and the `phpredis` PHP extension.
- HyperDB 1.9 at `/opt/hyperdb/db.php`.
- Redis Object Cache plugin files at `/opt/redis/redis-cache`.

The prepared Redis-ready AMI in `us-east-1` is `ami-0bc1a6582ef65149c`; the repository secret was updated to this ID. The workflow also accepts the GitHub Actions variable `AWS_REGION`; CloudFront-scoped WAF resources must be managed from `us-east-1`.

## Deployment

Every push to `main` triggers the workflow. **Pushing the pending template to `main` will create billable Redis and AWS Backup resources.** Review the CloudFormation change set and cost implications before merging or pushing infrastructure changes.

To deploy manually:

```bash
aws cloudformation validate-template \
  --template-body file://template.yaml \
  --region us-east-1

aws cloudformation deploy \
  --stack-name ecommerce-vpc \
  --template-file template.yaml \
  --region us-east-1 \
  --role-arn arn:aws:iam::525391386782:role/EcommerceCloudFormationExecutionRole \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    WordPressAmiId=ami-0bc1a6582ef65149c \
    DatabaseMasterSecretArn=<existing-rds-master-secret-arn>
```

Use the AMI ID and master secret ARN for the target account. Never place passwords or unencrypted secret contents in the template, command line, README, or source control.

## After deployment

1. Check the CloudFormation stack status and events for `UPDATE_COMPLETE`.
2. Review stack outputs for `WordPressUrl`, RDS endpoints, Redis endpoint, backup vault, and backup plan.
3. Confirm both ALB targets are healthy and both EC2 instances use the expected AMI.
4. Confirm the RDS replica is `available` and reports active replication.
5. Verify the CloudFront distribution is deployed and has the WAF ACL attached.
6. Confirm the Redis replication group is available and WordPress can reach it over TLS.
7. Confirm AWS Backup has created recovery points for the expected resources; perform a restore test.
8. Open `WordPressUrl` and complete the WordPress installation wizard if the site is not initialized.

## Operations and limitations

- RDS replica reads can lag behind writes; use the primary for operations requiring read-after-write consistency. Replica promotion is a separate operational action.
- RDS-managed password rotation is disabled by the static credential handoff required for MySQL read replicas. Coordinate any password update with Secrets Manager and the RDS primary.
- CloudFront-to-ALB traffic is HTTP; only viewer-to-CloudFront traffic uses HTTPS.
- Redis nodes and retained backup recovery points incur ongoing charges. Review retention, instance sizes, and AWS pricing for the target region.
- CloudFormation updates to the launch template do not by themselves guarantee existing EC2 instances are refreshed. Perform and monitor a rolling Auto Scaling instance refresh when changing AMIs.
- AWS Backup recovery points should be monitored and restore-tested regularly.

## Stack outputs

The template exports the VPC and subnet IDs; HTTPS WordPress URL; ALB URL; CloudFront distribution ID and WAF ARN; primary database and read-replica endpoints/AZ; Redis endpoint and port; backup vault and plan IDs; dashboard name; and EFS file system ID.
