![UJLV7eebMePrbY5EZuLoZ3](https://github.com/user-attachments/assets/0e6a6492-0e7c-43cb-8cb5-5b672907b974)

Configure the **Secrets Store CSI Driver** with Kubernetes to securely access secrets from AWS Secrets Manager, Azure Key Vault, or Google Cloud Secret Manager without storing secrets in Kubernetes.

---

## Step 1: Install Secrets Store CSI Driver

1. **Add the Helm Repository**:
   ```bash
   helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
   ```

2. **Install the Driver**:
   ```bash
   helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system
   ```
   This command installs the CSI driver in your Kubernetes cluster, allowing integration with external secret managers.

---

## Step 2: Configure Secrets with External Secret Managers

Choose the setup based on your cloud provider.

---

### **Option A: AWS Secrets Manager**

#### Step 2A.1: Configure IAM Role for Kubernetes Service Account

1. **Create an IAM Policy** that allows access to specific AWS Secrets:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "secretsmanager:GetSecretValue"
         ],
         "Resource": "arn:aws:secretsmanager:region:account-id:secret:your-secret-name*"
       }
     ]
   }
   ```

2. **Attach the Policy to an IAM Role** and **associate this role with your Kubernetes Service Account** that requires access to AWS Secrets.

   - If using EKS, you can associate the IAM role with a Kubernetes service account using IAM roles for service accounts (IRSA).

3. **Example YAML for IAM Role Binding**:
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: aws-secret-sa
     namespace: default
     annotations:
       eks.amazonaws.com/role-arn: arn:aws:iam::account-id:role/your-iam-role-name
   ```

#### Step 2A.2: Deploy `SecretProviderClass` for AWS Secrets Manager

1. **Create a `SecretProviderClass` resource** for AWS Secrets Manager:
   ```yaml
   apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
   kind: SecretProviderClass
   metadata:
     name: aws-secrets
   spec:
     provider: aws
     parameters:
       objects: |
         - objectName: "your-secret-name"
           objectType: "secretsmanager"
           objectAlias: "mySecret"
   ```

#### Step 2A.3: Use the Secret in Your Pod

1. **Mount the secrets as volumes in the pod definition**:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: app-using-aws-secret
   spec:
     serviceAccountName: aws-secret-sa
     containers:
     - name: app-container
       image: your-app-image
       volumeMounts:
       - name: secrets-store
         mountPath: "/mnt/secrets"
         readOnly: true
     volumes:
     - name: secrets-store
       csi:
         driver: secrets-store.csi.k8s.io
         readOnly: true
         volumeAttributes:
           secretProviderClass: "aws-secrets"
   ```

---

### **Option B: Azure Key Vault**

#### Step 2B.1: Set Up Azure Identity and Role Assignment

1. **Create a Managed Identity** for your Kubernetes cluster.
   
2. **Assign Access to Key Vault**:
   ```bash
   az keyvault set-policy -n <YourKeyVaultName> --secret-permissions get --spn <YourManagedIdentityClientID>
   ```

#### Step 2B.2: Deploy `SecretProviderClass` for Azure Key Vault

1. **Create a `SecretProviderClass` resource** for Azure Key Vault:
   ```yaml
   apiVersion: secrets-store.csi.x-k8s.io/v1
   kind: SecretProviderClass
   metadata:
     name: azure-keyvault-secrets
   spec:
     provider: azure
     parameters:
       usePodIdentity: "false"
       useVMManagedIdentity: "true" # Set to "false" if using Service Principal
       keyvaultName: "YourKeyVaultName"
       cloudName: "" # optional, for non-public regions
       objects: |
         array:
           - |
             objectName: "your-secret-name"
             objectType: secret
             objectVersion: ""
       tenantId: "<YourTenantID>"
   ```

#### Step 2B.3: Use the Secret in Your Pod

1. **Mount the secrets as volumes in the pod definition**:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: app-using-azure-secret
   spec:
     containers:
     - name: app-container
       image: your-app-image
       volumeMounts:
       - name: secrets-store
         mountPath: "/mnt/secrets"
         readOnly: true
     volumes:
     - name: secrets-store
       csi:
         driver: secrets-store.csi.k8s.io
         readOnly: true
         volumeAttributes:
           secretProviderClass: "azure-keyvault-secrets"
   ```

