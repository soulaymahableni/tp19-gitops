# TP19 : GitOps avec Argo CD

Ce dépôt est la **source de vérité** du déploiement de `whoami` : Argo CD, installé dans le cluster K3s, compare en permanence ce dépôt et le cluster, et les réconcilie. On ne fait plus de `kubectl apply` à la main : on fait une Pull Request.
Les réponses aux questions sont dans [REPONSES.md](REPONSES.md).

```
.
├── base/                         ← Deployment + Service whoami (durcis)
├── envs/
│   ├── dev/                      ← whoami-dev     : 1 replica, v1.10.1
│   ├── staging/                  ← whoami-staging : 2 replicas, v1.10.0
│   └── prod/                     ← whoami-prod    : 3 replicas, v1.10.0
├── argocd/
│   ├── application-dev.yaml      ← Partie 2-3 : Application whoami-dev
│   ├── applicationset.yaml       ← Partie 4 : une Application par dossier envs/* (prod en manuel)
│   └── defi/application-helm.yaml← chart Helm du TP15 via Argo CD
└── captures/
```

## 0. Préparation

Argo CD demande environ 1 Go de RAM. Retirer les outils lourds des TP précédents **après les avoir poussés sur GitHub** :

```bash
free -m
helm uninstall falco -n falco 2>/dev/null; helm uninstall trivy-operator -n trivy-system 2>/dev/null
helm uninstall vault -n vault 2>/dev/null; helm uninstall external-secrets -n external-secrets 2>/dev/null
kubectl delete ns tp17 tp18 falco trivy-system vault external-secrets --ignore-not-found
free -m
```

Créer le dépôt **public** `tp19-gitops` sur GitHub (sans README), puis :

```bash
cp -r gitops-starter ~/tp19-gitops && cd ~/tp19-gitops
sed -i 's/TODO-utilisateur/soulaymahableni/' argocd/*.yaml argocd/defi/*.yaml
git init -b main && git add . && git commit -m "feat: dépôt GitOps initial"
git remote add origin https://github.com/soulaymahableni/tp19-gitops.git
git push -u origin main
```

## 1. Partie 1 : installer Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server --timeout=900s     # images : plusieurs minutes
kubectl -n argocd get pods
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8443:443 --address 0.0.0.0 >/dev/null 2>&1 &
```

Navigateur (PC) : `https://192.168.190.141:8443` → *Paramètres avancés → Continuer* → `admin` + mot de passe → *User Info → Update password*.

CLI :

```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 755 argocd /usr/local/bin/argocd && rm argocd
argocd login localhost:8443 --insecure --username admin       # nouveau mot de passe
argocd version --short
```

## 2. Partie 2 : première Application

```bash
kubectl apply -f argocd/application-dev.yaml
argocd app get whoami-dev            # OutOfSync / Missing
argocd app diff whoami-dev
argocd app sync whoami-dev
kubectl -n whoami-dev get all
```

**Capture `argocd-synced.png`** : dans l'UI, `whoami-dev` **Synced** et **Healthy**, avec l'arbre des ressources (Service, Deployment, ReplicaSet, Pod).

## 3. Partie 3 : sync automatique, prune, self-heal, rollback

Activer `automated` : décommenter les 3 lignes dans `argocd/application-dev.yaml`, puis :

```bash
git commit -am "feat(dev): synchronisation automatique, prune et self-heal" && git push
kubectl apply -f argocd/application-dev.yaml
```

Changement par Git :

```bash
sed -i 's/count: 1/count: 2/' envs/dev/kustomization.yaml
git commit -am "feat(dev): 2 replicas" && git push
argocd app get whoami-dev --refresh
kubectl -n whoami-dev get deploy whoami        # 2/2
```

Self-heal :

```bash
kubectl -n whoami-dev scale deploy whoami --replicas=5
kubectl -n whoami-dev get deploy whoami -w     # 5 → revient à 2 en quelques secondes, Ctrl+C
```

**Capture `self-heal.png`** : le `scale` à 5 puis le retour à 2.

Prune :

