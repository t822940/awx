# AWX Deployment

Deploy AWX on Kubernetes after Terraform builds the VM.

## Usage

```bash
ansible-playbook main.yml
```

## Access AWX

```bash
kubectl port-forward svc/awx-demo-service 8080:80 -n awx
```

Open http://localhost:8080