---

### **Option C: Google Cloud Secret Manager**

#### Step 2C.1: Enable Workload Identity on GKE

1. **Enable Workload Identity for your GKE cluster**:
   ```bash
   gcloud container clusters update your-cluster-name --workload-pool=your-project-id.svc.id.goog
   ```

#### Step 2C.2: Create a Google Service Account and Grant Access

1. **Create a Service Account with access to Google Secret Manager**:
   ```bash
   gcloud secrets add-iam-policy-binding your-secret-name \
       --role roles/secretmanager.secretAccessor \
       --member "serviceAccount:your-service-account@your-project-id.iam.gserviceaccount.com"
   ```

#### Step 2C.3: Deploy `SecretProviderClass` for Google Cloud Secret Manager

1. **Create a `SecretProviderClass` resource** for Google Cloud Secret Manager:
   ```yaml
   apiVersion: secrets-store.csi.x-k8s.io/v1
   kind: SecretProviderClass
   metadata:
     name: gcp-secrets
   spec:
     provider: gcp
     parameters:
       secrets: |
         - projects/your-project-id/secrets/your-secret-name
   ```

#### Step 2C.4: Use the Secret in Your Pod

1. **Mount the secrets as volumes in the pod definition**:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: app-using-gcp-secret
   spec:
     containers:
     - name: app-container
       image: your-app-image
       volumeMounts:
       - name: secrets-store
         mountPath: "/mnt/secrets"
         readOnly: true
     volumes:
     - name: secrets-store
       csi:
         driver: secrets-store.csi.k8s.io
         readOnly: true
         volumeAttributes:
           secretProviderClass: "gcp-secrets"
   ```

To automate the setup of the Secrets Store CSI Driver with AWS Secrets Manager for a Python application in Kubernetes, here’s a single Bash script you can use:

```bash
#!/bin/bash

# Step 1: Install the CSI Driver
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system

# Step 2: Configure AWS IAM Role
cat <<EoF > iam-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:region:account-id:secret:your-secret-name*"
    }
  ]
}
EoF
aws iam create-policy --policy-name SecretsAccessPolicy --policy-document file://iam-policy.json
aws iam create-role --role-name KubernetesSecretRole --assume-role-policy-document file://iam-policy.json

# Associate IAM Role with Kubernetes Service Account
cat <<EoF > sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-secret-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::account-id:role/KubernetesSecretRole
EoF
kubectl apply -f sa.yaml

# Step 3: Create SecretProviderClass
cat <<EoF > secret-provider.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "your-secret-name"
        objectType: "secretsmanager"
        objectAlias: "mySecret"
EoF
kubectl apply -f secret-provider.yaml

# Step 4: Create Pod with Secret Mount
cat <<EoF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: python-app
spec:
  serviceAccountName: aws-secret-sa
  containers:
    - name: python-container
      image: your-python-app-image
      volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "aws-secrets"
EoF
kubectl apply -f pod.yaml

echo "Setup complete. Pod 'python-app' is running with secrets from AWS Secrets Manager."
```

### Explanation
1. **Install CSI Driver** using Helm.
2. **Create IAM Policy and Role** for accessing AWS Secrets Manager and associate it with a Kubernetes service account.
3. **Define SecretProviderClass** to fetch secrets from AWS.
4. **Create a Pod** configured to mount the secret.

This script sets up the CSI Driver, configures IAM permissions, and deploys your Python application pod with the mounted secret. Adjust AWS account details, secret names, and container images as needed.

To automate the configuration for Azure Key Vault with Kubernetes Secrets Store CSI Driver in a script, here’s a single Bash script to set up everything, assuming you are using Azure Managed Identity.

### Script for Azure Key Vault Integration

```bash
#!/bin/bash

# Step 1: Install the CSI Driver
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system

# Step 2: Configure Azure Managed Identity and Access Policy

