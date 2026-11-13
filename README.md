# ansible-k8s-kubeadm-lab

Rôle Ansible qui déploie un cluster **Kubernetes kubeadm** (1 control-plane + N workers) sur **RHEL / AlmaLinux / Rocky 8-9**, avec **containerd** et **Calico** (opérateur Tigera). Le dépôt inclut un labo Vagrant/VirtualBox de 3 VM prêt à l'emploi.

Le rôle est idempotent : il peut être rejoué autant de fois que nécessaire, et il **répare** les états abîmés qu'on rencontre réellement en labo (paquets corrompus, kubelet masqué, cache d'images corrompu, firewalld à moitié démarré…).

## Architecture

| VM | IP | Rôle | Ressources |
|---|---|---|---|
| redhat-vm1 | 192.168.56.11 | control-plane | 2 vCPU / 2 Go |
| redhat-vm2 | 192.168.56.12 | worker | 2 vCPU / 2 Go |
| redhat-vm3 | 192.168.56.13 | worker | 2 vCPU / 2 Go |

- Kubernetes 1.28 (paramétrable), containerd 1.7 (cgroup systemd), Calico 3.28 en VXLAN
- Pod CIDR `10.244.0.0/16`, service CIDR `10.96.0.0/12`

## Prérequis (poste hôte)

- VirtualBox, Vagrant, Ansible ≥ 2.12
- 8 Go de RAM minimum (16 Go conseillés), virtualisation activée dans le BIOS

```bash
ansible-galaxy collection install -r requirements.yml
```

## Utilisation

```bash
vagrant up                      # crée les 3 VM
ansible all -m ping             # 3 × pong
ansible-playbook k8s_cluster.yml
```

Le run se termine par une vérification : tous les nœuds doivent être `Ready`. En cas d'échec, un diagnostic complet (nœuds, Calico, pods, logs des conteneurs en échec, événements) est affiché directement dans la sortie Ansible.

```bash
vagrant ssh redhat-vm1 -c "kubectl get nodes -o wide"
```

### Rejouer une seule étape

Chaque étape a son tag : `prereqs`, `packages`, `containerd`, `images`, `kubelet`, `firewall`, `reset`, `master`, `join`, `verify`.

```bash
ansible-playbook k8s_cluster.yml --tags containerd
ansible-playbook k8s_cluster.yml --tags master,join     # join a besoin de master
```

### Réinitialiser un worker abîmé

```bash
ansible-playbook k8s_cluster.yml -e k8s_reset_nodes=redhat-vm2
```

Le nœud est retiré du cluster, nettoyé (`kubeadm reset`, état CNI/Calico, iptables, interfaces) puis rejoint automatiquement. Le control-plane est protégé.

## Structure du rôle

```
roles/kubernetes_cluster/
├── defaults/main.yml      # toutes les variables (versions, réseau, firewall, images…)
├── handlers/main.yml
├── tasks/
│   ├── main.yml           # orchestration + tags
│   ├── preflight.yml      # inventaire, OS, RAM/CPU, IP du nœud
│   ├── prereqs.yml        # /etc/hosts, swap, SELinux, modules noyau, sysctl
│   ├── packages.yml       # dépôts, paquets, contrôle d'intégrité rpm -V
│   ├── containerd.yml     # config CRI + SystemdCgroup
│   ├── images.yml         # pré-pull avec reprises + réparation du cache containerd
│   ├── kubelet.yml        # node-ip, réparation d'un kubelet masqué
│   ├── firewall.yml       # mode labo (désactivé) ou mode durci
│   ├── reset.yml          # réinitialisation ciblée de workers
│   ├── master.yml         # kubeadm init, kube-proxy, Calico, commande de join
│   ├── worker.yml         # kubeadm join
│   └── verify.yml         # attente Ready + diagnostic automatique
└── templates/calico.yml.j2
```

## Variables principales

| Variable | Défaut | Rôle |
|---|---|---|
| `kubernetes_version` | `1.28.2` | Version Kubernetes (dépôt et cri-tools suivent) |
| `calico_version` | `3.28.2` | Version Calico |
| `pod_network_cidr` | `10.244.0.0/16` | CIDR des pods |
| `k8s_node_ip` | `ansible_host` | IP annoncée par kubelet (évite le NAT Vagrant) |
| `k8s_firewalld_enabled` | `false` | `true` = firewalld + règles K8s/Calico |
| `k8s_reset_nodes` | `[]` | Workers à réinitialiser puis faire rejoindre |
| `k8s_ready_timeout` | `600` | Attente max (s) des nœuds `Ready` |

## Problèmes résolus par le rôle

Retour d'expérience du labo, chaque cas est détecté et corrigé automatiquement :

| Symptôme | Cause | Traitement |
|---|---|---|
| `No package cri-tools available` | dépôt Kubernetes ajouté trop tard | dépôts configurés avant toute installation |
| `rc: 137` pendant dnf | VM sous-dimensionnée (OOM) | contrôle RAM/CPU en preflight |
| `Exec format error`, segfault de `crictl` | paquets corrompus (dnf interrompu) | `rpm -V` + réinstallation ciblée |
| `kubelet.service is masked` | masque persistant/runtime | démasquage + revérification |
| `firewalld` en timeout | démarrage raté sur petite VM | firewalld désactivé en mode labo |
| kube-proxy en `CrashLoopBackOff` | module `nf_conntrack` absent | module chargé et persisté |
| `unexpected commit digest` | couche d'image corrompue dans le cache containerd | purge du cache `ingest` + nouveau pull |
| Nœuds annoncés en `10.0.2.15` | interface NAT Vagrant | `--node-ip` et autodétection Calico `NodeInternalIP` |

## Licence

MIT
