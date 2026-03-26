# 🖥️ TP OpenShift Virtualization
## Architecture Hybride : VMs + Conteneur

> **Namespace :** `tp-openshift-virt`  
> **Plateforme :** Red Hat OpenShift Container Platform (Trial 60 jours)

---

## 📐 Architecture du Projet

```
                         INTERNET
                             │
                    ┌────────▼────────┐
                    │   VM1 - pfSense │  ← VirtualMachine OpenShift
                    │    (Firewall)   │
                    └────┬───────┬────┘
                         │       │
              ┌──────────▼──┐  ┌─▼──────────────┐
              │  RÉSEAU DMZ │  │  RÉSEAU LAN     │
              │192.168.100.0│  │ 192.168.10.0/24 │
              └──────┬──────┘  └───────┬─────────┘
                     │                 │
           ┌──────────▼──────┐  ┌──────▼──────────┐
           │  VM2 - Ubuntu   │  │  MySQL           │
           │  Nginx + Node   │  │  (CONTENEUR)     │
           │  VirtualMachine │  │  StatefulSet Pod │
           │ 192.168.100.10  │  │  mysql-service   │
           └─────────────────┘  └──────────────────┘
                  ↑ connexion DB ──────────────►
```

### Pourquoi cette architecture ?
| Composant | Type | Raison |
|-----------|------|--------|
| **VM1 pfSense** | VirtualMachine | pfSense est un OS complet (FreeBSD), impossible à conteneuriser |
| **VM2 Ubuntu Web** | VirtualMachine | Démontre OpenShift Virtualization avec cloud-init |
| **MySQL** | Conteneur (Pod) | Pratique réelle en entreprise, image officielle `mysql:8.0`, limite 2 VMs |

---

## 🗂️ Structure du Repo

```
tp-openshift-virt/
├── .github/
│   └── workflows/
│       └── deploy.yml                  ← Pipeline CI/CD GitHub Actions
├── network/
│   ├── nad-dmz.yaml                    ← Réseau DMZ (192.168.100.0/24)
│   ├── nad-lan.yaml                    ← Réseau LAN (192.168.10.0/24)
│   └── networkpolicy.yaml              ← Isolation MySQL (3 règles)
├── vms/
│   ├── vm1-pfsense/
│   │   └── virtualmachine.yaml         ← VM pfSense + DataVolumes
│   └── vm2-web/
│       └── virtualmachine.yaml         ← VM Ubuntu + cloud-init
├── conteneur/
│   └── mysql/
│       ├── secret.yaml                 ← Credentials MySQL (sécurisés)
│       ├── pvc.yaml                    ← Volume persistant 20Gi
│       ├── statefulset.yaml            ← Déploiement MySQL 8.0
│       └── service.yaml                ← Service interne (ClusterIP)
└── README.md
```

---

## 🖥️ Détail des Composants

### VM1 — Firewall pfSense (VirtualMachine)
| Paramètre | Valeur |
|-----------|--------|
| **Type** | OpenShift VirtualMachine (KubeVirt) |
| **OS** | pfSense CE 2.7.2 |
| **CPU** | 2 cores |
| **RAM** | 1 Gi |
| **Interface WAN** | Pod Network → Internet (NAT) |
| **Interface DMZ** | 192.168.100.1 |
| **Interface LAN** | 192.168.10.1 |

### VM2 — Serveur Web (VirtualMachine)
| Paramètre | Valeur |
|-----------|--------|
| **Type** | OpenShift VirtualMachine (KubeVirt) |
| **OS** | Ubuntu Server 22.04 LTS |
| **CPU** | 2 cores |
| **RAM** | 2 Gi |
| **Disque** | 20 Gi |
| **IP** | 192.168.100.10 (DMZ) |
| **Services** | Nginx (port 80) → Node.js (port 3000) |
| **Config** | cloud-init automatique |

### MySQL — Base de données (Conteneur)
| Paramètre | Valeur |
|-----------|--------|
| **Type** | OpenShift StatefulSet (Pod) |
| **Image** | `mysql:8.0` (officielle Docker Hub) |
| **RAM** | 512Mi (request) / 1Gi (limit) |
| **Stockage** | PVC 20Gi persistant |
| **Accès** | `mysql-service:3306` (ClusterIP interne) |
| **Base** | `appdb` |
| **Utilisateur** | `appuser` / `AppPass123!` |

---

## 🔐 Secrets GitHub à Configurer

Dans ton repo GitHub → **Settings** → **Secrets and variables** → **Actions** :