# Replace these with your actual values
KEYVAULT_NAME="your-keyvault-name"
TENANT_ID="your-tenant-id"
MANAGED_IDENTITY_NAME="your-managed-identity"
NAMESPACE="default"

# Create Managed Identity (if not already created)
az identity create --name $MANAGED_IDENTITY_NAME --resource-group your-resource-group

# Grant Key Vault access permissions to the managed identity
az keyvault set-policy -n $KEYVAULT_NAME --secret-permissions get --spn $(az identity show --name $MANAGED_IDENTITY_NAME --query clientId -o tsv)

# Step 3: Associate Managed Identity with Kubernetes Service Account

cat <<EoF > azure-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: azure-secret-sa
  namespace: $NAMESPACE
  annotations:
    azure.workload.identity/client-id: $(az identity show --name $MANAGED_IDENTITY_NAME --query clientId -o tsv)
EoF
kubectl apply -f azure-sa.yaml

# Step 4: Create SecretProviderClass for Azure Key Vault

cat <<EoF > secret-provider-azure.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault-secrets
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    keyvaultName: "$KEYVAULT_NAME"
    cloudName: "" # Optional for non-public regions
    objects: |
      array:
        - |
          objectName: "your-secret-name"
          objectType: secret
          objectVersion: ""
    tenantId: "$TENANT_ID"
EoF
kubectl apply -f secret-provider-azure.yaml

# Step 5: Create Pod with Secret Mount

cat <<EoF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: python-app
spec:
  serviceAccountName: azure-secret-sa
  containers:
    - name: python-container
      image: your-python-app-image
      volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-keyvault-secrets"
EoF
kubectl apply -f pod.yaml

echo "Setup complete. Pod 'python-app' is running with secrets from Azure Key Vault."
```

### Explanation

1. **Install CSI Driver** for Kubernetes.
2. **Set up Managed Identity** and assign permissions in Azure Key Vault.
3. **Associate Managed Identity** with a Kubernetes Service Account.
4. **Define SecretProviderClass** for Azure Key Vault.
5. **Deploy the Pod** to mount and use the secret securely.

Replace placeholder values with your Azure and Kubernetes-specific configurations.

Here’s a single Bash script for setting up the Secrets Store CSI Driver with Google Cloud Secret Manager on GCP. This script includes setting up Workload Identity, creating a service account, and deploying a Kubernetes pod to access secrets securely.

```bash
#!/bin/bash

# Step 1: Install the CSI Driver
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system

# Step 2: Enable Workload Identity on GKE
CLUSTER_NAME="your-cluster-name"
PROJECT_ID="your-project-id"
gcloud container clusters update $CLUSTER_NAME --workload-pool=$PROJECT_ID.svc.id.goog

# Step 3: Create Google Service Account and Assign Secret Manager Permissions

GSA_NAME="gcp-secret-accessor"
gcloud iam service-accounts create $GSA_NAME --project $PROJECT_ID

SECRET_NAME="your-secret-name"  # The GCP Secret Manager secret name
gcloud secrets add-iam-policy-binding $SECRET_NAME \
    --role roles/secretmanager.secretAccessor \
    --member "serviceAccount:$GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com"

# Step 4: Bind GSA to Kubernetes Service Account

NAMESPACE="default"
KSA_NAME="gcp-secret-sa"
kubectl create serviceaccount $KSA_NAME --namespace $NAMESPACE

gcloud iam service-accounts add-iam-policy-binding \
    $GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$KSA_NAME]"

kubectl annotate serviceaccount \
    $KSA_NAME \
    --namespace $NAMESPACE \
    iam.gke.io/gcp-service-account=$GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com

# Step 5: Create SecretProviderClass for GCP Secret Manager

cat <<EoF > secret-provider-gcp.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: gcp-secrets
spec:
  provider: gcp
  parameters:
    secrets: |
      - projects/$PROJECT_ID/secrets/$SECRET_NAME
EoF
kubectl apply -f secret-provider-gcp.yaml

# Step 6: Create Pod with Secret Mount

