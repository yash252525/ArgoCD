# ArgoCD Setup on AWS EC2 with Kind Cluster

This guide documents the complete setup of ArgoCD for continuous deployment using Kind (Kubernetes in Docker) on an AWS EC2 instance.

## Prerequisites

- AWS EC2 instance (Ubuntu)
- Security Group with required ports open:
  - 22 (SSH)
  - 8443 (ArgoCD UI)
  - 5000 (Vote service - optional)
  - 5001 (Result service - optional)

## Installation Steps

### 1. Install Docker

```bash
sudo apt-get update
sudo apt-get install docker.io
systemctl start docker
systemctl enable docker
sudo usermod -aG docker $USER
systemctl restart docker
```

**Note:** Log out and back in for group changes to take effect.

### 2. Setup Project Structure

```bash
mkdir -p ArgoCD/installers
cd ArgoCD/installers/
```

### 3. Install Kind (Kubernetes in Docker)

Create `kind.sh`:
```bash
vim kind.sh
chmod +x kind.sh
./kind.sh
```

Verify installation:
```bash
kind --version
```

### 4. Create Kind Cluster with Custom Configuration

Create `config.yml` with your cluster configuration, then:

```bash
kind create cluster --config config.yml --name my-cluster
```

### 5. Install kubectl

Create `kubectl.sh`:
```bash
vim kubectl.sh
chmod +x kubectl.sh
./kubectl.sh
```

Verify installation:
```bash
kubectl get pods
```

### 6. Install ArgoCD

Create ArgoCD namespace:
```bash
kubectl create ns argocd
```

Apply ArgoCD manifests:
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check services:
```bash
kubectl get svc -n argocd
```

### 7. Expose ArgoCD Server

Patch service to NodePort (optional):
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

**Important:** Port-forward with external access (required for EC2):
```bash
kubectl port-forward -n argocd service/argocd-server 8443:443 --address=0.0.0.0 &
```

> **Note:** The `--address=0.0.0.0` flag is crucial for external access. Without it, the service only binds to localhost.

### 8. Get ArgoCD Admin Password

```bash
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

### 9. Access ArgoCD UI

Open browser:
```
https://<EC2-PUBLIC-IP>:8443
```

Login credentials:
- **Username:** `admin`
- **Password:** (from step 8)

## Deploy Applications via ArgoCD

### Create Application in ArgoCD UI

1. Click **"+ NEW APP"**
2. Fill in details:
   - **Application Name:** voting-app
   - **Project:** default
   - **Repository URL:** https://github.com/yash252525/Kubernetes
   - **Path:** VotingApp/deployments
   - **Cluster URL:** https://kubernetes.default.svc
   - **Namespace:** default
3. Click **"CREATE"**
4. Click **"SYNC"** to deploy

### Enable Auto-Sync (Optional)

In ArgoCD UI:
1. Click on your application
2. Click **"APP DETAILS"**
3. Click **"ENABLE AUTO-SYNC"**
4. Enable **"PRUNE RESOURCES"** and **"SELF HEAL"** as needed

## Port Forwarding for Application Services

### Vote Service (Frontend)
```bash
kubectl port-forward svc/vote 5000:5000 --address=0.0.0.0 &
```

### Result Service (Results Dashboard)
```bash
kubectl port-forward svc/result 5001:5001 --address=0.0.0.0 &
```

**Note:** Add ports 5000 and 5001 to your EC2 security group.

## Monitoring

Watch pods status:
```bash
kubectl get po -w
```

Check services:
```bash
kubectl get svc
```

Check ArgoCD applications:
```bash
kubectl get applications -n argocd
```

## Troubleshooting

### Port-Forward Not Accessible Externally

**Problem:** Port-forward only accessible from localhost.

**Solution:** Always use `--address=0.0.0.0`:
```bash
kubectl port-forward -n argocd service/argocd-server 8443:443 --address=0.0.0.0 &
```

### Kill Existing Port-Forward

```bash
pkill -f "port-forward.*argocd"
```

### NodePort Not Working on Kind

Kind clusters require special port mapping in `config.yml`. Port-forwarding is the recommended method for accessing services on EC2.

### Make Port-Forward Persistent

Create systemd service for automatic restart:
```bash
cat > /etc/systemd/system/argocd-port-forward.service <<EOF
[Unit]
Description=ArgoCD Port Forward
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/kubectl port-forward -n argocd svc/argocd-server 8443:443 --address=0.0.0.0
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable argocd-port-forward.service
systemctl start argocd-port-forward.service
```

## Useful Commands

```bash
# Create kubectl alias
echo 'alias k=kubectl' >> ~/.bashrc
source ~/.bashrc

# View ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server

# List all resources in namespace
kubectl get all -n argocd

# Delete ArgoCD (if needed)
kubectl delete ns argocd
```

## Security Best Practices

1. **Change default admin password** after first login
2. **Use Kubernetes Secrets** for sensitive data (database passwords, etc.)
3. **Restrict Security Group** rules to your IP instead of 0.0.0.0/0
4. **Enable RBAC** in ArgoCD for team access
5. **Use private repositories** with SSH keys or tokens

## Repository Structure

```
ArgoCD/
└── installers/
    ├── kind.sh          # Kind installation script
    ├── kubectl.sh       # kubectl installation script
    └── config.yml       # Kind cluster configuration
```

## References

- [ArgoCD Official Documentation](https://argo-cd.readthedocs.io/)
- [Kind Documentation](https://kind.sigs.k8s.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

## Author

**Yash Londhe**
- GitHub: [@yash252525](https://github.com/yash252525)

## License

MIT License
