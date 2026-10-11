# Mini-projet 1 — Déployer une application web à deux niveaux avec Kubernetes (k3s)

## 1. Présentation du projet

Ce mini-projet a pour objectif de déployer une application web composée de deux services sur un cluster Kubernetes k3s à trois nœuds.

L'application comprend :
- **Frontend :** interface web basée sur NGINX.
- **API :** service REST basé sur Podinfo, qui renvoie des informations au format JSON.
- **Services ClusterIP :** communication interne entre les composants.
- **Ingress Traefik :** point d'entrée HTTP permettant d'accéder au frontend et à l'API par des noms d'hôte.

Le déploiement est réalisé de manière déclarative à l'aide de manifestes YAML versionnés dans Git.

## 2. Objectifs pédagogiques

- Déployer des applications avec des Deployments Kubernetes.
- Gérer les réplicas et les labels.
- Exposer les applications avec des Services ClusterIP.
- Tester la résolution DNS interne de Kubernetes.
- Configurer un Ingress Traefik pour plusieurs noms d'hôte.
- Effectuer une mise à jour progressive (*Rolling Update*) et un rollback.
- Diagnostiquer et corriger des pannes volontaires.
- Versionner les manifestes et les preuves de validation dans GitHub.

## 3. Architecture

```text
                         Utilisateur
                              |
                       Poste Windows
                              |
                       Ingress Traefik
                        /           \
             web.k3s.local         api.k3s.local
                    |                    |
              frontend-svc            api-svc
               ClusterIP              ClusterIP
                    |                    |
             Deployment frontend  Deployment api
                2 réplicas            3 réplicas
                    |                    |
              NGINX Pods             Podinfo Pods
                    \                    /
                     \--- DNS interne ---/
```

### Environnement technique

| Composant | Technologie |
|---|---|
| Orchestration | Kubernetes k3s |
| Machines virtuelles | VMware Workstation |
| Système des nœuds | Ubuntu Server |
| Frontend | NGINX |
| API | Podinfo |
| Routage HTTP | Traefik Ingress Controller |
| Configuration | YAML |
| Versionnement | Git et GitHub |

## 4. Structure du dépôt

```text
mini-projet-1-k3s/
├── manifests/
│   ├── namespace.yaml
│   ├── api-deployment.yaml
│   ├── frontend-deployment.yaml
│   ├── api-service.yaml
│   ├── frontend-service.yaml
│   └── ingress.yaml
├── screenshots/
└── README.md
```

Le dossier `manifests/` contient les ressources Kubernetes. Le dossier `screenshots/` regroupe les captures d'écran démontrant les tests et les résultats obtenus.

## 5. Prérequis

- Un cluster k3s constitué d'un nœud control plane et de deux workers.
- Les trois nœuds en état `Ready`.
- Traefik installé et opérationnel.
- Un accès SSH au serveur k3s.
- Git installé et un dépôt GitHub.
- Un accès réseau entre le poste Windows et les nœuds Kubernetes.

Vérification du cluster :

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

## 6. Déploiement

### 6.1 Créer le namespace

Toutes les ressources de l'application doivent être placées dans un namespace dédié au binôme.

```bash
kubectl apply -f manifests/namespace.yaml
kubectl get namespaces
```

### 6.2 Déployer l'API

L'API utilise trois réplicas et une version d'image explicitement définie dans le manifeste.

```bash
kubectl apply -f manifests/api-deployment.yaml
kubectl get deployment -n mon-binome
kubectl get pods -n mon-binome -o wide
```

Remplacer `mon-binome` par le nom réel du namespace utilisé.

### 6.3 Déployer le frontend

Le frontend NGINX utilise deux réplicas.

```bash
kubectl apply -f manifests/frontend-deployment.yaml
kubectl get deployment -n mon-binome
kubectl get pods -n mon-binome -o wide
```

### 6.4 Créer les Services

Les deux composants sont exposés à l'intérieur du cluster à l'aide de Services de type `ClusterIP`.

```bash
kubectl apply -f manifests/api-service.yaml
kubectl apply -f manifests/frontend-service.yaml
kubectl get services -n mon-binome
kubectl get endpoints -n mon-binome
```

### 6.5 Vérifier la communication interne

Depuis un Pod du frontend, tester l'API avec le nom DNS du Service, sans utiliser directement son adresse IP.

Exemple de requête :

```bash
kubectl exec -n mon-binome NOM_DU_POD_FRONTEND -- \
  wget -qO- http://api-svc:9898
```