cat <<EoF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: python-app
spec:
  serviceAccountName: $KSA_NAME
  containers:
    - name: python-container
      image: your-python-app-image
      volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "gcp-secrets"
EoF
kubectl apply -f pod.yaml

echo "Setup complete. Pod 'python-app' is running with secrets from Google Cloud Secret Manager."
```

### Explanation

1. **Install CSI Driver** for Kubernetes.
2. **Enable Workload Identity** in GKE.
3. **Create GCP Service Account** and grant permissions to access GCP secrets.
4. **Bind the GCP Service Account** to a Kubernetes Service Account using Workload Identity.
5. **Define SecretProviderClass** to retrieve secrets from Google Cloud Secret Manager.
6. **Deploy the Pod** to mount and access the secret.

Replace placeholder values with your GCP and Kubernetes-specific configurations. This configuration allows secure access to secrets for applications running in GKE.
---

Each of these configurations securely mounts the secrets directly into the Kubernetes pods from the external secret managers, eliminating the need to store them in Kubernetes `Secret` objects. This setup provides a secure way to handle sensitive data with automatic refresh capabilities, role-based access control, and compliance with best practices.

To set up the Secrets Store CSI Driver with Google Cloud Secret Manager in a Kubernetes cluster using Terraform, here’s an example script that automates the necessary steps.

1. **Create Terraform configuration** for:
   - Enabling Workload Identity.
   - Setting up GCP IAM roles and policies.
   - Creating Kubernetes resources, including `ServiceAccount`, `SecretProviderClass`, and a `Pod` for testing.

### Terraform Script

```hcl
provider "google" {
  project = "your-project-id"
  region  = "your-region"
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

resource "google_container_cluster" "primary" {
  name               = "your-cluster-name"
  location           = "your-region"
  workload_identity_config {
    identity_namespace = "${var.project_id}.svc.id.goog"
  }
  node_config {
    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
}

# Step 1: Create a Google Service Account (GSA) for accessing secrets
resource "google_service_account" "gcp_secret_sa" {
  account_id   = "gcp-secret-accessor"
  display_name = "GCP Secret Accessor"
}

# Step 2: Grant Secret Access to the Service Account
resource "google_project_iam_binding" "secret_manager_access" {
  project = var.project_id
  role    = "roles/secretmanager.secretAccessor"
  members = ["serviceAccount:${google_service_account.gcp_secret_sa.email}"]
}

# Step 3: Map GSA to Kubernetes Service Account (KSA)
resource "google_service_account_iam_binding" "workload_identity_binding" {
  service_account_id = google_service_account.gcp_secret_sa.name
  role               = "roles/iam.workloadIdentityUser"
  members = [
    "serviceAccount:${var.project_id}.svc.id.goog[default/gcp-secret-sa]"
  ]
}

# Kubernetes Service Account (KSA)
resource "kubernetes_service_account" "ksa" {
  metadata {
    name      = "gcp-secret-sa"
    namespace = "default"
    annotations = {
      "iam.gke.io/gcp-service-account" = google_service_account.gcp_secret_sa.email
    }
  }
}

# Step 4: Create SecretProviderClass for GCP Secret Manager
resource "kubernetes_manifest" "secret_provider_class" {
  manifest = {
    "apiVersion" = "secrets-store.csi.x-k8s.io/v1"
    "kind"       = "SecretProviderClass"
    "metadata" = {
      "name" = "gcp-secrets"
    }
    "spec" = {
      "provider"   = "gcp"
      "parameters" = {
        "secrets" = <<EOF
- projects/${var.project_id}/secrets/your-secret-name
EOF
      }
    }
  }
}

# Step 5: Deploy a Pod with Secret Mounted
resource "kubernetes_pod" "python_app" {
  metadata {
    name      = "python-app"
    namespace = "default"
  }
  spec {
    service_account_name = kubernetes_service_account.ksa.metadata[0].name
    container {
      name  = "python-container"
      image = "your-python-app-image"
      volume_mount {
        name       = "secrets-store"
        mount_path = "/mnt/secrets"
        read_only  = true
      }
    }
    volume {
      name = "secrets-store"
      csi {
        driver = "secrets-store.csi.k8s.io"
        volume_attributes = {
          "secretProviderClass" = "gcp-secrets"
        }
      }
    }
  }
}

variable "project_id" {
  description = "Your Google Cloud Project ID"
}
```

### Explanation of Key Components

1. **Google Service Account (GSA)**: Creates a GSA and binds it to roles for accessing secrets.
2. **Kubernetes Service Account (KSA)**: Associates the KSA with GSA using Workload Identity.
3. **SecretProviderClass**: Configures GCP Secret Manager integration.
4. **Kubernetes Pod**: Deploys a test pod that mounts the secret from Secret Manager at `/mnt/secrets`.

Update values like `project_id`, `your-cluster-name`, and `your-secret-name` before running. This script manages both the GCP and Kubernetes configurations for a seamless setup.

Here’s a Terraform script for setting up the Secrets Store CSI Driver with AWS Secrets Manager in a Kubernetes cluster.

This script includes the following:

1. **AWS IAM Role and Policy** setup for accessing AWS Secrets Manager.
2. **Kubernetes Service Account** associated with the AWS IAM Role.
3. **SecretProviderClass** for Secrets Store CSI Driver to fetch secrets from AWS.
4. **Kubernetes Pod** to mount the secret.

### Terraform Script

```hcl
provider "aws" {
  region = "your-aws-region"
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

variable "namespace" {
  default = "default"
}

# Step 1: Create IAM Policy for Secrets Manager Access
resource "aws_iam_policy" "secrets_manager_policy" {
  name        = "SecretsManagerAccessPolicy"
  description = "Policy to allow access to AWS Secrets Manager secrets"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["secretsmanager:GetSecretValue"]
        Resource = "arn:aws:secretsmanager:your-aws-region:account-id:secret:your-secret-name*"
      }
    ]
  })
}

