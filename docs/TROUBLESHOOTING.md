# Troubleshooting Guide

## Problemas Comuns

### 1. Cluster EKS não Criado

**Sintomas:**
- `eksctl create cluster` falha
- Erro de permissões AWS
- Timeout na criação

**Soluções:**
```bash
# Verificar credenciais AWS
aws sts get-caller-identity

# Verificar permissões IAM
aws iam list-attached-user-policies --user-name SEU_USUARIO

# Verificar quotas AWS
aws service-quotas get-service-quota --service-code eks --quota-code L-1194D53C

# Limpar recursos órfãos
aws cloudformation list-stacks --region us-east-1 | grep ROLLBACK
```

### 2. ArgoCD não Acessível

**Sintomas:**
- Port-forward falha
- UI não carrega
- Senha não funciona

**Soluções:**
```bash
# Verificar pods
kubectl get pods -n argocd
kubectl describe pod argocd-server-xxx -n argocd

# Resetar senha admin
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$rRyBsGSHK6.uc8fntPwVIuLVHgsAhAX7TcdrqW/RADU0lfHh.oom.",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'

# Port-forward alternativo
kubectl port-forward svc/argocd-server -n argocd 8080:80 --address 0.0.0.0
```

### 3. LoadBalancer Pendente

**Sintomas:**
- Service fica em estado "Pending"
- EXTERNAL-IP não é atribuído

**Soluções:**
```bash
# Verificar AWS Load Balancer Controller
kubectl get deployment -n kube-system aws-load-balancer-controller

# Instalar se não existir
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=k8s-gitops-demo

# Verificar service annotations
kubectl describe svc ingress-nginx-controller -n ingress-nginx
```

### 4. Certificados TLS Inválidos

**Sintomas:**
- Browser rejeita certificado
- ERR_CERT_AUTHORITY_INVALID

**Soluções:**
```bash
# Verificar certificado
kubectl get secret sample-app-tls -o yaml

# Recriar certificados
cd ~/k8s-certs
openssl verify -CAfile ca.crt server.crt

# Adicionar CA ao sistema (opcional para desenvolvimento)
sudo cp ca.crt /usr/local/share/ca-certificates/k8s-demo-ca.crt
sudo update-ca-certificates
```

### 5. DNS não Resolve

**Sintomas:**
- sample-app.local não resolve
- Connection refused

**Soluções:**
```bash
# Verificar /etc/hosts
cat /etc/hosts | grep sample-app

# Obter IP correto do LoadBalancer
kubectl get svc ingress-nginx-controller -n ingress-nginx -o wide

# Testar resolução
nslookup sample-app.local
dig sample-app.local

# Alternativa: usar IP direto
kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### 6. Pods CrashLoopBackOff

**Sintomas:**
- Pods reiniciam continuamente
- Status CrashLoopBackOff

**Soluções:**
```bash
# Verificar logs
kubectl logs POD_NAME --previous
kubectl describe pod POD_NAME

# Verificar recursos
kubectl top pods
kubectl describe node

# Verificar image pull
kubectl get events | grep Failed
```

### 7. ArgoCD Sync Falha

**Sintomas:**
- Application "OutOfSync"
- Sync errors no UI

**Soluções:**
```bash
# Verificar logs do ArgoCD
kubectl logs -n argocd deployment/argocd-application-controller

# Força resync
argocd app sync sample-webapp --force

# Verificar RBAC
kubectl auth can-i create deployments --as=system:serviceaccount:argocd:argocd-application-controller
```

### 8. Ingress não Funciona

**Sintomas:**
- 404 errors
- Backend service não encontrado

**Soluções:**
```bash
# Verificar ingress
kubectl describe ingress sample-webapp-ingress

# Verificar controller do NGINX
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Testar service diretamente
kubectl port-forward svc/sample-webapp-service 8080:80
```

### 9. Recursos Insuficientes

**Sintomas:**
- Pods em estado "Pending"
- FailedScheduling events

**Soluções:**
```bash
# Verificar recursos dos nodes
kubectl describe nodes
kubectl top nodes

# Escalar node group
eksctl scale nodegroup --cluster=k8s-gitops-demo --nodes=3 ng-xxx

# Verificar resource quotas
kubectl describe resourcequota
```

### 10. Network Issues

**Sintomas:**
- Pods não conseguem comunicar
- DNS resolution falha

**Soluções:**
```bash
# Verificar CNI
kubectl get pods -n kube-system | grep aws-node

# Testar conectividade
kubectl run test-pod --image=busybox --rm -it -- sh
# Dentro do pod: nslookup kubernetes.default

# Verificar security groups
aws ec2 describe-security-groups --filters "Name=group-name,Values=*eks*"
```

## Comandos de Debug

### Logs Centralizados
```bash
# ArgoCD
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=100

# NGINX Ingress
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=100

# Application
kubectl logs -l app=sample-webapp --tail=100
```

### Status Geral
```bash
# Cluster health
kubectl get componentstatuses

# Todos os recursos
kubectl get all -A

# Events recentes
kubectl get events --sort-by=.metadata.creationTimestamp | tail -20
```

### Performance
```bash
# Resource usage
kubectl top nodes
kubectl top pods -A

# Metrics server (se instalado)
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
```

## Limpeza e Reset

### Reset ArgoCD
```bash
# Deletar applications
kubectl delete applications -n argocd --all

# Reset ArgoCD
kubectl delete namespace argocd
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Reset Cluster
```bash
# Deletar recursos
kubectl delete -f k8s-manifests/
kubectl delete -f argocd-apps/

# Reset namespace
kubectl delete namespace default
kubectl create namespace default
```

### Remover Cluster Completo
```bash
# CUIDADO: Remove todo o ambiente
eksctl delete cluster --name=k8s-gitops-demo --region=us-east-1
```

## Contatos e Suporte

- **GitHub Issues**: https://github.com/SEU_USUARIO/k8s-gitops-demo/issues
- **Kubernetes Docs**: https://kubernetes.io/docs/
- **ArgoCD Docs**: https://argo-cd.readthedocs.io/
- **AWS EKS Docs**: https://docs.aws.amazon.com/eks/
