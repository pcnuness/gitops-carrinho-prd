# 🏗️ Arquitetura da Solução GitOps

## 📋 Visão Geral

Esta solução implementa uma plataforma completa de Internal Developer Platform (IDP) baseada em GitOps, integrando **Backstage**, **ArgoCD**, **GitLab CI**, **Helm** e **Kubernetes** para suportar **400+ desenvolvedores** e múltiplos microserviços.

## 🎯 Objetivos

- ✅ Self-service para desenvolvedores via Backstage
- ✅ Deployments padronizados e auditáveis
- ✅ Separação entre CI e CD
- ✅ Escalabilidade para centenas de microserviços
- ✅ Segurança e governança centralizadas
- ✅ Observabilidade integrada

---

## 🔄 Fluxo Completo de Deploy

```
┌─────────────────────────────────────────────────────────────────────┐
│                        FLUXO DE DEPLOYMENT                          │
└─────────────────────────────────────────────────────────────────────┘

1. DESENVOLVEDOR                                                        
   │                                                                    
   ├─► [Backstage Portal]                                              
   │   └─► Preenche formulário:                                        
   │       • Nome do microserviço                                      
   │       • Linguagem (Java/Node)                                     
   │       • Tag da imagem (ECR)                                       
   │       • Configurações                                             
   │                                                                    
   └─► Aciona Pipeline de Deploy                                       
       │                                                                
       ▼                                                                
                                                                        
2. GITLAB CI - PIPELINE DE MANIFESTS                                   
   │                                                                    
   ├─► Stage: VALIDATE                                                 
   │   └─► Valida parâmetros obrigatórios                             
   │   └─► Verifica existência do Helm Chart                          
   │                                                                    
   ├─► Stage: RENDER                                                   
   │   └─► Seleciona Helm Chart (Java ou Node)                        
   │   └─► Gera values-override.yaml com parâmetros                   
   │   └─► Renderiza manifestos K8s                                    
   │   └─► Valida manifestos (kubectl dry-run)                        
   │   └─► Organiza arquivos (01-namespace.yaml, etc.)                
   │                                                                    
   ├─► Stage: COMMIT                                                   
   │   └─► Clona repositório GitOps                                   
   │   └─► Cria branch: update-<service>-<tag>                        
   │   └─► Adiciona manifestos em applications/<service>/             
   │   └─► Commit e push                                              
   │   └─► [Opcional] Auto-merge para main                            
   │                                                                    
   └─► Stage: NOTIFY                                                   
       └─► Notifica Slack/Teams sobre sucesso/falha                   
       │                                                                
       ▼                                                                
                                                                        
3. REPOSITÓRIO GITOPS (GitHub/GitLab)                                  
   │                                                                    
   └─► gitops-carrinho-prd/                                           
       └─► applications/                                               
           ├─► payment-service/                                        
           │   ├─► 01-namespace.yaml                                   
           │   ├─► 02-serviceaccount.yaml                             
           │   ├─► 03-configmap.yaml                                  
           │   ├─► 04-deployment.yaml                                 
           │   ├─► 05-service.yaml                                    
           │   ├─► 06-ingress.yaml                                    
           │   └─► 07-hpa.yaml                                        
           │                                                            
           ├─► user-service/                                           
           └─► notification-service/                                   
       │                                                                
       ▼                                                                
                                                                        
4. ARGOCD - GitOps Engine                                              
   │                                                                    
   ├─► ApplicationSet Controller                                       
   │   └─► Monitora applications/*                                     
   │   └─► Auto-descobre novos microserviços                          
   │   └─► Cria Application para cada diretório                       
   │                                                                    
   ├─► Application Controller                                          
   │   └─► Sincroniza estado desejado (Git) vs atual (K8s)           
   │   └─► Aplica manifestos no cluster                              
   │   └─► Self-heal: corrige drifts                                 
   │   └─► Prune: remove recursos deletados                           
   │                                                                    
   └─► [Status disponível no Backstage]                               
       │                                                                
       ▼                                                                
                                                                        
5. KUBERNETES CLUSTER (EKS)                                            
   │                                                                    
   ├─► Namespace criado/atualizado                                    
   ├─► Deployment com nova versão                                     
   ├─► Service exposto                                                
   ├─► Ingress configurado (ALB)                                      
   ├─► HPA para autoscaling                                           
   └─► Health checks ativos                                           
       │                                                                
       ▼                                                                
                                                                        
6. MONITORAMENTO & OBSERVABILIDADE                                     
   │                                                                    
   ├─► Prometheus scrape metrics                                       
   ├─► Grafana dashboards                                             
   ├─► Logs centralizados                                             
   └─► Alertas configurados                                           
```