# Step 2: Create IAM Role and Attach Policy
resource "aws_iam_role" "secrets_manager_role" {
  name = "K8SSecretsManagerRole"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "attach_policy" {
  role       = aws_iam_role.secrets_manager_role.name
  policy_arn = aws_iam_policy.secrets_manager_policy.arn
}

# Step 3: Kubernetes Service Account (KSA) with IAM Role Association
resource "kubernetes_service_account" "ksa" {
  metadata {
    name      = "aws-secret-sa"
    namespace = var.namespace
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.secrets_manager_role.arn
    }
  }
}

# Step 4: SecretProviderClass for AWS Secrets Manager
resource "kubernetes_manifest" "secret_provider_class" {
  manifest = {
    "apiVersion" = "secrets-store.csi.x-k8s.io/v1alpha1"
    "kind"       = "SecretProviderClass"
    "metadata" = {
      "name" = "aws-secrets"
    }
    "spec" = {
      "provider" = "aws"
      "parameters" = {
        "objects" = <<EOF
- objectName: "your-secret-name"
  objectType: "secretsmanager"
  objectAlias: "mySecret"
EOF
      }
    }
  }
}

# Step 5: Deploy Pod with Secret Mounted
resource "kubernetes_pod" "python_app" {
  metadata {
    name      = "python-app"
    namespace = var.namespace
  }
  spec {
    service_account_name = kubernetes_service_account.ksa.metadata[0].name
    container {
      name  = "python-container"
      image = "your-python-app-image"
      volume_mount {
        name       = "secrets-store"
        mount_path = "/mnt/secrets"
        read_only  = true
      }
    }
    volume {
      name = "secrets-store"
      csi {
        driver = "secrets-store.csi.k8s.io"
        volume_attributes = {
          "secretProviderClass" = "aws-secrets"
        }
      }
    }
  }
}
```

### Explanation of Components

1. **IAM Policy and Role**: Configures a policy and IAM role for access to AWS Secrets Manager.
2. **Kubernetes Service Account**: Links the IAM role to the Kubernetes service account using annotations for IAM roles for service accounts (IRSA).
3. **SecretProviderClass**: Defines access to AWS Secrets Manager through CSI.
4. **Pod Definition**: Deploys a test pod that mounts the AWS secret.

### Variables to Update

Replace placeholders like `your-aws-region`, `account-id`, `your-secret-name`, and `your-python-app-image` with your specific AWS and Kubernetes configuration details.

Here’s a Terraform script to set up the Secrets Store CSI Driver with Azure Key Vault in Kubernetes. This includes creating a Managed Identity, assigning permissions in Key Vault, and linking the Kubernetes Service Account to the Managed Identity.

### Terraform Script for Azure Key Vault Integration

```hcl
provider "azurerm" {
  features {}
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}

