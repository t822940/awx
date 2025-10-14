# AWX Deployment

Deploy AWX on Kubernetes after Terraform builds the VM.

## Usage

```bash
ansible-playbook main.yml
```

## Access AWX

Open http://YOUR_SERVER_IP:30081

Replace `YOUR_SERVER_IP` with your Kubernetes node IP address. 