Le nom `api-svc` doit correspondre au Service défini dans le manifeste. Le résultat attendu est une réponse HTTP de l'API.

### 6.6 Configurer l'Ingress

Appliquer le manifeste Ingress :

```bash
kubectl apply -f manifests/ingress.yaml
kubectl get ingress -n mon-binome
kubectl describe ingress -n mon-binome
```

Les deux noms d'hôte utilisés sont :

- `web.k3s.local` : accès au frontend.
- `api.k3s.local` : accès à l'API.

Sur Windows, ajouter au fichier `C:\Windows\System32\drivers\etc\hosts`, en utilisant un éditeur lancé en administrateur, une ligne avec l'adresse IP d'un nœud accessible :

```text
IP_DU_NOEUD web.k3s.local api.k3s.local
```

Remplacer `IP_DU_NOEUD` par l'adresse IP réelle du nœud.

Tester ensuite depuis PowerShell :

```powershell
curl.exe -i http://web.k3s.local
curl.exe -i http://api.k3s.local
```

## 7. Mise à jour progressive et rollback

### 7.1 Préparer le test

Avant la mise à jour, relever la version actuelle de l'API et lancer une boucle de requêtes depuis PowerShell :

```powershell
while ($true) {
    curl.exe -s -o NUL -w "%{http_code}`n" http://api.k3s.local
    Start-Sleep -Milliseconds 500
}
```

Cette commande affiche le code HTTP de chaque requête. Interrompre la boucle avec `Ctrl + C`.

### 7.2 Mettre à jour l'API

Modifier le tag de l'image dans `manifests/api-deployment.yaml` pour utiliser la version plus récente retenue pour le projet, puis appliquer le manifeste :

```bash
kubectl apply -f manifests/api-deployment.yaml
kubectl rollout status deployment/api -n mon-binome
kubectl get replicasets -n mon-binome
kubectl get pods -n mon-binome
```

Pendant l'opération, vérifier les réponses HTTP de la boucle. Un test sans interruption doit montrer que les requêtes continuent de réussir pendant le remplacement progressif des Pods.

### 7.3 Effectuer un rollback

Pour revenir à la révision précédente du Deployment :

```bash
kubectl rollout undo deployment/api -n mon-binome
kubectl rollout status deployment/api -n mon-binome
kubectl rollout history deployment/api -n mon-binome
```

Vérifier ensuite la version de l'API et l'état des Pods.

## 8. Diagnostic des pannes

### 8.1 Panne A — Tag d'image inexistant

**Injection :** remplacer temporairement le tag valide par un tag inexistant dans le manifeste de l'API.

**Observations :**

```bash
kubectl get pods -n mon-binome
kubectl describe pod NOM_DU_POD -n mon-binome
kubectl get events -n mon-binome --sort-by=.metadata.creationTimestamp
```

**Cause attendue :** l'image demandée ne peut pas être récupérée depuis le registre, ce qui peut provoquer un état `ErrImagePull` ou `ImagePullBackOff`.

**Correction :** restaurer le tag valide, réappliquer le manifeste et vérifier le retour des trois réplicas à l'état opérationnel.

### 8.2 Panne B — Sélecteur de Service incorrect

**Injection :** modifier temporairement le sélecteur du Service API afin qu'il ne corresponde plus aux labels des Pods API.

**Diagnostic :**

```bash
kubectl get services -n mon-binome
kubectl get pods -n mon-binome --show-labels
kubectl get endpoints api-svc -n mon-binome
kubectl describe service api-svc -n mon-binome
```

**Symptôme attendu :** le Service ne possède plus d'endpoints correspondant aux Pods API et les requêtes peuvent échouer.

**Correction :** rétablir les labels attendus dans le sélecteur du Service, réappliquer le manifeste et vérifier que les endpoints sont de nouveau présents.

Les pannes sont injectées uniquement dans le cadre du test, puis corrigées. Les résultats observés et la cause racine doivent être consignés dans le compte rendu.

## 9. Captures d'écran

Les preuves du projet sont regroupées dans le dossier `screenshots/`.

| Capture | Résultat démontré |
|---|---|
| `01-cluster-nodes-ready.png` | Les nœuds du cluster sont prêts |
| `02-namespace-created.png` | Namespace dédié |
| `03-api-deployment-3-replicas.png` | Trois réplicas API |
| `04-api-pods-distribution.png` | État et répartition des Pods API |
| `05-frontend-deployment-2-replicas.png` | Deux réplicas frontend |
| `06-services-clusterip.png` | Services internes |
| `07-api-endpoints.png` | Endpoints du Service API |
| `08-frontend-api-dns-test.png` | Communication interne par DNS |
| `09-ingress-rules.png` | Règles Ingress |
| `10-traefik-ingress-test.png` | Test de l'API via Ingress |
| `11-frontend-access.png` | Accès au frontend |
| `12-api-version-before-update.png` | Version initiale |
| `13-rolling-update-no-downtime.png` | Requêtes pendant la mise à jour |
| `14-replicasets-after-update.png` | ReplicaSets avant/après |
| `15-api-version-after-update.png` | Nouvelle version |
| `16-rollback-completed.png` | Retour à la révision précédente |
| `17-image-pull-error.png` | Symptômes du tag inexistant |
| `18-diagnostic-events.png` | Événements et diagnostic |
| `19-image-error-fixed.png` | Rétablissement de l'API |
| `20-service-selector-error.png` | Endpoints absents après l'erreur |
| `21-service-selector-corrected.png` | Endpoints restaurés |
| `22-git-repository.png` | Organisation du dépôt GitHub |

Ne conserver dans ce tableau que les captures réellement réalisées.

## 10. Réponses aux questions pédagogiques

### Étape 0 — Pourquoi un namespace ?

Un namespace permet d'isoler logiquement les ressources d'un projet dans un même cluster. Il facilite l'organisation, l'administration et la séparation des ressources entre les binômes.

### Étape 1 — Répartition des Pods API

Le scheduler Kubernetes choisit les nœuds sur lesquels placer les Pods en fonction des ressources disponibles, des contraintes et des règles de planification. La répartition réelle doit être vérifiée avec `kubectl get pods -o wide`.

### Étape 2 — Pourquoi des labels distincts ?

Les labels permettent d'identifier les ressources et les sélecteurs des Services. Des labels mal choisis ou identiques dans les mauvais contextes peuvent entraîner une sélection involontaire de Pods.

### Étape 3 — DNS et endpoints

Un Service possède un nom DNS interne. Dans le même namespace, il peut être appelé par son nom court, par exemple `api-svc`. Son nom DNS complet suit la forme :

`api-svc.mon-binome.svc.cluster.local`

Les endpoints indiquent les destinations auxquelles le Service peut acheminer le trafic. Leur absence peut révéler un sélecteur incorrect ou des Pods non prêts.

### Étape 4 — Pourquoi un Ingress ?

Un Ingress permet de centraliser le routage HTTP par nom d'hôte ou chemin vers plusieurs Services. Il évite de devoir exposer séparément chaque application avec un NodePort. Le contrôleur Traefik applique les règles configurées.

### Étape 5 — ReplicaSets et rollback

Un Rolling Update remplace progressivement les Pods de l'ancienne version par ceux de la nouvelle version, selon la stratégie de mise à jour du Deployment. Les ReplicaSets conservent l'historique des révisions et peuvent permettre un retour en arrière. La commande `kubectl rollout undo` déclenche ce rollback.

### Étape 6 — Diagnostic

Le diagnostic repose sur une démarche progressive : observer l'état des Pods, examiner les détails et événements, vérifier les labels et les endpoints, puis corriger la cause racine. Les symptômes et la correction effectivement observés doivent être décrits dans le compte rendu.

## 11. Difficultés rencontrées et enseignements

Ce mini-projet permet de comprendre l'importance de la cohérence entre les labels des Pods et les sélecteurs des Services, ainsi que le rôle du DNS interne dans la communication entre composants. Il montre également l'intérêt des manifestes déclaratifs versionnés, des mises à jour progressives et des outils de diagnostic Kubernetes pour identifier et corriger les erreurs.

**À compléter avant la remise :** ajouter les difficultés réellement rencontrées, les commandes utilisées et les résultats observés pendant la séance.

## 12. Conclusion

Ce projet met en pratique les fondations de Kubernetes avec k3s : déploiement déclaratif, réplication, découverte de services, exposition HTTP avec Ingress, mise à jour progressive et diagnostic. Les manifestes et les captures permettent de documenter les étapes et de faciliter la reproduction du déploiement.

## 13. Auteur(s)

- **Étudiante :** Arij Belaid

- **Formation :** Master Professionnel DevOps & Cloud
- **Établissement :** ISET Tozeur
- **Module :** Orchestration de conteneurs avec Kubernetes (k3s)
