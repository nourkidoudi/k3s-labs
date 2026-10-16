# Kubernetes avec k3s — Comptes rendus des labs

**Étudiante :** Nour Kidoudi
**Formation :** Master DevOps & Cloud Computing — ISET Tozeur
**Enseignant :** Dr-Ing Haithem Hafsi
**Environnement :** 3 VMs Ubuntu Server sous VMware (1 serveur + 2 agents), k3s v1.36.5+k3s1

## Sommaire

- [Partie 1 – Fondations](#partie-1--fondations)
  - [Lab 1.1 – Installer k3s et explorer le cluster](#lab-11--installer-k3s-et-explorer-le-cluster)
  - [Lab 1.2 – Prise en main de kubectl](#lab-12--prise-en-main-de-kubectl)

> L'alias `k` (pour `kubectl`) est utilisé dans tous les labs à partir du Lab 1.2.

---

# Partie 1 – Fondations

## Lab 1.1 – Installer k3s et explorer le cluster

### Objectifs

- Installer un cluster k3s à 1 serveur et 2 agents
- Vérifier l'état des nœuds
- Repérer les composants du cluster dans le namespace `kube-system`

### Plan du cluster

| Machine | Rôle | Adresse IP |
|---|---|---|
| `k3s-server` | Serveur (control plane) | 192.168.237.131 |
| `k3s-agent1` | Agent | 192.168.237.132 |
| `k3s-agent2` | Agent | 192.168.237.133 |

> **Remarque sur les noms.** Le cluster était déjà en place avant ce lab. Les nœuds sont donc enregistrés dans Kubernetes sous leurs anciens noms (`k3s-master`, `k3s-worker1`, `k3s-worker2`), alors que les noms d'hôte ont été changés en `k3s-server`, `k3s-agent1` et `k3s-agent2` comme demandé dans l'énoncé. Dans les captures, les deux noms apparaissent donc selon la commande (`hostname` ou `kubectl get nodes`). Le jeton d'adhésion a été masqué dans les captures.

### Étape 1 : Préparer les trois machines

Nom d'hôte unique sur chaque VM, puis relevé de l'adresse IP :

```bash
sudo hostnamectl set-hostname k3s-server     # k3s-agent1 et k3s-agent2 sur les deux autres
hostname
ip -4 addr show | grep inet
sudo ufw disable
```

**Serveur** (`k3s-server`, IP 192.168.237.131) :

![hostname et IP du serveur](images/lab-1-1/01-hostname-ip-serveur.png)

Les captures des agents (étape 4) montrent aussi `hostname`, l'adresse IP et la désactivation du pare-feu :
`k3s-agent1` → 192.168.237.132 et `k3s-agent2` → 192.168.237.133.

### Étape 2 : Installer le serveur

```bash
curl -sfL https://get.k3s.io | sh -
sudo systemctl status k3s --no-pager
sudo k3s kubectl get nodes
```

![statut du service k3s](images/lab-1-1/02-status-k3s-serveur.png)

**Point de contrôle**

- [x] Le service `k3s` est `active (running)`
- [x] Un nœud serveur apparaît à l'état `Ready` (voir étape 5)

### Étape 3 : Récupérer le jeton d'adhésion

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Le jeton est un secret : il n'est volontairement pas affiché dans ce compte rendu.

### Étape 4 : Joindre les deux agents

Sur chaque agent (`<IP_SERVEUR>` = 192.168.237.131, `<JETON>` masqué) :

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<IP_SERVEUR>:6443 K3S_TOKEN=<JETON> sh -
sudo systemctl status k3s-agent --no-pager
```

**k3s-agent1**

![jonction de l'agent 1](images/lab-1-1/03-agent1-jonction.png)

**k3s-agent2**

![jonction de l'agent 2](images/lab-1-1/04-agent2-jonction.png)

Dans les deux cas, le service `k3s-agent` est `active (running)`.

### Étape 5 : Vérifier le cluster

```bash
sudo k3s kubectl get nodes -o wide
```

![kubectl get nodes -o wide](images/lab-1-1/05-get-nodes-wide.png)

**Point de contrôle**

- [x] Trois nœuds sont à l'état `Ready`
- [x] Les rôles et les adresses IP correspondent au plan : un nœud `control-plane` (192.168.237.131) et deux nœuds sans rôle défini, donc agents (192.168.237.132 et 192.168.237.133)

### Étape 6 : Utiliser kubectl sans sudo

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
kubectl get nodes
```

La capture de l'étape 7 (`kubectl get pods -A -o wide`) est exécutée avec `kubectl` seul, sans `sudo`, ce qui confirme que le kubeconfig est bien en place.

### Étape 7 : Explorer le cluster

```bash
kubectl cluster-info
kubectl get namespaces
kubectl get pods -A -o wide
kubectl get all -n kube-system
sudo ls /var/lib/rancher/k3s/server/manifests/
```

**Tous les Pods du cluster**

![kubectl get pods -A -o wide](images/lab-1-1/06-get-pods-all.png)

Composants repérés dans `kube-system` :

| Composant | Pod | Nœud |
|---|---|---|
| CoreDNS | `coredns-7cfb7bc9c7-n5s8s` | k3s-master (serveur) |
| Traefik | `traefik-7c8544f77-kchlz` | k3s-master (serveur) |
| ServiceLB pour Traefik | `svclb-traefik-...` (3 Pods, un par nœud) | serveur, worker1, worker2 |
| local-path-provisioner | `local-path-provisioner-77b9867795-w6nmx` | k3s-master (serveur) |
| metrics-server | `metrics-server-6dc596dfb8-5cnp4` | k3s-master (serveur) |
| Jobs d'installation de Traefik | `helm-install-traefik-crd-...` et `helm-install-traefik-...` | `Completed` |

**Manifestes déployés automatiquement par k3s**

![contenu du dossier manifests](images/lab-1-1/07-manifests-k3s.png)

Le dossier contient `ccm.yaml`, `coredns.yaml`, `local-storage.yaml`, `metrics-server`, `rolebindings.yaml`, `runtimes.yaml` et `traefik.yaml`. k3s applique automatiquement tout manifeste placé dans ce dossier.

**Point de contrôle**

- [x] Les Pods de `kube-system` sont `Running` ou `Completed` (les deux Jobs Helm de Traefik sont `Completed`)
- [x] Le Pod Traefik est identifié ; il est exposé par un Service de type `LoadBalancer`, relayé par les Pods `svclb-traefik-*`

> **Remarque :** `metrics-server` est `Running` mais affiche `0/1` dans la colonne `READY` : le Pod tourne, mais il n'est pas déclaré prêt. Cela peut empêcher `kubectl top` de renvoyer des métriques.

Les autres Pods visibles (`default`, `lab2`, `tasks`) viennent d'exercices précédents sur le même cluster.

### Questions de réflexion

**1. Quel composant joue le rôle de Ingress controller dans k3s, et dans quel namespace est-il ?**

C'est **Traefik**, installé par défaut avec k3s dans le namespace **`kube-system`** (Pod `traefik-...`). Il est déployé à l'aide d'un chart Helm (Jobs `helm-install-traefik*`) à partir du manifeste `traefik.yaml` du dossier `manifests`.

**2. Pourquoi le jeton d'adhésion est-il sensible ?**

Toute machine qui connaît le jeton et peut joindre l'API sur le port 6443 peut **rejoindre le cluster comme nœud**. Elle peut alors recevoir des Pods, et donc accéder aux Secrets et aux données de ces Pods. Le jeton ne doit donc jamais être publié (dépôt Git, capture d'écran, etc.).

**3. Où se trouve le datastore du cluster dans cette installation à un seul serveur ?**

Avec un seul serveur, k3s utilise par défaut une base **SQLite** (via l'adaptateur Kine) stockée sur le serveur, dans `/var/lib/rancher/k3s/server/db/`. Il n'y a pas d'etcd dans cette configuration.

---

## Lab 1.2 – Prise en main de kubectl

### Objectifs

- Gagner en vitesse avec l'alias et l'autocomplétion
- Travailler dans un namespace dédié
- Explorer l'API avec `api-resources` et `explain`
- Générer, appliquer et inspecter un manifeste

### Étape 1 : Alias et autocomplétion

```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
k version
```

L'alias `k` est utilisé dans toutes les captures suivantes (`k get pod web -o wide`, `k config view ...`).

### Étape 2 : Travailler dans votre namespace

```bash
k create namespace tp-k8s
k config get-contexts
k config set-context --current --namespace=tp-k8s
k config view --minify | grep namespace
```

### Étape 3 : Explorer l'API

```bash
k api-resources | head -20
k explain pod
k explain pod.spec.containers
k explain deployment.spec.strategy
```

![namespace courant et api-resources](images/lab-1-2/01-namespace-api-resources.png)

La première commande confirme que le namespace courant est `tp-k8s`. `k api-resources` liste les types de ressources de l'API (nom, abréviation, version, namespacée ou non, kind), par exemple `pods` / `po` / `v1` / `Pod` ou `services` / `svc`.

### Étape 4 : Générer un manifeste sans l'écrire à la main

```bash
k run web --image=nginx:alpine --dry-run=client -o yaml > web.yaml
cat web.yaml
k apply -f web.yaml
k get pod web -o wide
k get pod web -o yaml | head -40
```

![k get pod web -o wide](images/lab-1-2/02-get-pod-web-wide.png)

**Point de contrôle**

- [x] Le Pod `web` est `Running` (`1/1`, 0 redémarrage)
- [x] Il s'exécute sur le nœud **k3s-worker1**, avec l'IP de Pod `10.42.1.37`

### Étape 5 : Inspecter un Pod

```bash
k describe pod web
k logs web
k exec -it web -- sh
# dans le conteneur : hostname, wget -qO- localhost, exit
```

Section `Events` de `k describe pod web` :

![events du Pod web](images/lab-1-2/03-describe-events.png)

Les événements racontent le démarrage du Pod, dans l'ordre :

1. `Scheduled` : le scheduler a assigné `tp-k8s/web` au nœud `k3s-worker1`
2. `Pulled` : l'image `nginx:alpine` était déjà présente sur la machine
3. `Created` : le conteneur a été créé
4. `Started` : le conteneur a démarré

### Étape 6 : Extraire des informations précises

```bash
k get pod web -o jsonpath='{.status.podIP}'; echo
k get pods -o custom-columns=NOM:.metadata.name,NOEUD:.spec.nodeName,IP:.status.podIP
```

![custom-columns](images/lab-1-2/04-custom-columns.png)

`custom-columns` affiche uniquement le nom, le nœud et l'IP de chaque Pod du namespace. On retrouve `web` sur `k3s-worker1` avec l'IP `10.42.1.37`, comme à l'étape 4.

### Étape 7 : Modifier puis recréer

```bash
k label pod web tier=frontend
k get pods --show-labels
k delete -f web.yaml
k get pods
```

![k get pods --show-labels](images/lab-1-2/05-show-labels.png)

Le Pod `web` porte maintenant deux labels : `run=web` (ajouté par `kubectl run`) et `tier=frontend` (ajouté à la main).

### Questions de réflexion

**1. Quelle différence entre `kubectl create` et `kubectl apply` ?**

`kubectl create` est **impératif** : il crée l'objet et échoue s'il existe déjà. `kubectl apply` est **déclaratif** : il crée l'objet s'il n'existe pas, ou le met à jour pour qu'il corresponde au fichier. On peut le relancer autant de fois que nécessaire, ce qui le rend adapté à un fichier suivi dans Git.

**2. À quoi sert `--dry-run=client -o yaml` ?**

À **générer le manifeste YAML sans rien créer** dans le cluster : `kubectl` construit l'objet localement, l'affiche au format YAML et ne l'envoie pas à l'API. C'est un moyen rapide d'obtenir un squelette (`apiVersion`, `kind`, `metadata`, `spec`) à modifier, au lieu de l'écrire à la main.

**3. Où trouver la raison d'un Pod qui ne démarre pas ?**

Dans la section **`Events`** de `kubectl describe pod <nom>` (planification, téléchargement de l'image, erreurs de démarrage). On complète avec `kubectl logs <pod>` (et `--previous` après un redémarrage) pour lire la sortie du conteneur.

### Nettoyage

```bash
k delete namespace tp-k8s
k config set-context --current --namespace=default
```

---
