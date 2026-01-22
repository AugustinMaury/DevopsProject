# DevOps Project - GitOps Pipeline avec Kind, Jenkins, ArgoCD

## Table des matières
1. [Création du cluster](#1---création-du-cluster)
2. [Création des namespaces](#2---création-des-namespaces)
3. [Installation de Gitea](#3---installation-de-gitea)
4. [Configuration des repositories](#4---configuration-des-repositories)
5. [Installation de Jenkins](#5---installation-de-jenkins)
6. [Installation de ArgoCD](#6---installation-de-argocd)
7. [Installation de ArgoCD CLI](#7---installation-de-argocd-cli)
8. [Création du Docker Registry](#8---création-du-docker-registry)
9. [Configuration des credentials Jenkins](#9---configuration-des-credentials-jenkins)
10. [Création du job pipeline](#10---création-du-job-pipeline)
11. [Setup de ArgoCD](#11---setup-de-argocd)

---

## 1 - Création du cluster

Création du cluster avec Kind et un fichier `kind-config.yaml` :

```bash
kind create cluster --config kind-config.yaml --name devops-project
```

---

## 2 - Création des namespaces

Création des namespaces dans le cluster : `infra`, `dev`, `prod`, `argocd`

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl create namespace infra
kubectl create namespace argocd
```

Pour utiliser un namespace par défaut :

```bash
kubectl config set-context --current --namespace=infra
```

---

## 3 - Installation de Gitea

> GitHub mais en local - à installer dans `infra`

Installer Helm si pas déjà fait, puis :

```bash
helm repo add gitea-charts https://dl.gitea.io/charts/
helm repo update
helm repo list  # pour vérif que c'est bon

helm install gitea gitea-charts/gitea -f .\gitea-values.yaml
```

**Pour utiliser Gitea :**

```bash
kubectl port-forward svc/gitea-http 3000:3000
```

---

## 4 - Configuration des repositories

Créer 2 repos pour le source code et les manifests dans Gitea, puis push le bon code dans les repos :

### App_Repo
- `Jenkinsfile` - décrit la pipeline
- `Dockerfile` - décrit comment build l'image de l'app
- App source code

### Manifests_Repo
```
dev/
├── deployment.yaml
├── configmap.yaml
└── service.yaml
prod/
	to do
```

---

## 5 - Installation de Jenkins

> À installer dans `infra`

```bash
helm repo add jenkins https://charts.jenkins.io 
helm repo update

helm install jenkins jenkins/jenkins -f .\jenkins-values.yaml  
```

**Pour utiliser Jenkins :**

```bash
kubectl port-forward svc/jenkins 8080:8080
```

---

## 6 - Installation de ArgoCD

> À installer dans le namespace `argocd`

ArgoCD a besoin de permissions très larges donc on le sépare du reste de l'infra dans lequel on veut pouvoir réduire les permissions pour de la sécurité (infra crée des ressources, ArgoCD déploie des ressources).

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## 7 - Installation de ArgoCD CLI

> Dans un PowerShell avec permissions administrateur

```powershell
$version = (Invoke-RestMethod https://api.github.com/repos/argoproj/argo-cd/releases/latest).tag_name
$url = "https://github.com/argoproj/argo-cd/releases/download/$version/argocd-windows-amd64.exe" 
Invoke-WebRequest -Uri $url -OutFile argocd.exe  # environ 210Mo
Move-Item .\argocd.exe C:\Windows\System32\argocd.exe
```

!! Si vous avez une erreur rate limit de l'api github, il faut utiliser un réseau perso et pas celui de l'efrei

**Pour vérifier l'installation :**

```bash
argocd version
```

---

## 8 - Création du Docker Registry

Le registry est créé comme un container externe au cluster car le kubelet qui doit créer les pods n'a pas accès au DNS du cluster. Il ne peut donc pas pull d'un registry interne qui est resolvable uniquement si on a accès au DNS du cluster.

```bash
docker run -d --restart=no --name kind-registry --network kind -p 5000:5000 registry:2
```

---

## 9 - Configuration des credentials Jenkins

Ajouter les credentials Gitea dans Jenkins :
- Mettre l'id `gitea-credentials`
- Mettre le même username/password que dans le `gitea-values.yaml`

---

## 10 - Création du job pipeline

Créer le job pipeline dans Jenkins qui va utiliser le Jenkinsfile dans le source code repo.
- Definition `Pipeline script from SMC`
- Repository URL : `http://gitea-http.infra.svc.cluster.local:3000/admin/App_Repo.git`
- ajouter les credentials créé à l'étape 9

---

## 11 - Setup de ArgoCD

### Exposer ArgoCD

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

### Connexion

| Champ | Valeur |
|-------|--------|
| Username | `admin` |
| Password | Voir commande ci-dessous |

**Récupérer le mot de passe à décoder** (encodé en base64) :

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}"
```

### Login dans ArgoCD

```bash
argocd login localhost:8081 --insecure 
```

### Ajouter le repo dans ArgoCD

```bash
argocd repo add http://gitea-http.infra.svc.cluster.local:3000/admin/Manifests_Repo.git --username admin --password admin123 --insecure-skip-server-verification
```

### Création de l'application

```bash
argocd app create whoami-goapp-dev --repo http://gitea-http.infra.svc.cluster.local:3000/admin/Manifests_Repo.git --path dev --dest-server https://kubernetes.default.svc --dest-namespace dev --sync-policy automated --auto-prune --self-heal
```

### Port forward l'application

```bash
kubectl port-forward svc/whoami-goapp -n dev 8082:80
```

---

## Test

```bash
curl localhost:8082/whoami
```

---

## TODO

- [ ] Config webhook Gitea pour lancer la pipeline Jenkins automatiquement


ajouter dans le read me :

changement du kind config
changement du jenkins config pour avoir le job + credentials directement
ajouter le ingress dans manifest pour que l'app soit accesible sans avoir a faire le port foward a la main
installer le nginx config pour que le ingress marche

