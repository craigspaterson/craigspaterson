# Approach for Terraform Deployments to Google Cloud Platform with GitHub Actions

This document outlines an approach for deploying Terraform configurations to Google Cloud Platform (GCP) environments using GitHub Actions. The solution accommodates multiple target environments (Alpha, Beta, Gamma, and Prod), each with a blue-green deployment strategy (Primary and Secondary), and uses provider modules and variable sets for configuration management.

---

## 1. **Repository Structure**
Organize the repository to support clear separation of configurations, variables, and workflows:

- **`/terraform/modules`**: Contains reusable Terraform provider modules.
- **`/terraform/stacks`**: Contains environment-specific configurations:
  - **`/terraform/stacks/alpha-blue`**
  - **`/terraform/stacks/alpha-green`**
  - **`/terraform/stacks/beta-blue`**, etc.
- **`/terraform/variables`**: Contains `.tfvars` files for each stack:
  - **`alpha-blue.tfvars`**
  - **`beta-green.tfvars`**, etc.
- **`/.github/workflows`**: Contains GitHub Actions workflow YAML files for deployment automation.

---

## 2. **Terraform State Management**
To ensure state consistency and support for multiple environments, use a shared GCP blob storage solution for Terraform remote state management.

### **GCP Blob Storage Setup**
1. Create a GCP storage bucket:
   ```bash
   gcloud storage buckets create terraform-state-storage --location=us-central1
   ```
2. Configure the bucket to support multiple application IDs and environments:
   - Use prefixes for separate state files per application and environment:
     ```
     <application_id>/<environment>/terraform.tfstate
     ```
3. Enable uniform bucket-level access control.

### **Remote State Configuration**
In each Terraform stack configuration, specify the backend configuration:
```hcl
terraform {
  backend "gcs" {
    bucket  = "terraform-state-storage"
    prefix  = "<application_id>/<environment>"
    project = "<gcp_project_id>"
  }
}
```

---

## 3. **Environment Variables and Secrets**
Sensitive and secret values are managed using GitHub Secrets. These secrets are passed into Terraform configurations using environment variable conventions.

### **Examples of Secrets in GitHub**
- `GCP_CREDENTIALS` (base64-encoded GCP service account key)
- `DATABASE_PASSWORD`
- `API_SECRET_KEY`

### **Passing Secrets to Terraform**
1. Define secrets in the GitHub repository settings under **Settings > Secrets and variables > Actions**.
2. Export secrets as environment variables in GitHub Actions workflows:
   ```yaml
   env:
     GOOGLE_CREDENTIALS: ${{ secrets.GCP_CREDENTIALS }}
     TF_VAR_database_password: ${{ secrets.DATABASE_PASSWORD }}
     TF_VAR_api_secret_key: ${{ secrets.API_SECRET_KEY }}
   ```

---

## 4. **GitHub Actions Workflow**
Use GitHub Actions to orchestrate Terraform deployments for each stack.

### **Example Workflow**
Create a workflow file (e.g., `.github/workflows/deploy.yml`):
```yaml
name: Terraform Deployment

on:
  push:
    branches:
      - main

jobs:
  deploy:
    name: Deploy Terraform
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [alpha-blue, alpha-green, beta-blue, beta-green, gamma-blue, gamma-green, prod-blue, prod-green]
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.3

      - name: Authenticate with GCP
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_CREDENTIALS }}

      - name: Initialize Terraform
        working-directory: terraform/stacks/${{ matrix.environment }}
        run: terraform init

      - name: Plan Terraform
        working-directory: terraform/stacks/${{ matrix.environment }}
        run: terraform plan -var-file=../../variables/${{ matrix.environment }}.tfvars

      - name: Apply Terraform
        working-directory: terraform/stacks/${{ matrix.environment }}
        run: terraform apply -var-file=../../variables/${{ matrix.environment }}.tfvars -auto-approve
        env:
          GOOGLE_CREDENTIALS: ${{ secrets.GCP_CREDENTIALS }}
          TF_VAR_database_password: ${{ secrets.DATABASE_PASSWORD }}
          TF_VAR_api_secret_key: ${{ secrets.API_SECRET_KEY }}
```

### **Key Features**
- **Matrix Strategy**: Automates deployments for all target environments (e.g., `alpha-blue`, `beta-green`).
- **Environment Variables**: Passes secrets securely to Terraform.
- **Steps**: Includes `init`, `plan`, and `apply` stages for Terraform.

---

## 5. **Variable Management**
### **Instance Variable Files (`.tfvars`)**
Store non-sensitive configuration values in `.tfvars` files for each environment:
```hcl
# alpha-blue.tfvars
instance_type = "small"
region        = "us-central1"
replica_count = 2
```
```hcl
# prod-blue.tfvars
instance_type = "large"
region        = "us-central1"
replica_count = 5
```

### **Sensitive Variables**
Sensitive variables (e.g., passwords or API keys) are excluded from `.tfvars` files and managed entirely via GitHub Secrets.

---

## 6. **Provider Modules**
Use provider modules for reusable GCP resource definitions. For example:
```hcl
module "compute_instance" {
  source           = "terraform-google-modules/vm/google"
  version          = "~> 7.0"
  name             = var.instance_name
  machine_type     = var.instance_type
  zone             = var.zone
  network_interface = var.network_interface
}
```

Define these modules in `/terraform/modules`.

---

## 7. **Blue-Green Deployment Strategy**
Each environment (Alpha, Beta, Gamma, Prod) has a **Primary** (Blue) and **Secondary** (Green) deployment. This allows for seamless updates and rollbacks:
- Deploy updates to the Secondary environment.
- Run tests and validate changes.
- Promote the Secondary environment to Primary for cutover.

---

## 8. **Summary**
This approach provides a scalable, secure, and automated solution for managing Terraform deployments to GCP using GitHub Actions. By leveraging modular configurations, GitHub Secrets, and a shared GCP blob storage backend, this solution ensures flexibility across multiple environments and supports blue-green deployment strategies.