# Multi-Environment Terraform CI/CD Pipeline - Complete Setup Guide

This guide contains all steps needed to deploy a multi-environment Terraform pipeline using Azure DevOps and Google Cloud Platform.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Part 1: Import Repository to Azure DevOps](#part-1-import-repository-to-azure-devops)
4. [Part 2: Configure Google Cloud Platform](#part-2-configure-google-cloud-platform)
5. [Part 3: Configure Azure DevOps](#part-3-configure-azure-devops)
6. [Part 4: Update Configuration Files](#part-4-update-configuration-files)
7. [Part 5: Run the Pipeline](#part-5-run-the-pipeline)
8. [Part 6: Verify Deployment](#part-6-verify-deployment)
9. [Part 7: Cleanup](#part-7-cleanup)
10. [Troubleshooting](#troubleshooting)

---

## Overview

This project deploys infrastructure across three environments:

- **Development**: 1 VM (e2-micro), auto-deploys on code changes
- **Staging**: 2 VMs (e2-small), auto-deploys after Dev succeeds
- **Production**: 3 VMs (e2-medium), requires manual approval

Each environment includes:
- VPC network
- Subnet with specified CIDR
- Firewall rules (SSH access)
- Compute instances
- Separate state storage in GCS

**Estimated time**: 30-45 minutes for setup, 15-20 minutes for first deployment

**Estimated cost**: $0.10-0.50 if cleaned up immediately; $50-70/month if left running

---

## Prerequisites

Before starting, ensure you have:

1. **Azure DevOps Account**
   - Sign up at https://dev.azure.com
   - Free tier is sufficient

2. **Google Cloud Platform Account**
   - Sign up at https://cloud.google.com
   - Billing must be enabled
   - Credit card required (free tier available)

3. **Local Tools** (for configuration)
   - `gcloud` CLI installed and configured
   - `git` installed
   - Terminal/command line access

4. **Basic Knowledge**
   - Git fundamentals
   - Cloud computing concepts
   - Command line usage

---

## Part 1: Import Repository to Azure DevOps

### Step 1.1: Create Azure DevOps Project

1. Navigate to https://dev.azure.com
2. Click "New Project"
3. Enter project name: `terraform-multi-env`
4. Select visibility: Private
5. Click "Create"

### Step 1.2: Import from GitHub

1. In your project, go to "Repos" → "Files"
2. Click "Import" at the bottom of the page
3. Enter repository URL:
   ```
   https://github.com/YOUR-INSTRUCTOR/terraform-ado-pipeline.git
   ```
4. Click "Import"
5. Wait 30-60 seconds for import to complete

**Alternative method** (if import doesn't work):

```bash
# Clone from GitHub
git clone https://github.com/YOUR-INSTRUCTOR/terraform-ado-pipeline.git
cd terraform-ado-pipeline

# Add Azure DevOps remote
git remote add ado https://YOUR-ORG@dev.azure.com/YOUR-ORG/terraform-multi-env/_git/terraform-multi-env

# Push to Azure DevOps
git push ado main
```

---

## Part 2: Configure Google Cloud Platform

### Step 2.1: Create GCP Project

```bash
# Set a unique project ID
export PROJECT_ID="terraform-lab-$(whoami)-$(date +%s)"

# Create the project
gcloud projects create ${PROJECT_ID} --name="Terraform Multi-Env Lab"

# Link billing account (replace BILLING_ACCOUNT_ID with your actual billing account)
# Get billing account ID: gcloud beta billing accounts list
gcloud beta billing projects link ${PROJECT_ID} \
  --billing-account=BILLING_ACCOUNT_ID

# Set as default project
gcloud config set project ${PROJECT_ID}

# Save the project ID for later use
echo "Your Project ID: ${PROJECT_ID}"
echo ${PROJECT_ID} > ~/terraform-project-id.txt
```

### Step 2.2: Enable Required APIs

```bash
# Enable Compute Engine and Cloud Storage APIs
gcloud services enable \
  compute.googleapis.com \
  storage-api.googleapis.com \
  --project=${PROJECT_ID}
```

### Step 2.3: Create Service Account

```bash
# Create service account
gcloud iam service-accounts create terraform-pipeline \
  --display-name="Terraform CI/CD Pipeline" \
  --project=${PROJECT_ID}

# Grant Editor role to service account
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:terraform-pipeline@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/editor"

# Create and download service account key
gcloud iam service-accounts keys create ~/terraform-pipeline-key.json \
  --iam-account=terraform-pipeline@${PROJECT_ID}.iam.gserviceaccount.com

echo "Service account key saved to: ~/terraform-pipeline-key.json"
```

### Step 2.4: Create State Storage Buckets

```bash
# Create GCS buckets for Terraform state (one per environment)
for ENV in dev staging prod; do
  gsutil mb -p ${PROJECT_ID} \
    -l us-central1 \
    gs://${PROJECT_ID}-${ENV}-tfstate

  # Enable versioning for state history
  gsutil versioning set on gs://${PROJECT_ID}-${ENV}-tfstate

  echo "Created bucket: ${PROJECT_ID}-${ENV}-tfstate"
done

# Verify buckets were created
gsutil ls | grep tfstate
```

---

## Part 3: Configure Azure DevOps

### Step 3.1: Upload Service Account Key

1. In Azure DevOps, navigate to "Pipelines" → "Library"
2. Click the "Secure files" tab
3. Click "+ Secure file"
4. Upload the file `~/terraform-pipeline-key.json`
5. Ensure the filename is exactly: `terraform-pipeline-key.json`
6. Click "Authorize" (padlock icon) to allow pipeline access

### Step 3.2: Create Variable Group

1. In Library, click the "Variable groups" tab
2. Click "+ Variable group"
3. Name: `terraform-variables` (must match exactly)
4. Add the following variables:

| Variable Name | Value | Description |
|--------------|-------|-------------|
| GCP_PROJECT_ID | your-project-id | From Step 2.1 |
| GCP_REGION | us-central1 | GCP region |
| TF_VERSION | 1.6.0 | Terraform version |

5. Click "Save"

### Step 3.3: Create Environments

Create three separate environments:

**Environment 1: Development**
1. Go to "Pipelines" → "Environments"
2. Click "+ Create environment"
3. Name: `terraform-dev`
4. Description: `Development environment`
5. Click "Create"

**Environment 2: Staging**
1. Click "+ Create environment"
2. Name: `terraform-staging`
3. Description: `Staging environment`
4. Click "Create"

**Environment 3: Production (with approval)**
1. Click "+ Create environment"
2. Name: `terraform-prod`
3. Description: `Production environment`
4. Click "Create"
5. Click on the `terraform-prod` environment
6. Click the three dots (⋮) → "Approvals and checks"
7. Click "+" → "Approvals"
8. Add yourself as an approver
9. Enter instructions: "Review Terraform plan before approving"
10. Set minimum approvers: 1
11. Click "Create"

---

## Part 4: Update Configuration Files

You need to update the configuration files with your GCP project ID.

### Step 4.1: Clone Repository Locally

```bash
# Clone your Azure DevOps repository
git clone https://YOUR-ORG@dev.azure.com/YOUR-ORG/terraform-multi-env/_git/terraform-multi-env
cd terraform-multi-env

# Verify you're in the right directory
ls -la
```

### Step 4.2: Update Backend Configuration

Update the backend.tf file in each environment:

**For Dev environment:**
```bash
cat > terraform/environments/dev/backend.tf << EOF
terraform {
  backend "gcs" {
    bucket = "${PROJECT_ID}-dev-tfstate"
    prefix = "terraform/state"
  }
}
EOF
```

**For Staging environment:**
```bash
cat > terraform/environments/staging/backend.tf << EOF
terraform {
  backend "gcs" {
    bucket = "${PROJECT_ID}-staging-tfstate"
    prefix = "terraform/state"
  }
}
EOF
```

**For Production environment:**
```bash
cat > terraform/environments/prod/backend.tf << EOF
terraform {
  backend "gcs" {
    bucket = "${PROJECT_ID}-prod-tfstate"
    prefix = "terraform/state"
  }
}
EOF
```

### Step 4.3: Create terraform.tfvars Files

**For Dev environment:**
```bash
cat > terraform/environments/dev/terraform.tfvars << EOF
project_id     = "${PROJECT_ID}"
environment    = "dev"
region         = "us-central1"
subnet_cidr    = "10.0.1.0/24"
instance_count = 1
machine_type   = "e2-micro"
disk_size_gb   = 10
EOF
```

**For Staging environment:**
```bash
cat > terraform/environments/staging/terraform.tfvars << EOF
project_id     = "${PROJECT_ID}"
environment    = "staging"
region         = "us-central1"
subnet_cidr    = "10.1.1.0/24"
instance_count = 2
machine_type   = "e2-small"
disk_size_gb   = 20
EOF
```

**For Production environment:**
```bash
cat > terraform/environments/prod/terraform.tfvars << EOF
project_id     = "${PROJECT_ID}"
environment    = "prod"
region         = "us-central1"
subnet_cidr    = "10.2.1.0/24"
instance_count = 3
machine_type   = "e2-medium"
disk_size_gb   = 50
EOF
```

### Step 4.4: Commit and Push Changes

```bash
# Stage all changes
git add .

# Commit
git commit -m "Configure project with GCP project ID: ${PROJECT_ID}"

# Push to Azure DevOps
git push origin main
```

---

## Part 5: Run the Pipeline

### Step 5.1: Create Pipeline

1. In Azure DevOps, go to "Pipelines" → "Pipelines"
2. Click "New pipeline"
3. Select "Azure Repos Git"
4. Select your repository: `terraform-multi-env`
5. Choose "Existing Azure Pipelines YAML file"
6. Path: `/azure-pipelines.yml`
7. Click "Continue"
8. Review the pipeline YAML
9. Click "Run"

### Step 5.2: Authorize Permissions

On first run, the pipeline will request permissions. Click "Permit" for each:

1. **Variable group**: terraform-variables
2. **Secure file**: terraform-pipeline-key.json
3. **Environment**: terraform-dev
4. **Environment**: terraform-staging
5. **Environment**: terraform-prod

These are one-time authorizations.

### Step 5.3: Monitor Pipeline Execution

The pipeline executes in stages:

1. **Validate** (2-3 minutes)
   - Installs Terraform
   - Validates all environment configurations
   - Checks module syntax

2. **Deploy to Dev** (3-4 minutes)
   - Initializes Terraform
   - Creates plan
   - Applies changes
   - Deploys 1 VM

3. **Deploy to Staging** (4-5 minutes)
   - Initializes Terraform
   - Creates plan
   - Applies changes
   - Deploys 2 VMs

4. **Deploy to Production** (5-6 minutes)
   - **PAUSES for approval**
   - Click "Review" button
   - Review the deployment details
   - Click "Approve"
   - Deploys 3 VMs

**Total time**: 15-20 minutes for complete deployment

---

## Part 6: Verify Deployment

### Step 6.1: Check Pipeline Status

In Azure DevOps:
1. Go to "Pipelines" → "Pipelines"
2. Click on the running/completed pipeline
3. Verify all stages show green checkmarks
4. Review logs if needed

### Step 6.2: Verify GCP Resources

```bash
# List all compute instances
gcloud compute instances list --project=${PROJECT_ID}

# Expected output should show:
# - dev-vm-1 (e2-micro)
# - staging-vm-1, staging-vm-2 (e2-small)
# - prod-vm-1, prod-vm-2, prod-vm-3 (e2-medium)

# Get detailed information with IPs
gcloud compute instances list \
  --project=${PROJECT_ID} \
  --format="table(
    name,
    zone,
    machineType.basename(),
    networkInterfaces[0].accessConfigs[0].natIP:label=EXTERNAL_IP,
    status
  )"
```

### Step 6.3: Verify Networks

```bash
# List VPC networks
gcloud compute networks list --project=${PROJECT_ID}

# Expected: dev-vpc, staging-vpc, prod-vpc

# List subnets
gcloud compute networks subnets list --project=${PROJECT_ID}

# Expected: dev-subnet, staging-subnet, prod-subnet
```

### Step 6.4: Verify State Files

```bash
# Check state files exist in GCS
gsutil ls -r gs://${PROJECT_ID}-dev-tfstate/terraform/state/
gsutil ls -r gs://${PROJECT_ID}-staging-tfstate/terraform/state/
gsutil ls -r gs://${PROJECT_ID}-prod-tfstate/terraform/state/

# Each should contain default.tfstate
```

### Step 6.5: View Terraform Outputs (Optional)

```bash
# Set credentials
export GOOGLE_APPLICATION_CREDENTIALS=~/terraform-pipeline-key.json

# View dev outputs
cd terraform/environments/dev
terraform init
terraform output

# View staging outputs
cd ../staging
terraform init
terraform output

# View production outputs
cd ../prod
terraform init
terraform output
```

---

## Part 7: Cleanup

**IMPORTANT**: Clean up resources immediately after completing the lab to avoid ongoing charges.

### Step 7.1: Destroy Infrastructure

```bash
# Set credentials
export GOOGLE_APPLICATION_CREDENTIALS=~/terraform-pipeline-key.json
export PROJECT_ID=$(cat ~/terraform-project-id.txt)

# Change to repository directory
cd /path/to/terraform-multi-env

# Destroy production environment
cd terraform/environments/prod
terraform init
terraform destroy -auto-approve

# Destroy staging environment
cd ../staging
terraform init
terraform destroy -auto-approve

# Destroy development environment
cd ../dev
terraform init
terraform destroy -auto-approve

echo "All infrastructure destroyed"
```

### Step 7.2: Delete State Buckets

```bash
# Delete GCS buckets
gsutil -m rm -r gs://${PROJECT_ID}-dev-tfstate
gsutil -m rm -r gs://${PROJECT_ID}-staging-tfstate
gsutil -m rm -r gs://${PROJECT_ID}-prod-tfstate

echo "State buckets deleted"
```

### Step 7.3: Delete GCP Project (Optional)

```bash
# This will delete everything in the project
gcloud projects delete ${PROJECT_ID}

# Confirm when prompted
```

### Step 7.4: Remove Local Files

```bash
# Remove service account key
rm ~/terraform-pipeline-key.json
rm ~/terraform-project-id.txt

# Remove cloned repository (if desired)
cd ~
rm -rf terraform-multi-env
```

---

## Troubleshooting

### Issue: Pipeline permission errors

**Symptoms**: "This pipeline needs permission to access a resource"

**Solution**:
- Click "Permit" for each resource
- Verify names match exactly:
  - Variable group: `terraform-variables`
  - Secure file: `terraform-pipeline-key.json`
  - Environments: `terraform-dev`, `terraform-staging`, `terraform-prod`

### Issue: Backend initialization failed

**Symptoms**: "Error: Failed to get existing workspaces"

**Solution**:
```bash
# Verify bucket exists
gsutil ls gs://${PROJECT_ID}-dev-tfstate

# If not, create it
gsutil mb -p ${PROJECT_ID} -l us-central1 gs://${PROJECT_ID}-dev-tfstate
gsutil versioning set on gs://${PROJECT_ID}-dev-tfstate

# Verify backend.tf has correct bucket name
cat terraform/environments/dev/backend.tf
```

### Issue: GCP API not enabled

**Symptoms**: "Error: Error creating instance" or "API not enabled"

**Solution**:
```bash
# Enable required APIs
gcloud services enable compute.googleapis.com storage-api.googleapis.com --project=${PROJECT_ID}

# Verify APIs are enabled
gcloud services list --enabled --project=${PROJECT_ID}
```

### Issue: Service account permission denied

**Symptoms**: "Error: google: could not find default credentials"

**Solution**:
```bash
# Verify service account has correct role
gcloud projects get-iam-policy ${PROJECT_ID} \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:terraform-pipeline@*"

# Re-add Editor role if missing
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:terraform-pipeline@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/editor"
```

### Issue: Terraform state locked

**Symptoms**: "Error: Error acquiring the state lock"

**Solution**:
```bash
# View lock info
gsutil cat gs://${PROJECT_ID}-dev-tfstate/terraform/state/default.tflock

# Force unlock (use with caution)
cd terraform/environments/dev
terraform force-unlock LOCK_ID
```

### Issue: Quota exceeded

**Symptoms**: "Error: Quota 'CPUS' exceeded"

**Solution**:
```bash
# Check current quotas
gcloud compute project-info describe --project=${PROJECT_ID}

# Request quota increase in GCP Console
# Or use smaller machine types (e2-micro for all environments)
```

### Issue: Pipeline hangs at "Install Terraform"

**Symptoms**: Pipeline stuck after downloading Terraform

**Solution**:
- This should be fixed in the latest pipeline version
- Verify you have the latest code from the repository
- The pipeline now installs Terraform in `/tmp/terraform-install`

### Issue: No resources created

**Symptoms**: Pipeline succeeds but no VMs in GCP

**Solution**:
```bash
# Verify terraform.tfvars exists and has correct project_id
cat terraform/environments/dev/terraform.tfvars

# Check if resources were actually created
gcloud compute instances list --project=${PROJECT_ID}

# Review pipeline logs for errors
# In Azure DevOps: Pipelines → Click pipeline → View logs
```

---

## Testing Changes

To test that the pipeline works correctly, make a small change:

```bash
# Update dev VM size
cat > terraform/environments/dev/terraform.tfvars << EOF
project_id     = "${PROJECT_ID}"
environment    = "dev"
region         = "us-central1"
subnet_cidr    = "10.0.1.0/24"
instance_count = 1
machine_type   = "e2-small"
disk_size_gb   = 15
EOF

# Commit and push
git add terraform/environments/dev/terraform.tfvars
git commit -m "Test: Upgrade dev VM to e2-small"
git push origin main

# Watch pipeline automatically trigger and update only Dev environment
```

---

## Architecture Overview

```
Repository Structure:
terraform/
├── modules/
│   └── compute/          # Reusable infrastructure module
│       ├── main.tf       # VPC, subnet, firewall, VMs
│       ├── variables.tf  # Input variables
│       └── outputs.tf    # Output values
└── environments/
    ├── dev/              # Development environment
    ├── staging/          # Staging environment
    └── prod/             # Production environment

Pipeline Flow:
1. Validate → Validates all Terraform configurations
2. Dev → Auto-deploys to development
3. Staging → Auto-deploys after Dev succeeds
4. Production → Requires manual approval before deployment

State Management:
Each environment has its own GCS bucket:
- PROJECT_ID-dev-tfstate
- PROJECT_ID-staging-tfstate
- PROJECT_ID-prod-tfstate
```

---

## Key Features

- **Multi-environment deployment**: Dev, Staging, Production
- **Progressive deployment**: Changes flow through environments
- **Approval gates**: Production requires manual approval
- **State isolation**: Separate state files per environment
- **Modular design**: Reusable Terraform modules
- **Automated validation**: Checks on every commit
- **Secure credential management**: Uses Azure DevOps secure files
- **Infrastructure as Code**: All infrastructure defined in version control

---

## What You've Learned

By completing this lab, you've gained hands-on experience with:

- Multi-environment CI/CD pipeline design
- Terraform state management with remote backends
- Azure DevOps pipeline configuration and execution
- GCP infrastructure provisioning
- Service account and IAM management
- Approval gates and deployment controls
- Infrastructure as Code best practices

---

## Additional Resources

- **Terraform Documentation**: https://www.terraform.io/docs
- **Azure DevOps Pipelines**: https://docs.microsoft.com/azure/devops/pipelines/
- **GCP Compute Engine**: https://cloud.google.com/compute/docs
- **Terraform GCP Provider**: https://registry.terraform.io/providers/hashicorp/google/latest/docs

---

## Support

If you encounter issues not covered in this guide:

1. Review the Troubleshooting section above
2. Check Azure DevOps pipeline logs for error messages
3. Verify all configuration values match your setup
4. Ensure GCP APIs are enabled and quotas are sufficient
5. Contact your instructor or open an issue in the repository

---

**Remember**: Always clean up resources after completing the lab to avoid unnecessary charges.
