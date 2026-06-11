# Self-Healing DevOps Infrastructure

> Projet de stage EY — Infrastructure DevOps auto-réparatrice sur environnement de virtualisation imbriquée.

---

## Table des matières

- [Description du projet](#description-du-projet)
- [Architecture](#architecture)
- [Stack technique](#stack-technique)
- [Infrastructure mise en place](#infrastructure-mise-en-place)
- [Prérequis](#prérequis)
- [Installation et déploiement](#installation-et-déploiement)
- [Mécanismes de self-healing actifs](#mécanismes-de-self-healing-actifs)
- [Structure du projet](#structure-du-projet)
- [Prochaines étapes](#prochaines-étapes)

---

## Description du projet

Ce projet a pour objectif de concevoir et implémenter une infrastructure DevOps capable de **détecter automatiquement les pannes** et de **se réparer sans intervention humaine**.

Il est réalisé dans le cadre d'un stage d'été chez **EY (Ernst & Young)** sur un poste de travail personnel (32 Go RAM) en utilisant la technique de **virtualisation imbriquée** (nested virtualization).

### Objectifs

- Déployer un cluster Kubernetes multi-noeuds sur VMware ESXi
- Automatiser la configuration de l'infrastructure avec Ansible
- Mettre en place les mécanismes de self-healing natifs de Kubernetes
- Déployer une stack de monitoring complète (Prometheus + Grafana + Loki)
- Valider la résilience avec le chaos engineering (LitmusChaos)
- Mettre en place un pipeline CI/CD GitOps (GitHub Actions + ArgoCD)

---

## Architecture

```
PC Windows (32 Go RAM)
└── VMware Workstation Pro
    ├── VM Admin (1 Go RAM)          ← Ansible, kubectl
    └── VM ESXi Hypervisor (20 Go RAM)
        ├── VM k8s-master  (4 Go)    ← Control plane K8s
        ├── VM k8s-worker-1 (6 Go)   ← Worker node K8s
        └── VM k8s-worker-2 (6 Go)   ← Worker node K8s

Services externes :
└── GitHub                           ← Code source + CI/CD
    ├── GitHub Actions               ← Pipeline CI/CD
    └── Container Registry           ← Images Docker
```

### Flux des interactions

```
Developer → GitHub (git push)
GitHub Actions → build + test + push image
ArgoCD → sync manifestes → Kubernetes Cluster
Prometheus → scrape metrics → Alertmanager → Webhook → kubectl restart/rollback
Loki → collect logs → Grafana (dashboards)
```

---

## Stack technique

| Couche | Outil | Rôle |
|---|---|---|
| Virtualisation | VMware ESXi 8.0 | Hyperviseur bare-metal simulé |
| Poste admin | VMware Workstation Pro | Hôte de la nested virtualization |
| OS des VMs | Ubuntu Server 22.04 LTS | Système d'exploitation des noeuds |
| Configuration | Ansible | Provisionnement automatisé des VMs |
| Orchestration | Kubernetes 1.29 (kubeadm) | Orchestration des conteneurs |
| Réseau K8s | Calico CNI | Communication inter-pods |
| CI/CD | GitHub Actions + ArgoCD | Pipeline et déploiement GitOps |
| Monitoring métriques | Prometheus | Collecte des métriques |
| Dashboards | Grafana | Visualisation temps réel |
| Logs | Loki + Promtail | Centralisation des logs |
| Alerting | Alertmanager | Détection et actions correctives |
| Chaos Engineering | LitmusChaos | Validation de la résilience |

---

## Infrastructure mise en place

### Phase 1 — VMware ESXi ✅

- VMware Workstation Pro installé sur Windows
- VM ESXi 8.0 créée avec nested virtualization activée
  - CPU : 4 vCores
  - RAM : 20 Go
  - Réseau : NAT
- Option "Virtualize Intel VT-x/EPT" activée
- Interface web ESXi accessible sur `https://192.168.198.133`
- SSH activé sur ESXi
- Autostart configuré pour les VMs (redémarrage automatique après reboot ESXi)

### Phase 2 — Infrastructure as Code (Ansible) ✅

- VM Admin Ubuntu Server 22.04 créée dans VMware Workstation
- Ansible installé sur la VM Admin
- 3 VMs Ubuntu Server 22.04 créées manuellement dans ESXi :

| VM | IP | CPU | RAM | Rôle |
|---|---|---|---|---|
| k8s-master | 192.168.198.134 | 2 vCPU | 4 Go | Control plane |
| k8s-worker-1 | 192.168.198.135 | 2 vCPU | 6 Go | Worker node |
| k8s-worker-2 | 192.168.198.136 | 2 vCPU | 6 Go | Worker node |

- Playbooks Ansible écrits et exécutés :
  - `common.yml` : installation de containerd, kubeadm, kubelet, kubectl sur les 3 VMs
  - `k8s-master.yml` : initialisation du cluster, installation de Calico CNI
  - `k8s-workers.yml` : jonction des workers au cluster

### Phase 3 — Cluster Kubernetes ✅

- Cluster Kubernetes 1.29 opérationnel avec 3 noeuds
- Calico CNI installé (communication inter-pods)
- Hostnames configurés correctement (k8s-master, k8s-worker-1, k8s-worker-2)
- Self-healing natif actif :
  - Auto-restart des pods crashés
  - ReplicaSet maintien du nombre de replicas
  - Rescheduling des pods sur worker disponible

```bash
# Vérification du cluster
kubectl get nodes
# NAME           STATUS   ROLES           AGE   VERSION
# k8s-master     Ready    control-plane   Xm    v1.29.x
# k8s-worker-1   Ready    <none>          Xm    v1.29.x
# k8s-worker-2   Ready    <none>          Xm    v1.29.x
```

---

## Prérequis

### Matériel

- PC avec au minimum 32 Go RAM
- Processeur supportant la virtualisation imbriquée (Intel VT-x/EPT ou AMD-V)
- 200 Go de stockage disponible

### Logiciels

- VMware Workstation Pro 17+
- VMware ESXi 8.0 (ISO)
- Ubuntu Server 22.04 LTS (ISO)

### Comptes

- Compte GitHub (pour le CI/CD)

---

## Installation et déploiement

### 1. Préparer ESXi

```bash
# Activer SSH sur ESXi depuis l'interface web
# Host → Actions → Services → Enable Secure Shell (SSH)

# Vérifier la nested virtualization
ssh root@<IP-ESXI>
cat /proc/cpuinfo | grep vmx
```

### 2. Créer les VMs dans ESXi

Créer 3 VMs Ubuntu Server 22.04 dans l'interface web ESXi :

| VM | CPU | RAM | Disque |
|---|---|---|---|
| k8s-master | 2 | 4096 MB | 40 Go |
| k8s-worker-1 | 2 | 6144 MB | 40 Go |
| k8s-worker-2 | 2 | 6144 MB | 40 Go |

Après installation, changer les hostnames :

```bash
sudo hostnamectl set-hostname k8s-master    # sur la VM master
sudo hostnamectl set-hostname k8s-worker-1  # sur worker-1
sudo hostnamectl set-hostname k8s-worker-2  # sur worker-2
```

### 3. Configurer l'inventaire Ansible

Éditer `ansible/inventory.ini` avec les IPs de tes VMs :

```ini
[master]
k8s-master ansible_host=<IP-MASTER>

[workers]
k8s-worker-1 ansible_host=<IP-WORKER-1>
k8s-worker-2 ansible_host=<IP-WORKER-2>

[k8s:children]
master
workers

[all:vars]
ansible_user=omar
ansible_password=omar
ansible_become=true
ansible_become_method=sudo
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

### 4. Lancer les playbooks Ansible

```bash
cd ansible/

# Tester la connexion
ansible all -i inventory.ini -m ping

# Installer containerd + kubeadm sur les 3 VMs
ansible-playbook -i inventory.ini playbooks/common.yml -K

# Initialiser le cluster sur le master
ansible-playbook -i inventory.ini playbooks/k8s-master.yml -K

# Faire rejoindre les workers
ansible-playbook -i inventory.ini playbooks/k8s-workers.yml -K
```

### 5. Vérifier le cluster

```bash
ssh omar@<IP-MASTER>

# Vérifier les noeuds
kubectl get nodes

# Vérifier les pods système
kubectl get pods -n kube-system

# Tester le self-healing
kubectl create deployment nginx-test --image=nginx --replicas=2
kubectl get pods -w
kubectl delete pod <nom-pod>
kubectl get pods -w  # observer la recréation automatique
```

---

## Mécanismes de self-healing actifs

### Couche 1 — ESXi (niveau hyperviseur)

| Mécanisme | Description | Status |
|---|---|---|
| Autostart VMs | Redémarre les VMs automatiquement après reboot ESXi | ✅ Actif |
| vSphere HA | Migration vers un autre hôte (nécessite plusieurs ESXi) | ⬜ Non applicable |

### Couche 2 — Kubernetes (niveau conteneur)

| Mécanisme | Description | Status |
|---|---|---|
| Auto-restart pods | Kubernetes redémarre un pod crashé automatiquement | ✅ Actif |
| ReplicaSet | Maintient le nombre de replicas défini en permanence | ✅ Actif |
| Rescheduling | Migre les pods sur un worker disponible si un noeud tombe | ✅ Actif |
| Liveness Probe | Détecte si un pod est vivant → redémarre si échec | 🔜 Phase 3 suite |
| Readiness Probe | Retire le pod du load balancer s'il n'est pas prêt | 🔜 Phase 3 suite |
| HPA | Scale automatique selon la charge CPU | 🔜 Phase 3 suite |

### Couche 3 — ArgoCD (niveau déploiement)

| Mécanisme | Description | Status |
|---|---|---|
| Auto-sync GitOps | Synchronise le cluster avec Git automatiquement | 🔜 Phase 4 |
| Rollback automatique | Retour à la version stable si déploiement échoue | 🔜 Phase 4 |

### Couche 4 — Prometheus + Alertmanager (niveau monitoring)

| Mécanisme | Description | Status |
|---|---|---|
| Détection anomalies | Alerte si CPU > 85%, pod crashé, disk > 90% | 🔜 Phase 5 |
| Action corrective auto | Webhook déclenche kubectl restart/rollback | 🔜 Phase 5 |

---

## Structure du projet

```
selfhealing-devops-infra/
├── README.md                          ← Ce fichier
├── .gitignore
├── ansible/
│   ├── inventory.ini                  ← IPs et credentials des VMs
│   └── playbooks/
│       ├── common.yml                 ← Installation containerd + kubeadm
│       ├── k8s-master.yml             ← Initialisation du cluster
│       └── k8s-workers.yml            ← Jonction des workers
├── k8s/
│   ├── deployment.yaml                ← Déploiement app de test avec probes
│   └── hpa.yaml                       ← Horizontal Pod Autoscaler
└── .github/
    └── workflows/
        └── deploy.yml                 ← Pipeline GitHub Actions (à venir)
```

---

## Prochaines étapes

- [ ] **Phase 3 (suite)** : Liveness/Readiness Probes + HPA
- [ ] **Phase 4** : Pipeline CI/CD GitHub Actions + ArgoCD GitOps
- [ ] **Phase 5** : Stack monitoring Prometheus + Grafana + Loki + Alertmanager
- [ ] **Phase 6** : Chaos Engineering avec LitmusChaos
- [ ] **Phase 7** : Documentation finale + présentation jury

---

## Auteur

**Omar** — Stagiaire EY, étudiant ingénieur 4A Informatique  
Polytech Nantes / ESPRIT — Architecture IT et Cloud Computing  
Stage d'été 2025
