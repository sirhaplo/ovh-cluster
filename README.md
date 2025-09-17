# K3S Cluster hosted on OVH

Goal of this project is to have a HA K8S cluster based on bare metal on OVH.

**Steps and functionalities**
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

**Setup each node**  
```bash
## Update your machine
sudo apt update && sudo apt upgrade -y

## Change name to each machine
sudo echo node1 > /etc/hostname 

## Reboot
sudo reboot
```

**Install K3S on master node**  
```bash
# On master node install the first node
curl -sfL https://get.k3s.io

# Get the node token
cat /var/lib/rancher/k3s/server/node-token

# Get kube config
cat /etc/rancher/k3s/k3s.yaml
```

**Install K3S on master node**  
```bash
# On each node install K3S
curl -sfL https://get.k3s.io | K3S_URL=https://change_with_master_node:6443 K3S_TOKEN=change_with_token sh -
```

**Kube config**  
On your machine paste the kube config on `~/.kube/config`
Find your cluster and in server value change the url with the ip of your master node.

**Check on your machine**  
`kubectl get nodes` and you should see 3 nodes

## 3 - Argocd install
**Install ArgoCD with its manifest**  
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Install App of apps**  
Install ArgoCD application that will sync your cluster with your repository
```bash
kubectl apply -f bootstrap/app-of-apps.yaml
```

This will install the Infrastructure and Postgress Apps

## 4 Services setup

The setup consists in various service that will be preinstalled :

### Prometheus and Grafana
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

### Postgres operator
```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.27/releases/cnpg-1.27.0.yaml
```
It will be used in the next steps

### App of Apps
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

You must manually give access to your bucket creating a secret with service account json key :
```bash
kubectl create secret generic backup-creds -n database --from-file=gcsCredentials=service_account.json -o yaml --dry-run=client > gcs_crendentials.yaml
kubectl apply -f gcs_crendentials.yaml
```

#### Access instance

Proxy service port and access to localhost:5432
```bash
kubectl -n database port-forward svc/postgres-rw 5432:5432
```

#### Backup and restore
In `utils/scripts` there are 2 files to create a manual backup and restore

**Backup**
It will create a snaphot immediatly
```bash
kubectl apply -f backup-request.yaml
```

**Restore**
It will create a new cluster with the data from the backup. You can use the new cluster as primary cluster or copy the data you need
```bash
kubectl apply -f cluster-recovery.yaml
```

## 7 - Next steps
Next steps to do should be :

### MetalLB
It will orchestrate an internal load balancer using all external IP.  
So there will be a better reachability

### Sealed secrets / External secrets
The secrets saved on this repository are insecure.  
Sealed secret operator or external vaults will be a better solution