variable "resource_group_name" {
  default = "your-resource-group"
}
variable "keyvault_name" {
  default = "your-keyvault-name"
}
variable "namespace" {
  default = "default"
}

# Step 1: Create Managed Identity
resource "azurerm_user_assigned_identity" "managed_identity" {
  resource_group_name = var.resource_group_name
  location            = "your-location"
  name                = "k8s-secret-accessor"
}

# Step 2: Set Access Policy in Azure Key Vault
resource "azurerm_key_vault_access_policy" "key_vault_access_policy" {
  key_vault_id = azurerm_key_vault.key_vault.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = azurerm_user_assigned_identity.managed_identity.principal_id

  secret_permissions = ["get"]
}

# Step 3: Kubernetes Service Account (KSA) with Managed Identity Annotation
resource "kubernetes_service_account" "ksa" {
  metadata {
    name      = "azure-secret-sa"
    namespace = var.namespace
    annotations = {
      "azure.workload.identity/client-id" = azurerm_user_assigned_identity.managed_identity.client_id
    }
  }
}

# Step 4: SecretProviderClass for Azure Key Vault
resource "kubernetes_manifest" "secret_provider_class" {
  manifest = {
    "apiVersion" = "secrets-store.csi.x-k8s.io/v1"
    "kind"       = "SecretProviderClass"
    "metadata" = {
      "name" = "azure-keyvault-secrets"
    }
    "spec" = {
      "provider" = "azure"
      "parameters" = {
        "usePodIdentity"       = "false"
        "useVMManagedIdentity" = "true"
        "keyvaultName"         = var.keyvault_name
        "objects" = <<EOF
array:
  - |
    objectName: "your-secret-name"
    objectType: secret
    objectVersion: ""
EOF
        "tenantId" = data.azurerm_client_config.current.tenant_id
      }
    }
  }
}

# Step 5: Deploy Pod with Secret Mounted
resource "kubernetes_pod" "python_app" {
  metadata {
    name      = "python-app"
    namespace = var.namespace
  }
  spec {
    service_account_name = kubernetes_service_account.ksa.metadata[0].name
    container {
      name  = "python-container"
      image = "your-python-app-image"
      volume_mount {
        name       = "secrets-store"
        mount_path = "/mnt/secrets"
        read_only  = true
      }
    }
    volume {
      name = "secrets-store"
      csi {
        driver = "secrets-store.csi.k8s.io"
        volume_attributes = {
          "secretProviderClass" = "azure-keyvault-secrets"
        }
      }
    }
  }
}

# Data source to get tenant ID
data "azurerm_client_config" "current" {}

# Key Vault for Access Policy
resource "azurerm_key_vault" "key_vault" {
  name                = var.keyvault_name
  location            = "your-location"
  resource_group_name = var.resource_group_name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"
}
```

### Explanation of Components

1. **Managed Identity**: Creates an Azure Managed Identity to access Azure Key Vault secrets.
2. **Key Vault Access Policy**: Grants the Managed Identity read access to secrets.
3. **Kubernetes Service Account**: Links the Kubernetes Service Account (KSA) to the Azure Managed Identity.
4. **SecretProviderClass**: Configures access to the Azure Key Vault.
5. **Pod Definition**: Deploys a test pod that mounts the Key Vault secret at `/mnt/secrets`.

### Variables to Update

Replace placeholders like `resource_group_name`, `keyvault_name`, `your-location`, and `your-secret-name` before running the script. This configuration securely integrates your Kubernetes application with Azure Key Vault.
