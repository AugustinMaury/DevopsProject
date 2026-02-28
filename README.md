# DevOps Project - GitOps Pipeline avec Kind, Jenkins, ArgoCD

## Table des matières
1. [Création du cluster](#1---création-du-cluster)
2. [Création des namespaces](#2---création-des-namespaces)
3. [Installation de Gitea, Jenkins et Nginx](#3---installation-de-gitea-jenkins-et-nginx-ingress-controller)
4. [Installation de ArgoCD](#4---installation-de-argocd)
5. [Installation de ArgoCD CLI](#5---installation-de-argocd-cli)
6. [Création du Docker Registry](#6---création-du-docker-registry)
7. [Ingress config](#7---ingress-config)
8. [Configuration des repositories](#8---configuration-des-repositories)
9. [Setup de ArgoCD](#9---setup-de-argocd)
10. [Monitoring](#10---monitoring)
11. [Logging](#11---logging)


---

## 1 - Création du cluster

Création du cluster avec Kind et le fichier de config `kind-config.yaml` :
```bash
kind create cluster --config kind-config.yaml --name devops-project
```

La config :
- Utilise une image spécifique de Kind
- Fait un port mapping pour 443 et 80
- Ajoute un endpoint pour le registry d'image

---

## 2 - Création des namespaces

Création des namespaces dans le cluster : `infra`, `dev`, `prod`, `argocd`, `ingress-nginx`
```bash
kubectl create namespace dev
```
```bash
kubectl create namespace prod
```
```bash
kubectl create namespace infra
```
```bash
kubectl create namespace argocd
```
```bash
kubectl create namespace ingress-nginx
```

Pour utiliser le namespace infra par défaut (toutes les commandes suivantes gèrent les namespaces) :
```bash
kubectl config set-context --current --namespace=infra
```

---

## 3 - Installation de Gitea, Jenkins et Nginx (ingress controller)

Si vous ne voulez pas gérer l'ingress et passer par des port-forward : skip l'installation de Nginx.

### Ajout des repos Helm

Installer Helm si pas déjà fait, puis ajouter les repos :
```bash
helm repo add gitea-charts https://dl.gitea.io/charts/
```
```bash
helm repo add jenkins https://charts.jenkins.io
```
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```
```bash
helm repo list  # pour vérif que c'est bon
```
```bash
helm repo update
```

### Installation dans le cluster
```bash
helm install gitea gitea-charts/gitea -f .\gitea-values.yaml
```
```bash
helm install jenkins jenkins/jenkins -f .\jenkins-values.yaml
```
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx -f ingress-nginx-values.yaml -n ingress-nginx
```

### Credentials par défaut

Dans les fichiers de config pour Gitea et Jenkins, on crée les credentials :
- **Username** : `admin`
- **Password** : `admin123`

Dans le fichier config de Jenkins, on crée automatiquement le job pour la pipeline et on ajoute les credentials de Gitea + le trigger de la pipeline (push sur le repo).\
Dans le fichier config de Gitea, on ajoute la liste des adresses qui peuvent communiquer avec gitea (dans notre cas, tout le cluster)

---

## 4 - Installation de ArgoCD

ArgoCD a besoin de permissions très larges donc on le sépare du reste de l'infra dans lequel on veut pouvoir réduire les permissions pour de la sécurité (infra crée des ressources, ArgoCD déploie des ressources).
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## 5 - Installation de ArgoCD CLI

> Dans un PowerShell avec permissions administrateur
```powershell
$version = (Invoke-RestMethod https://api.github.com/repos/argoproj/argo-cd/releases/latest).tag_name
```
```powershell
$url = "https://github.com/argoproj/argo-cd/releases/download/$version/argocd-windows-amd64.exe"
```
```powershell
Invoke-WebRequest -Uri $url -OutFile argocd.exe  # environ 210Mo
```
```powershell
Move-Item .\argocd.exe C:\Windows\System32\argocd.exe
```

> ⚠️ Si vous avez une erreur rate limit de l'API GitHub, il faut utiliser un wifi perso et pas celui de l'Efrei.

Pour vérifier l'installation :
```bash
argocd version
```

---

## 6 - Création du Docker Registry

Le registry est créé comme un container externe au cluster car le kubelet qui doit créer les pods n'a pas accès au DNS du cluster. Il ne peut donc pas pull une image d'un registry interne qui est resolvable uniquement si on a accès au DNS du cluster.
```bash
docker run -d --restart=no --name kind-registry --network kind -p 5000:5000 registry:2
```

---

## 7 - Ingress config

Si vous avez skip l'installation de l'ingress controller, faire cette étape à la place : [Port-forwards](#port-forwards)

### Application des règles d'ingress
```bash
kubectl apply -f ingress-rules.yaml
```

### Modification du fichier hosts

Il faut modifier le fichier `etc/hosts` en ajoutant ces lignes à la fin :
```
127.0.0.1 jenkins.local
127.0.0.1 gitea.local
127.0.0.1 argocd.local
127.0.0.1 whoami.local
```

> ⚠️ Il faut ouvrir le notepad en tant qu'administrateur.

Localisation du fichier hosts sur Windows :
```
C:\Windows\System32\drivers\etc
```

Bon courage sur Linux et Mac lol

---

## 8 - Configuration des repositories

Créer 2 repos pour le source code et les manifests dans Gitea, puis push le bon code dans les repos.

### App_Repo

Structure :
```
Jenkinsfile
Dockerfile
main.go
go.mod
```
Il faut rajouter le webhook pour trigger la pipeline : Settings > Webhooks > Add Webhook > Gitea

Target URL : 
```
http://jenkins.infra.svc.cluster.local:8080/generic-webhook-trigger/invoke?token=whoami-trigger-token
```
rien d'autre à changer pour créer le webhook.

### Manifests_Repo

Structure :
```
dev/
├── deployment.yaml
├── configmap.yaml
├── ingress.yaml
└── service.yaml
prod/
	to do
```

---

## 9 - Setup de ArgoCD

### Connexion

username : `admin`

Récupérer le mot de passe à décoder (encodé en base64) :
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}"
```

### Login dans ArgoCD

Avec l'ingress :
```bash
argocd login argocd.local --insecure 
```

Avec un port-forward :
```bash
argocd login localhost:8081 --insecure 
```

### Ajouter le repo dans ArgoCD
```bash
argocd repo add http://gitea-http.infra.svc.cluster.local:3000/admin/Manifests_Repo.git --username admin --password admin123 --insecure-skip-server-verification
```

### Création des applications dev et prod
```bash
kubectl apply -f argocd-apps.yaml
```

Sync l'app quand la pipeline passe (pour pas attendre 5 minutes pour qu'ArgoCD update) :
```bash
argocd app sync whoami-goapp-dev
```

### Création du webhook pour trigger un sync argocd

Ajouter un hook dans le ManifestsRepo avec l'url suivante : 
```bash
http://argocd-server.argocd.svc.cluster.local/api/webhook
```

Créer le certificat ArgoCD:
```bash
kubectl port-forward svc/argocd-server -n argocd 8443:443
```
Puis dans GitBash :

```bash
openssl s_client -connect localhost:8443 -showcerts </dev/null 2>/dev/null | sed -n '/-----BEGIN/,/-----END/p' > argocd.crt
```

Puis crée un secret avec kubectl :

```bash
kubectl create secret generic argocd-cert --from-file=argocd.crt
```
---

## Port-forwards

Si vous n'utilisez pas l'ingress controller, utilisez ces commandes pour accéder aux services.

**Pour Gitea :**
```bash
kubectl port-forward svc/gitea-http 3000:3000
```

**Pour Jenkins :**
```bash
kubectl port-forward svc/jenkins 8080:8080
```

**Pour ArgoCD :**
```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

**Pour l'application :**

> À faire uniquement quand l'app est créée.
```bash
kubectl port-forward svc/whoami-goapp -n dev 8082:80
```

Continuer la config : [Configuration des repositories](#8---configuration-des-repositories)

---

## 10 - Monitoring

### Installation

Ajouter le repo Helm Prometheus :
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
```bash
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring -f grafana-values.yaml
```

Ajouter `grafana.local` dans `etc/hosts` pour l'ingress.



### Alerting

Créer le secret SMTP pour l'alertmanager :
```bash
kubectl create secret generic alertmanager-smtp --from-literal=password='aaaa aaaa aaaa aaaa' -n monitoring
```

Remplacer `aaaa aaaa aaaa aaaa` par un vrai app password.

### Tester l'alerting

Lancer un pod qui échoue :
```bash
kubectl run test-fail --image=busybox -n infra -- /bin/sh -c "exit 1"
```

Attendre 2 minutes, puis supprimer le pod :
```bash
kubectl delete pod test-fail
```

---

## 11 - Logging

### Installation de Loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```
```bash
helm repo update
```
```bash
helm install loki grafana/loki -n monitoring -f loki-values.yaml
```