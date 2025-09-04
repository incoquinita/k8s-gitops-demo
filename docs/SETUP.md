# Setup Detalhado

## Pré-requisitos

### Sistema
- Linux Mint / Ubuntu 20.04+
- 8GB RAM mínimo
- 20GB espaço livre
- Docker (opcional para build local)

### AWS
- Conta AWS com billing habilitado
- AWS CLI v2 configurado
- IAM User com permissões:
  - AmazonEKSClusterPolicy
  - AmazonEKSWorkerNodePolicy
  - AmazonEKS_CNI_Policy
  - AmazonEC2ContainerRegistryReadOnly
  - EC2 Full Access (para criação de VPC)

### Ferramentas
```bash
# Verificar versões
aws --version          # >= 2.0
kubectl version --client # >= 1.24
eksctl version         # >= 0.100
helm version          # >= 3.8
```

## Instalação Passo a Passo

### 1. Preparar Ambiente
```bash
# Atualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar dependências básicas
sudo apt install -y curl wget unzip git jq
```

### 2. Instalar AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### 3. Configurar AWS
```bash
aws configure
# AWS Access Key ID: SUA_ACCESS_KEY
# AWS Secret Access Key: SUA_SECRET_KEY
# Default region name: us-east-1
# Default output format: json

# Verificar configuração
aws sts get-caller-identity
```

### 4. Instalar kubectl
```bash
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubectl
```

### 5. Instalar eksctl
```bash
ARCH=$(uname -m)
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### 6. Instalar Helm
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update
sudo apt install -y helm
```

### 7. Criar Cluster EKS
```bash
# Criar cluster (15-20 minutos)
eksctl create cluster \
  --name=k8s-gitops-demo \
  --region=us-east-1 \
  --node-type=t3.medium \
  --nodes=2 \
  --nodes-min=1 \
  --nodes-max=4 \
  --with-oidc \
  --managed

# Configurar kubectl
aws eks update-kubeconfig --region us-east-1 --name k8s-gitops-demo

# Verificar
kubectl get nodes
```

### 8. Instalar ArgoCD
```bash
# Criar namespace
kubectl create namespace argocd

# Instalar ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Aguardar pods ficarem prontos
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Instalar CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

### 9. Configurar Acesso ArgoCD
```bash
# Obter senha inicial
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Acessar: https://localhost:8080
# Usuario: admin
# Senha: (obtida no comando acima)
```

### 10. Instalar NGINX Ingress
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Aguardar LoadBalancer
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=300s
```

## Verificação da Instalação

### Status do Cluster
```bash
# Verificar nodes
kubectl get nodes -o wide

# Verificar todos os pods
kubectl get pods -A

# Informações do cluster
kubectl cluster-info
```

### Status do ArgoCD
```bash
# Pods do ArgoCD
kubectl get pods -n argocd

# Service do ArgoCD
kubectl get svc -n argocd

# Logs (se necessário)
kubectl logs -n argocd deployment/argocd-server
```

### Status do Ingress
```bash
# Controller do NGINX
kubectl get pods -n ingress-nginx

# LoadBalancer
kubectl get svc ingress-nginx-controller -n ingress-nginx

# Obter IP externo
kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

## Configuração DNS Local

```bash
# Obter IP do LoadBalancer
LB_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Adicionar ao /etc/hosts
echo "$LB_IP sample-app.local" | sudo tee -a /etc/hosts

# Verificar
ping sample-app.local
```

## Deploy da Aplicação

### Via kubectl
```bash
# Aplicar manifests
kubectl apply -f k8s-manifests/

# Verificar deployment
kubectl get pods,svc,ingress
```

### Via ArgoCD
```bash
# Aplicar configuração do ArgoCD
kubectl apply -f argocd-apps/sample-app.yaml

# Verificar no UI ou CLI
argocd app list
argocd app get sample-webapp
```

## Acesso às Aplicações

### ArgoCD UI
- **URL**: https://localhost:8080 (via port-forward)
- **Usuario**: admin
- **Senha**: Obtida via `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

### Sample Application
- **URL**: https://sample-app.local
- **Nota**: Aceitar certificado self-signed no browser

## Troubleshooting

### Pods não inicializando
```bash
kubectl describe pod POD_NAME
kubectl logs POD_NAME
kubectl get events --sort-by=.metadata.creationTimestamp
```

### ArgoCD não acessível
```bash
# Verificar service
kubectl get svc argocd-server -n argocd

# Port-forward alternativo
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

### LoadBalancer pendente
```bash
# Verificar AWS Load Balancer Controller (se necessário)
kubectl get svc -A | grep LoadBalancer
```

### Certificados TLS
```bash
# Verificar secret
kubectl get secret sample-app-tls

# Recriar se necessário
kubectl delete secret sample-app-tls
# Executar novamente o script de criação de certificados
```
