# AWX Deployment

This directory contains Ansible playbooks for deploying AWX (Ansible Web UI) on a Kubernetes cluster. The deployment is designed to run after Terraform has provisioned the necessary infrastructure.

## Files Overview

- **`main.yml`** - Main orchestration playbook that installs requirements and triggers AWX deployment
- **`deploy-awx.yml`** - Core deployment playbook that installs the AWX operator and AWX instance
- **`awx.yml`** - AWX Custom Resource (CR) definition with NodePort and PostgreSQL configuration
- **`postgres.yml`** - PostgreSQL deployment and service for AWX database
- **`postgres-secret.yml`** - Database credentials and connection configuration
- **`postgres-storage.yml`** - Persistent Volume and Persistent Volume Claim for database storage
- **`storage-class.yml`** - Local storage class for persistent volumes
- **`requirements.yml`** - Required Ansible collections for Kubernetes operations

## Prerequisites

Before running the AWX deployment, ensure you have:

1. **Kubernetes Cluster** - A running K3s or Kubernetes cluster
2. **kubectl Access** - kubectl configured to communicate with your cluster
3. **Ansible** - Ansible installed with Python kubernetes library
4. **Terraform State** - Terraform deployment completed (optional)
5. **Internet Access** - For downloading AWX operator and container images

**Note**: The `make` utility is optional. The playbook will automatically use `kubectl apply -k` if `make` is not available.

### Required Tools Installation

```bash
# Install Ansible (if not already installed)
pip install ansible

# Install Kubernetes Python library
pip install kubernetes

# Verify kubectl connectivity
kubectl get nodes
```

## Step-by-Step Deployment Process

### 1. Install Required Collections

The playbook will automatically install the required Ansible collections, but you can do this manually:

```bash
ansible-galaxy collection install -r requirements.yml --force
```

### 2. Run the Main Deployment

Execute the main playbook from the `awx` directory:

```bash
cd /path/to/awx_deployment/awx
ansible-playbook main.yml
```

### What Happens During Deployment

The deployment process consists of several automated steps:

1. **Collection Installation** - Installs `kubernetes.core` collection
2. **AWX Operator Clone** - Downloads AWX operator version 2.19.1 from GitHub
3. **Operator Deployment** - Deploys the AWX operator to the cluster using `make deploy` or `kubectl apply`
4. **Storage Setup** - Creates StorageClass and PostgreSQL persistent storage
5. **Database Deployment** - Deploys PostgreSQL with persistent storage and secrets
6. **Database Readiness** - Waits for PostgreSQL to be available
7. **AWX Instance Creation** - Applies the AWX Custom Resource configured to use PostgreSQL

### 3. Monitor Deployment Progress

After running the playbook, monitor the deployment:

```bash
# Check operator deployment
kubectl get pods -n awx-operator-system

# Check AWX namespace creation and pods
kubectl get namespaces
kubectl get pods -n awx

# Watch AWX deployment progress
kubectl get awx -n awx -w

# Check AWX pod logs
kubectl logs -n awx deployment/awx-task -f
```

### 4. Wait for AWX to be Ready

AWX deployment can take 5-15 minutes depending on your cluster resources and internet speed. The AWX instance is ready when:

- All pods in the `awx` namespace are running
- The AWX web service is accessible

```bash
# Check all resources in AWX namespace
kubectl get all -n awx

# Verify AWX status
kubectl get awx -n awx -o yaml
```

## Accessing AWX

### Web Interface

Once deployed, access AWX via:

```
http://YOUR_SERVER_IP:30081
```

Replace `YOUR_SERVER_IP` with your Kubernetes node's IP address.

### Getting Your Server IP

```bash
# Get node IP
kubectl get nodes -o wide

# Or if using a single-node cluster
hostname -I | awk '{print $1}'
```

### Default Login Credentials

- **Username**: `admin`
- **Password**: Retrieved from Kubernetes secret

Get the admin password:

```bash
kubectl get secret awx-admin-password -n awx -o jsonpath="{.data.password}" | base64 --decode && echo
```

## Configuration Details

### Configuration Details

**AWX Custom Resource (`awx.yml`):**
- **Service Type**: NodePort (for external access)
- **NodePort**: 30081 (fixed port for consistent access)
- **Database**: External PostgreSQL (managed locally)
- **Namespace**: awx

**PostgreSQL Configuration:**
- **Database**: PostgreSQL 13
- **Storage**: 8Gi persistent volume (local hostPath)
- **Data Location**: `/opt/awx-postgres-data` on the host
- **Credentials**: Stored in Kubernetes secret

### Customization Options

You can modify the deployment by editing `deploy-awx.yml`:

- **AWX Operator Version**: Change `repo_version` variable
- **Namespace**: Modify `ns` variable
- **Working Directory**: Adjust `workdir` path

## Troubleshooting

### Common Issues and Solutions

1. **Terraform State Not Found**
   ```bash
   # Check if Terraform state exists (optional step)
   ls -la ../terraform/terraform.tfstate
   # Note: This step is now optional and will be ignored if file doesn't exist
   ```

2. **kubectl Not Configured**
   ```bash
   # Configure kubectl for your cluster
   export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
   # or
   kubectl config view
   ```

3. **Make Not Available (Automatic Fallback)**
   ```bash
   # The playbook automatically detects when make is unavailable
   # and uses kubectl apply -k instead. No action needed.
   ```

4. **AWX Operator Deployment Failed**
   ```bash
   # Check operator logs
   kubectl logs -n awx-operator-system deployment/awx-operator-controller-manager
   
   # Try manual operator installation
   cd awx-operator
   kubectl apply -k config/default
   ```

5. **AWX Pods Not Starting**
   ```bash
   # Check pod status and events
   kubectl describe pods -n awx
   kubectl get events -n awx --sort-by='.lastTimestamp'
   ```

6. **NodePort Not Accessible**
   ```bash
   # Check service status
   kubectl get svc -n awx
   
   # Verify firewall rules (if applicable)
   sudo firewall-cmd --list-ports
   ```

### Cleanup

To remove AWX deployment:

```bash
# Delete AWX instance
kubectl delete awx awx -n awx

# Delete AWX operator
cd awx-operator
kubectl delete -k config/default

# Delete namespace
kubectl delete namespace awx
```

## Integration Notes

This AWX deployment is designed to work as part of a larger infrastructure automation pipeline:

1. **Terraform** provisions the infrastructure
2. **K3s deployment** sets up the Kubernetes cluster  
3. **AWX deployment** (this) installs the automation platform
4. **IPAM deployment** can be managed through AWX

The deployment waits for Terraform state file, ensuring proper sequencing in automated workflows. 