```bash
git rm base/service.yaml && sed -i '/service.yaml/d' base/kustomization.yaml
git commit -m "test: suppression du Service" && git push
argocd app get whoami-dev --refresh && kubectl -n whoami-dev get svc     # plus de Service
git revert --no-edit HEAD && git push
argocd app get whoami-dev --refresh && kubectl -n whoami-dev get svc     # Service revenu
```

Rollback par Git :

```bash
sed -i 's/newTag: v1.10.1/newTag: v9.9.9/' envs/dev/kustomization.yaml
git commit -am "feat(dev): whoami v9.9.9" && git push
argocd app get whoami-dev --refresh
sleep 60
argocd app get whoami-dev | grep -E 'Sync Status|Health Status'      # Degraded (ou Progressing)
kubectl -n whoami-dev get pods                                       # ImagePullBackOff
git revert --no-edit HEAD && git push
argocd app get whoami-dev --refresh
sleep 30
argocd app get whoami-dev | grep -E 'Sync Status|Health Status'      # Synced / Healthy
git log --oneline -4
```

**Capture `git-revert.png`** : l'état `Degraded`, puis `Healthy` après le `git revert` (UI ou CLI), avec `git log`.

## 4. Partie 4 : ApplicationSet

```bash
argocd app delete whoami-dev --cascade=false -y
kubectl apply -f argocd/applicationset.yaml
sleep 20
argocd app list
argocd app sync whoami-prod                    # prod : synchronisation manuelle
kubectl get deploy -A --field-selector metadata.name=whoami \
  -o custom-columns=NS:.metadata.namespace,REPLICAS:.spec.replicas,IMAGE:.spec.template.spec.containers[0].image
```

**Capture `applicationset.png`** : les trois Applications (UI ou `argocd app list`) et les trois Deployments (replicas 2 / 2 / 3, images différentes).

## 5. Partie 5 : promotion par Pull Request

```bash
git switch -c promote/staging-v1.10.1
sed -i 's/newTag: v1.10.0/newTag: v1.10.1/' envs/staging/kustomization.yaml
git diff
git commit -am "chore(staging): promotion whoami v1.10.1" && git push -u origin HEAD
```

Sur GitHub : *Compare & pull request* → titre clair → **Create pull request** → relire l'onglet *Files changed* → **Merge pull request**.

**Capture `promotion-pr.png`** : la PR (titre, diff `v1.10.0 → v1.10.1`, merge).

```bash
git switch main && git pull
argocd app get whoami-staging --refresh
sleep 60
clear
kubectl -n whoami-dev     get pod -l app=whoami -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.status.containerStatuses[0].imageID}{"\n"}{end}'
kubectl -n whoami-staging get pod -l app=whoami -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.status.containerStatuses[0].imageID}{"\n"}{end}'
```

**Capture `same-digest.png`** : le même `sha256:…` en dev et en staging.

Promotion vers prod (même méthode, branche `promote/prod-v1.10.1`, `envs/prod/kustomization.yaml`), puis après le merge :

```bash
git switch main && git pull
argocd app get whoami-prod --refresh
argocd app sync whoami-prod                    # manuel : c'est la décision de mise en production
```

Protection de `main` : *Settings → Rules → Rulesets → New branch ruleset* → nom `main`, *Enforcement: Active*, *Target branches: Include default branch*, cocher **Require a pull request before merging** avec **1 required approval** → *Create*. (Seul sur le dépôt, on ne peut pas approuver sa propre PR : ajouter *Repository admin* dans *Bypass list* pour pouvoir merger pendant le TP.)

## 6. Défis

* Chart Helm du TP15 : `kubectl apply -f argocd/defi/application-helm.yaml`.
* Argo CD Image Updater et multi-cluster : voir REPONSES.md.

## 7. Nettoyage

```bash
kubectl -n argocd delete applicationset whoami          # supprime aussi les Applications et leurs ressources
kubectl delete ns whoami-dev whoami-staging whoami-prod --ignore-not-found
pkill -f "port-forward svc/argocd-server"
```

## Captures

(à compléter)