---

## 🏛️ Componentes da Arquitetura

### 1. Backstage (Portal do Desenvolvedor)

**Função:** Interface unificada para desenvolvedores

**Responsabilidades:**
- Catálogo de microserviços
- Software Templates para criação de workloads
- Visualização de status de deployments (ArgoCD)
- Seleção de versões de imagens (ECR tags)
- Documentação centralizada
- Kubernetes plugin para visualizar recursos

---

### 2. GitLab CI (Pipeline de Manifestos)

**Função:** Renderização e versionamento de manifestos

**Responsabilidades:**
- Validar parâmetros de entrada
- Renderizar Helm Charts em manifestos K8s
- Versionar manifestos no repositório GitOps
- Notificar sobre status de deploy

**Stages:**
1. **Validate:** Valida parâmetros e verifica Helm Chart
2. **Render:** Renderiza templates com valores específicos
3. **Commit:** Adiciona manifestos ao repo GitOps
4. **Notify:** Envia notificações (Slack/Teams)

**Trigger:**
```bash
# Acionado pelo Backstage ou pipeline de build
curl -X POST \
  --form token=$CI_TRIGGER_TOKEN \
  --form "variables[MICROSERVICE_NAME]=payment-service" \
  --form "variables[MICROSERVICE_LANGUAGE]=java" \
  --form "variables[IMAGE_TAG]=v1.2.3" \
  "https://gitlab.com/api/v4/projects/$PROJECT_ID/trigger/pipeline"
```

---

### 3. Helm Charts (Templates Padronizados)

**Função:** Padrões de infraestrutura como código

**Charts Disponíveis:**

#### 📦 microservice-java
- Spring Boot Actuator
- JVM otimizada (G1GC)
- Resources: 500m-1000m CPU, 512Mi-1Gi Memory
- Health checks: /actuator/health/liveness e /readiness

#### 📦 microservice-node
- Express.js padrão
- Node.js otimizado
- Resources: 250m-500m CPU, 256Mi-512Mi Memory
- Health checks: /health/liveness e /readiness

**Recursos Inclusos:**
- ✅ Namespace
- ✅ ServiceAccount (IRSA ready)
- ✅ ConfigMap
- ✅ Deployment (com health checks)
- ✅ Service (ClusterIP)
- ✅ Ingress (AWS ALB)
- ✅ HorizontalPodAutoscaler

---

### 4. Repositório GitOps

**Função:** Source of truth para estado desejado do cluster

**Estrutura:**
```
gitops-carrinho-prd/
├── bootstraps/
│   ├── gitops-root.yaml              # ArgoCD App of Apps
│   └── control-plane/
│       ├── addons/
│       │   └── oss/
│       │       ├── appset-crossplane.yaml
│       │       ├── appset-grafana.yaml
│       │       └── appset-metrics-server.yaml
│       ├── argocd-config/
│       │   └── app-projects.yaml
│       └── workload/
│           └── appset-applications.yaml   # ← ApplicationSet para apps
│
├── addons/
│   ├── crossplane/
│   ├── grafana/
│   └── metrics-server/
│
├── applications/                          # ← Manifestos dos microserviços
│   ├── payment-service/
│   │   ├── 01-namespace.yaml
│   │   ├── 02-serviceaccount.yaml
│   │   ├── 03-configmap.yaml
│   │   ├── 04-deployment.yaml
│   │   ├── 05-service.yaml
│   │   ├── 06-ingress.yaml
│   │   └── 07-hpa.yaml
│   ├── user-service/
│   └── notification-service/
│
└── iac/
    ├── crossplane/
    └── terraform/
```

