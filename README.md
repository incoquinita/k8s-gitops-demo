# 🚀 Kubernetes GitOps Demo

Este repositório contém uma demonstração completa de GitOps usando:

- **AWS EKS** - Cluster Kubernetes gerenciado
- **ArgoCD** - Continuous Deployment
- **NGINX Ingress** - Load balancing e TLS
- **Sample Application** - Aplicação web de demonstração

## 🏗️ Arquitetura

```
┌────────────────────────────────────────────┐
│                  GitHub                    │
│  ┌─────────────────────────────────────┐   │
│  │         k8s-manifests/              │   │
│  │  ├── deployment.yaml                │   │
│  │  ├── service.yaml                   │   │
│  │  ├── ingress.yaml                   │   │
│  │  └── kustomization.yaml             │   │
│  └─────────────────────────────────────┘   │
└────────────────────────────────────────────┘
                    │
                    │ Git Pull
                    ▼
┌────────────────────────────────────────────┐
│              AWS EKS Cluster               │
│  ┌─────────────────────────────────────┐   │
│  │              ArgoCD                 │   │
│  │  ┌─────────────────────────────┐    │   │
│  │  │      Sample Application     │    │   │
│  │  │                             │    │   │
│  │  │  ┌─────┐ ┌─────┐ ┌─────┐    │    │   │
│  │  │  │Pod 1│ │Pod 2│ │Pod N│    │    │   │
│  │  │  └─────┘ └─────┘ └─────┘    │    │   │
│  │  └─────────────────────────────┘    │   │
│  └─────────────────────────────────────┘   │
└────────────────────────────────────────────┘
```

## 🚀 Quick Start

1. **Clone e Setup**
```bash
git clone https://github.com/incoquinita/k8s-gitops-demo.git
cd k8s-gitops-demo
chmod +x scripts/setup.sh
./scripts/setup.sh --auto
```

2. **Acessar ArgoCD**
```bash
# Obter senha
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Abrir: https://localhost:8080
```

3. **Acessar Aplicação**
```bash
# Configurar DNS local
kubectl get svc ingress-nginx-controller -n ingress-nginx
echo "EXTERNAL-IP sample-app.local" | sudo tee -a /etc/hosts

# Abrir: https://sample-app.local
```

## 📁 Estrutura do Projeto

```
k8s-gitops-demo/
├── README.md
├── k8s-manifests/           # Manifests Kubernetes
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── kustomization.yaml
├── argocd-apps/            # Configurações ArgoCD
│   ├── sample-app.yaml
│   └── app-of-apps.yaml
├── helm-charts/            # Charts Helm
│   └── sample-app/
├── scripts/                # Scripts de automação
│   ├── setup.sh
│   └── create-repo.sh
└── docs/                   # Documentação
    ├── SETUP.md
    ├── TROUBLESHOOTING.md
    └── ARCHITECTURE.md
```

## 🛠️ Customização

Edite os arquivos em `k8s-manifests/` e faça commit. ArgoCD sincronizará automaticamente.

## 📋 Comandos Úteis

```bash
# Ver status do ArgoCD
argocd app list
argocd app get sample-webapp

# Sincronizar manualmente
argocd app sync sample-webapp

# Ver recursos no cluster
kubectl get all -n default
```

## 🔧 Troubleshooting

Consulte [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) para problemas comuns.

## 🤝 Contribuição

1. Fork o projeto
2. Crie sua feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit suas mudanças (`git commit -m 'Add some AmazingFeature'`)
4. Push para a branch (`git push origin feature/AmazingFeature`)
5. Abra um Pull Request

## 📄 Licença

Distribuído sob a licença MIT. Veja `LICENSE` para mais informações.
