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

---

Each of these configurations securely mounts the secrets directly into the Kubernetes pods from the external secret managers, eliminating the need to store them in Kubernetes `Secret` objects. This setup provides a secure way to handle sensitive data with automatic refresh capabilities, role-based access control, and compliance with best practices.
