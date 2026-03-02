# AWS Elastic Disaster Recovery (DRS) -- Cross-Account & Cross-Region Setup Guide

## Scenario Overview

This document explains the step-by-step Disaster Recovery (DR) setup
using **AWS Elastic Disaster Recovery (DRS)**.

### Source (Primary) Environment

-   Region: ap-south-1 (Mumbai)
-   AWS Account: Production (DC Account)
-   Server Type: Ubuntu EC2 Application Server

### Target (DR) Environment

-   Region: ap-south-2 (Hyderabad)
-   AWS Account: DR Account (Separate AWS Account)
-   Connectivity: Internet-based replication (Public Subnet)

This setup enables continuous block-level replication from Mumbai to
Hyderabad in a separate AWS account.

------------------------------------------------------------------------

# Architecture Summary

1.  Source server runs AWS Replication Agent.
2.  Agent continuously replicates disk data to staging area in DR region.
3.  AWS DRS manages replication servers automatically.
4.  In case of disaster, Recovery Job launches EC2 instance in DR region.

------------------------------------------------------------------------

# Detailed Step-by-Step Implementation

## Step 1 -- Login to DR AWS Account (Hyderabad Region)

-   Login to DR AWS Account.
-   Switch region to **ap-south-2 (Hyderabad)**.

------------------------------------------------------------------------

## Step 2 -- Create IAM User for Replication Agent

Create an IAM user in DR account with:

### Required Policies:

-   `AWSElasticDisasterRecoveryAgentInstallationPolicy`

### Optional Policy (not strictly required):

-   `IAMReadOnlyAccess`

### After Creation:

-   Generate **Access Key**
-   Generate **Secret Key**
-   Store them securely (will be used during agent installation)

------------------------------------------------------------------------

## Step 3 -- Configure AWS Elastic Disaster Recovery in Hyderabad

1.  Go to **AWS Elastic Disaster Recovery (DRS)** service.
2.  When prompted, configure replication settings.

### Replication Settings:

-   Choose **Public Subnet** (since replication traffic is over internet).
-   Leave other settings as default unless specific customization is required.

⚠ Note: Replication server will NOT be created immediately. It gets
created automatically after agent installation on source server.

### Add Trusted Account:

-   Navigate to: Settings → Trusted Accounts
-   Add the Mumbai (Source) AWS Account ID.
-   Save configuration.

This allows cross-account replication.

------------------------------------------------------------------------

## Step 4 -- Login to Source Server (Mumbai)

SSH into the source server:

``` bash
ssh -i "your-key.pem" ubuntu@<source-private-ip>
```

Make sure: - Python3 is installed - Server has internet access -
Security groups allow outbound HTTPS (443)

------------------------------------------------------------------------

## Step 5 -- Install AWS Replication Agent on Source Server

### Download Installer from Hyderabad Regional Endpoint:

``` bash
wget https://aws-elastic-disaster-recovery-ap-south-2.s3.ap-south-2.amazonaws.com/latest/linux/aws-replication-installer-init.py
```

### Run Installer:

``` bash
sudo python3 aws-replication-installer-init.py
```

### During Installation, Provide:

-   AWS Region: `ap-south-2`
-   IAM Access Key
-   IAM Secret Key
-   Disk selection:
    -   Press Enter to replicate all disks
    -   Or specify devices (e.g., /dev/nvme0n1)

### Installation Behavior:

-   Agent installs required dependencies.
-   Registers source server with DRS.
-   Starts block-level replication.
-   Process may take 20--30 minutes depending on disk size.

------------------------------------------------------------------------

## Step 6 -- Verify Replication in DR Region

After successful installation:

### Check EC2 Console (Hyderabad)

You will see: - A **Replication Server** created automatically. -
Instance type typically t3.small (default staging configuration).

### Check DRS Console

Go to: - DRS → Source Servers

You should see: - Source server listed - Data replication status:
Initial Sync - Eventually status changes to: **Ready** - Data
replication status: Healthy

⚠ Important: Install replication agent on one server at a time to avoid
staging overload.

------------------------------------------------------------------------

## Step 7 -- Configure Launch Settings

In DRS Console:

1.  Go to **Default Launch Settings**
2.  Configure:
    -   Target VPC
    -   Subnet
    -   Security Group
    -   Instance Type
    -   IAM Role
    -   Key Pair

These settings define how your DR EC2 instance will launch.

------------------------------------------------------------------------

## Step 8 -- Perform Recovery Test

To test DR:

1.  Select Source Server
2.  Click **Initiate Recovery Job**
3.  Choose:
    -   Launch Type: Test Instance
4.  Launch recovery

DRS will: - Create EBS volumes from replicated data - Launch EC2
instance in Hyderabad - Attach volumes - Boot instance

------------------------------------------------------------------------

# Failover Procedure (Actual Disaster)

In case of real disaster:

1.  Stop application in primary (if possible).
2.  Initiate Recovery Job.
3.  Choose "Launch Recovery Instance".
4.  Update:
    -   DNS records
    -   Load Balancer target groups
    -   Route53 records
5.  Validate application functionality.

------------------------------------------------------------------------

# Failback Process

After primary region is restored:

1.  Re-protect original region.
2.  Reverse replication.
3.  Perform recovery back to Mumbai region.
4.  Cut traffic back to primary.

------------------------------------------------------------------------

# Best Practices

-   Enable EBS encryption.
-   Use PrivateLink or VPN instead of internet (recommended for production).
-   Monitor replication lag.
-   Perform DR drill every quarter.
-   Use CloudWatch alarms for replication health.
-   Restrict IAM keys after installation (rotate or delete).

------------------------------------------------------------------------

# Important Notes

-   Replication is continuous and near real-time.
-   You are billed for:
    -   Staging area resources
    -   EBS storage
    -   Data transfer
-   Always test DR before considering setup production-ready.

------------------------------------------------------------------------

# Conclusion

This guide covers complete cross-account, cross-region Disaster Recovery
setup using AWS Elastic Disaster Recovery between:

-   Primary: Mumbai (ap-south-1)
-   DR: Hyderabad (ap-south-2)

Following these steps ensures high availability and business continuity.