| Secret | Description | Commande pour obtenir |
|--------|-------------|----------------------|
| `OC_TOKEN` | Token d'authentification | `oc whoami -t` |
| `OC_SERVER` | URL de l'API du cluster | `oc whoami --show-server` |

---

## 🚀 Déploiement

### Étape 1 — Prérequis
```bash
# Vérifier la connexion au cluster
oc login --token=TON_TOKEN --server=URL_CLUSTER

# Vérifier que OpenShift Virtualization est installé
oc get hyperconverged -n openshift-cnv
```

### Étape 2 — Configurer les secrets GitHub
```bash
# Obtenir le token
oc whoami -t

# Obtenir l'URL du serveur
oc whoami --show-server
```

### Étape 3 — Pousser le code (déclenche le pipeline)
```bash
git add .
git commit -m "🚀 Deploy infrastructure hybride OpenShift"
git push origin main
```

### Étape 4 — Suivre le déploiement
Le pipeline GitHub Actions va dans l'ordre :
1. Se connecter au cluster OpenShift
2. Créer le namespace `tp-openshift-virt`
3. Déployer les réseaux (NAD DMZ + LAN + NetworkPolicies)
4. Déployer MySQL (Secret → PVC → StatefulSet → Service)
5. Déployer VM2 Ubuntu Web
6. Déployer VM1 pfSense
7. Afficher le résumé complet

---

## 🔧 Commandes de Vérification

```bash
# Sélectionner le namespace
oc project tp-openshift-virt

# Voir les VMs
oc get vm -n tp-openshift-virt

# Voir le pod MySQL
oc get pods -l app=mysql-db -n tp-openshift-virt

# Voir tous les composants
oc get vm,pods,svc,pvc,net-attach-def -n tp-openshift-virt

# Logs du conteneur MySQL
oc logs -l app=mysql-db -n tp-openshift-virt

# Tester MySQL depuis le cluster
oc exec -it $(oc get pod -l app=mysql-db -o name) -- \
  mysql -u appuser -pAppPass123! appdb -e "SHOW TABLES;"

# Accès console pfSense
virtctl console vm1-pfsense -n tp-openshift-virt

# Logs cloud-init VM2
virtctl ssh ubuntu@vm2-web -- sudo cat /var/log/cloud-init-output.log
```

---

## 🌐 Réseaux et Sécurité

### NetworkAttachmentDefinitions
| Nom | Subnet | Utilisé par |
|-----|--------|-------------|
| `dmz-network` | 192.168.100.0/24 | VM1 (pfSense), VM2 (Web) |
| `lan-network` | 192.168.10.0/24 | VM1 (pfSense), MySQL (via Service) |

### NetworkPolicies
| Règle | Effet |
|-------|-------|
| `allow-vm2-to-mysql` | MySQL accepte port 3306 uniquement depuis VM2 |
| `allow-vm2-internet-egress` | VM2 peut accéder à Internet |
| `deny-internet-to-mysql` | MySQL bloqué depuis Internet |

---

## 🐛 Dépannage Rapide

| Problème | Commande de diagnostic |
|----------|----------------------|
| VM bloquée Pending | `oc describe vm NOM_VM -n tp-openshift-virt` |
| Pod MySQL en erreur | `oc describe pod -l app=mysql-db -n tp-openshift-virt` |
| cloud-init non exécuté | `virtctl ssh ubuntu@vm2-web -- cloud-init status` |
| VM2 ne joint pas MySQL | `oc get networkpolicy -n tp-openshift-virt` |
| KubeVirt non installé | `oc get hyperconverged -n openshift-cnv` |
| DataVolume en erreur | `oc describe dv NOM_DV -n tp-openshift-virt` |

---

## 🔗 Ressources

| Ressource | URL |
|-----------|-----|
| Trial Red Hat OCP | [console.redhat.com/openshift](https://console.redhat.com/openshift) |
| Docs OpenShift Virt | [docs.openshift.com/.../virt](https://docs.openshift.com/container-platform/latest/virt/about-virt.html) |
| Image MySQL officielle | [hub.docker.com/_/mysql](https://hub.docker.com/_/mysql) |
| Ubuntu 22.04 cloud image | [cloud-images.ubuntu.com](https://cloud-images.ubuntu.com/jammy/current/) |
| Docs cloud-init | [cloudinit.readthedocs.io](https://cloudinit.readthedocs.io) |
| GitHub Actions OpenShift | [openshift-tools-installer](https://github.com/marketplace/actions/openshift-tools-installer) |

---

*TP réalisé dans le cadre du module OpenShift Virtualization — Architecture Hybride VMs + Conteneur*