**Principais Arquivos:**

**appset-applications.yaml:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: appset-applications
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/sua-empresa/gitops-carrinho-prd.git
        revision: main
        directories:
          - path: applications/*
  template:
    metadata:
      name: "app-{{path.basename}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/sua-empresa/gitops-carrinho-prd.git
        targetRevision: main
        path: "{{path}}"
      destination:
        server: https://kubernetes.default.svc
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

### 5. ArgoCD (GitOps Engine)

**Função:** Sincronização contínua Git → Kubernetes

**Componentes:**

#### ApplicationSet Controller
- **Auto-discovery:** Descobre automaticamente novos microserviços em `applications/*`
- **Multi-tenancy:** Cria Application isolada para cada microserviço
- **Git Generator:** Monitora diretórios no repositório GitOps

#### Application Controller
- **Sync:** Aplica manifestos do Git no cluster
- **Health Assessment:** Verifica saúde dos recursos
- **Self-Heal:** Reverte mudanças manuais no cluster
- **Prune:** Remove recursos deletados do Git

#### Sync Policy
```yaml
syncPolicy:
  automated:
    prune: true          # Remove recursos não presentes no Git
    selfHeal: true       # Reverte mudanças manuais
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
  syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
```

**Sync Waves:**
```yaml
# Ordem de aplicação usando annotations
annotations:
  argocd.argoproj.io/sync-wave: "-1"  # Namespace primeiro
  argocd.argoproj.io/sync-wave: "0"   # ServiceAccount, ConfigMap
  argocd.argoproj.io/sync-wave: "1"   # Deployment, Service
  argocd.argoproj.io/sync-wave: "2"   # Ingress, HPA
```

---

### 6. Kubernetes (EKS)

**Função:** Orquestração de containers

**Addons Essenciais:**
- **AWS ALB Ingress Controller:** Gerenciamento de ALB
- **Metrics Server:** Métricas para HPA
- **Crossplane:** Provisionamento de recursos AWS
- **Prometheus:** Coleta de métricas
- **Grafana:** Visualização de métricas

---

## 📈 Escalabilidade

### Suporte para Desenvolvedores usarem o self-services

**Estratégias:**

1. **Auto-discovery:** ApplicationSet descobre novos serviços automaticamente
2. **Namespaces isolados:** Cada microserviço em seu namespace
3. **Resource Quotas:** Limites por namespace
4. **HPA:** Autoscaling automático baseado em CPU/Memory
5. **Node Autoscaling:** Cluster Autoscaler para adicionar nodes


### Performance

- **Renderização paralela:** GitLab CI runners em paralelo
- **ArgoCD Sharding:** Múltiplos controllers para distribuir carga
- **Git shallow clone:** Clones rápidos do repositório GitOps
- **Manifest caching:** Cache de manifestos renderizados

---

## 🔄 CI/CD Separados

### CI Pipeline (Build)

**Responsabilidades:**
- Build da aplicação
- Testes unitários e integração
- Build da imagem Docker
- Push para ECR
- Scan de vulnerabilidades
- **Trigger da CD Pipeline**

```yaml
# .gitlab-ci.yml no repositório do microserviço
stages:
  - build
  - test
  - push
  - trigger-cd

build:
  stage: build
  script:
    - mvn clean package

test:
  stage: test
  script:
    - mvn test

push:ecr:
  stage: push
  script:
    - docker build -t $ECR_REGISTRY/payment-service:$CI_COMMIT_TAG .
    - docker push $ECR_REGISTRY/payment-service:$CI_COMMIT_TAG

trigger:cd:
  stage: trigger-cd
  script:
    - |
      curl -X POST \
        --form token=$GITOPS_TRIGGER_TOKEN \
        --form "variables[MICROSERVICE_NAME]=payment-service" \
        --form "variables[MICROSERVICE_LANGUAGE]=java" \
        --form "variables[IMAGE_TAG]=$CI_COMMIT_TAG" \
        "https://gitlab.com/api/v4/projects/$GITOPS_PROJECT_ID/trigger/pipeline"
```

### CD Pipeline (Deploy)

**Responsabilidades:**
- Renderizar manifestos Helm
- Validar manifestos
- Commitar no repositório GitOps
- ArgoCD aplica automaticamente

---

## 🎨 Experiência do Desenvolvedor (Backstage)

### Software Template para Deploy

**Formulário no Backstage:**
```yaml
parameters:
  - title: Microserviço
    properties:
      name:
        title: Nome
        type: string
        description: Nome do microserviço
      
      language:
        title: Linguagem
        type: string
        enum:
          - java
          - node
      
      imageTag:
        title: Versão da Imagem
        type: string
        description: Tag da imagem no ECR
        ui:field: EcrTagPicker  # Custom field

      environment:
        title: Ambiente
        type: string
        enum:
          - development
          - staging
          - production
```

### Visualização de Status

**Plugins do Backstage:**
- **ArgoCD Plugin:** Status de sync, health
- **Kubernetes Plugin:** Pods, logs em tempo real
- **ECR Plugin:** Listar tags disponíveis
- **Grafana Plugin:** Dashboards embarcados

---

## 🚀 Fluxo de Trabalho do Desenvolvedor

```
1. Desenvolvedor acessa Backstage
   │
   ├─► Navega até "Deploy Microservice"
   │
2. Preenche formulário
   │
   ├─► Nome: payment-service
   ├─► Linguagem: java
   ├─► Tag: v1.2.3
   ├─► Ambiente: production
   │
3. Clica em "Deploy"
   │
   ├─► Backstage aciona GitLab CI Pipeline
   │
4. Pipeline renderiza e commita manifestos
   │
   ├─► ArgoCD detecta mudanças
   ├─► ArgoCD sincroniza cluster
   │
5. Desenvolvedor visualiza status no Backstage
   │
   ├─► ArgoCD Plugin: Sync Status
   ├─► Kubernetes Plugin: Pods rodando
   ├─► Grafana Plugin: Métricas em tempo real
   │
6. Deploy concluído! ✅
```

---

## 💡 Benefícios da Arquitetura

### Para Desenvolvedores
- ✅ Self-service: Deploy sem depender de Ops
- ✅ Padronização: Helm Charts testados
- ✅ Visibilidade: Status em tempo real no Backstage
- ✅ Rollback fácil: Git revert + ArgoCD sync

### Para Platform Team
- ✅ Governança: Todos os deploys auditados no Git
- ✅ Escalabilidade: Suporta centenas de microserviços
- ✅ Segurança: RBAC, IRSA, Network Policies
- ✅ Observabilidade: Métricas e logs centralizados

### Para a Empresa
- ✅ Redução de custos: Automação reduz necessidade de Ops
- ✅ Time-to-market: Deploys em minutos, não horas
- ✅ Confiabilidade: GitOps garante estado consistente
- ✅ Compliance: Auditoria completa via Git history

---

## 🔮 Roadmap Futuro

### Fase 1 (Atual)
- ✅ Helm Charts para Java e Node
- ✅ Pipeline GitLab CI
- ✅ ArgoCD ApplicationSet
- ✅ Backstage básico

### Fase 2 (Próximos 3 meses)
- 🔄 Helm Charts para Python, Go, .NET
- 🔄 Progressive Delivery (Canary/Blue-Green)
- 🔄 Policy enforcement (OPA/Kyverno)
- 🔄 Cost tracking por microserviço

### Fase 3 (6 meses)
- 🔄 Multi-cluster support
- 🔄 Disaster Recovery automático
- 🔄 Service Mesh (Istio)
- 🔄 Chaos Engineering

---

## 📚 Referências

- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Backstage Architecture](https://backstage.io/docs/overview/architecture-overview)
- [GitOps Principles](https://opengitops.dev/)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)

---

## 📞 Contato

**Platform Team**
- Slack: #platform-team
- Email: platform@company.com
- Confluence: [Internal Developer Platform](https://wiki.company.com/idp)

