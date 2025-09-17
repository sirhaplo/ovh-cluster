# K3S Cluster hosted on OVH

Goal of this project is to have a HA K8S cluster based on bare metal on OVH.

### Steps and functionalities
0. Clone the repo
1. Buy virtual machines on OVH and setup 
2. Nodes setup and K3S install
3. Argocd install
4. Services setup
5. Infrastructure
    4.1. Prometheus + Grafana    
    4.2. Longhorn
    4.3. Cert manager
    4.4. Argocd ingress
6. Postgresql
    5.1. PostgresPG
    5.2. Backup & Restore
7. Next steps

## 0 - Clone the repo
It will be needed by argocd

## 1 - Buy infrastructure
For this setup i bought 3 VM on OVH.  
They are affordable and great for testing.
Each VM has it's own IP.
- 4 CPU
- 8 Gb ram
- 75 Gb SSD
- 1 Daily backup
- No traffic limit
- Ubuntu 24.04

Buy also a domain that will be usefull to manage ingresses

## 2 - Nodes setup

### Setup each node**  
```bash
## Update your machine
sudo apt update && sudo apt upgrade -y

## Change name to each machine
sudo echo node1 > /etc/hostname 

## Reboot
sudo reboot
```

### Install K3s on the first (server) node
```bash
curl -sfL https://get.k3s.io | sh -
# Get the node token
sudo cat /var/lib/rancher/k3s/server/node-token
# Kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml
```

### Join agent nodes (replace with server address and token)
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
```

Copy the kubeconfig to your workstation (`~/.kube/config`) and update the server address to the master node IP. Verify nodes with `kubectl get nodes`.


## 3 - Argocd install
####Install ArgoCD with its manifest**  
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```


## 4 Services setup

### Prometheus & Grafana — monitoring
Use `kube-prometheus-stack` to get cluster and application monitoring. Grafana provides dashboards and Prometheus stores metrics.

Install  
```bash
helm install kube-prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

Use it 
```bash 
export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=kube-prometheus" -oname)
kubectl --namespace monitoring port-forward $POD_NAME 3000
# Go to http://localhost:3000 - user admin pass prom-operator
```

### Postgres operator — CloudNativePG
The operator manages PostgreSQL clusters and coordinates backups and restores. The repository includes example Postgres resources that create a small cluster and a sample database.

Install operator (example):
```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.27/releases/cnpg-1.27.0.yaml
```
It will be used in the next steps

### App of Apps
App-of-Apps pattern
- The `bootstrap/app-of-apps.yaml` is an Argo CD Application that watches the `apps/` folder and creates child Applications. This makes repository management modular.

Note: Some components (large CRDs for Grafana/Postgres) are often easier to install manually once, rather than via Argo CD, to avoid sync-size/time issues.

Argocd will install an Application that will monitor `apps` folder in repository.  
In this folder there will be other Application definition that will install manifests from other folders.

```bash
kubectl apply -f bootstrap/app-of-apps.yaml
```

Grafana and postgres are difficult to install by argocd because the size of their CRD definition, so its better to install them one time on cluster start

## 5 - Infrastructure

Is an ArgoCD Application installed by App of Apps.
It install automaticaly : 
- Longhorn
- Cert manager
- Cert issuer with Let's encrypt **Change your mail**
- Ingress for argocd **Change your domain**

You must manually :
- Change the email in cert issuer
- Change the ingress domain


## 6 - Postgres
The postgres Application will install a Postgres cluster ready for backups with cloudnativepg.

It create a 2 instance cluster on database namespace.  
The will be a `example` database for `example-user` with `SuperPassword!` as password.  
There is a preconfigured backup at midmight on google cloud platform storage bucket.

### Backup on google cloud platform

You must manually give access to your bucket creating a secret with service account json key :
```bash
kubectl create secret generic backup-creds -n database --from-file=gcsCredentials=service_account.json -o yaml --dry-run=client > gcs_crendentials.yaml
kubectl apply -f gcs_crendentials.yaml
```

### Access instance

Proxy service port and access to localhost:5432 to access locally
```bash
kubectl -n database port-forward svc/postgres-rw 5432:5432
```

### Manual backup and restore
- `utils/postgres/backup-request.yaml` triggers an immediate backup snapshot.
- `utils/postgres/cluster-recovery.yaml` demonstrates a restore flow that recreates a cluster from a backup.

## 7 — Next steps

MetalLB (recommended)
- MetalLB provides a simple external IP allocation / load balancer for bare VMs and improves reachability for services that need external IPs.

Secrets management
- Do not keep plaintext secrets in the repository. Use sealed-secrets, ExternalSecrets, HashiCorp Vault, or a cloud KMS-backed solution.

Security & production considerations (brief)
- Harden the VMs (disable unused services, enable a firewall, keep packages up to date).
- Limit who can access the Argo CD UI and API (use an ingress with authentication, or expose only through a bastion).
- Use network policies and resource limits for workloads.