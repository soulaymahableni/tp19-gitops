# TP19 : réponses aux questions

## Les quatre principes GitOps (OpenGitOps)

1. **Déclaratif** : l'état désiré du système est décrit (manifests, Kustomize), pas une suite de commandes.
2. **Versionné et immuable** : cet état est stocké dans Git, avec l'historique complet, l'auteur et la date de chaque changement.
3. **Tiré automatiquement (pull)** : un agent dans le cluster (Argo CD) va chercher l'état désiré. Aucun accès entrant ni identifiant du cluster dans la CI : ici, Argo CD lit GitHub sans webhook ni port ouvert vers la VM.
4. **Réconcilié en continu** : l'agent compare sans cesse Git et le cluster, et corrige tout écart.

## Partie 2 : première Application

* Une **Application** relie une source (dépôt + chemin `envs/dev` + branche `main`) à une destination (cluster + namespace `whoami-dev`).
* Juste après sa création : **OutOfSync / Missing**, car Git décrit des ressources qui n'existent pas encore dans le cluster. `argocd app diff` montre exactement ce qui sera créé. Après `argocd app sync` : **Synced / Healthy**.
* `CreateNamespace=true` laisse Argo CD créer le namespace de destination.

## Partie 3 : sync automatique, prune, self-heal

* **automated** : Argo CD synchronise dès qu'il détecte un nouveau commit (vérification toutes les 3 minutes, ou immédiatement avec `--refresh` ou un webhook).
* **Changement par Git** (`replicas: 2`) : commit + push suffisent. Aucun `kubectl` n'a été tapé.
* **Self-heal** : `kubectl scale --replicas=5` crée une **dérive**. Argo CD voit que le cluster ne correspond plus à Git et remet 2 replicas en quelques secondes. Toute modification manuelle est annulée : Git est la seule source de vérité.
* **Prune** : sans `prune`, une ressource retirée de Git resterait orpheline dans le cluster. Avec `prune: true`, le Service supprimé de Git est supprimé du cluster.
* **Rollback par Git** : le tag `v9.9.9` produit `ImagePullBackOff` et l'Application passe **Degraded** (ou reste *Progressing* jusqu'au délai du rollout). Les anciens Pods continuent de servir grâce au rolling update. `git revert HEAD` crée un **nouveau commit** qui annule le précédent : l'historique garde la trace de l'erreur et de sa correction, et Argo CD redéploie automatiquement la version saine.
* **Pourquoi pas l'UI (History and rollback) ?** Avec `automated`, Argo CD remettrait aussitôt l'état décrit dans Git (`v9.9.9`). Le rollback dans l'UI créerait une différence entre Git et le cluster. En GitOps, on corrige toujours dans Git.

## Partie 4 : ApplicationSet

* Le **générateur git (directories)** parcourt `envs/*` et produit une Application par dossier : `whoami-dev`, `whoami-staging`, `whoami-prod`. Ajouter un environnement revient à ajouter un dossier.
* `--cascade=false` supprime l'Application `whoami-dev` **sans** supprimer ses ressources. L'ApplicationSet en recrée une du même nom, qui reprend les ressources existantes sans coupure.
* **Sync automatique pour dev et staging, manuelle pour la prod** : le champ `templatePatch` ajoute `syncPolicy.automated` uniquement si `{{.path.basename}}` est différent de `prod`. Alternatives : un générateur `list` avec un paramètre `autoSync: true/false` par environnement (combiné au générateur git avec un `matrix`), ou des **sync windows** dans un AppProject (par exemple : synchronisation de la prod autorisée seulement en semaine de 9 h à 17 h).
* Résultat : dev 2 replicas en `v1.10.1`, staging 2 replicas en `v1.10.0`, prod 3 replicas en `v1.10.0`.

## Partie 5 : promotion sans reconstruction

* La promotion **ne reconstruit pas l'image** : elle change seulement le tag référencé dans `envs/staging`. Staging exécute **exactement** l'artefact testé en dev, vérifiable par le **digest** (`imageID: …@sha256:…` identique dans les deux namespaces). Reconstruire produirait un autre artefact (dépendances, couches de base…), et le test en dev ne prouverait plus rien.
* Une **Pull Request par promotion** : le diff d'une ligne (`newTag`) est relu, approuvé et tracé. Après le merge, Argo CD déploie staging automatiquement. Pour la prod, la PR décide **ce qui** sera déployé, et `argocd app sync whoami-prod` décide **quand**.
* **Protection de `main`** (ruleset : PR obligatoire, 1 approbation) : plus personne ne peut pousser directement sur `main`. Comme Argo CD déploie tout ce qui arrive sur `main`, **la revue de la PR devient la revue du changement en production** : c'est le point de contrôle unique (quatre yeux), avec un historique d'audit complet (qui a proposé, qui a approuvé, quand). Le rollback est un `git revert` relu de la même façon.

## Défis

* **Argo CD Image Updater** (v1.x) : surveille le registre et, avec la stratégie `semver` limitée à `v1.10.*`, détecte le dernier tag. Il écrit le nouveau tag **dans Git** (write-back : commit dans `envs/dev`, par exemple via `.argocd-source-whoami-dev.yaml` ou `kustomization.yaml`). Argo CD déploie ensuite ce commit. Le principe GitOps est respecté : même une mise à jour automatique passe par Git.
* **Chart Helm via Argo CD** (`argocd/defi/application-helm.yaml`) : Argo CD fait lui-même le rendu (`helm template`) avec `valueFiles` et `parameters`. Il n'y a **pas de release Helm** (`helm list` ne montre rien) : l'historique et le rollback sont ceux d'Argo CD et de Git, pas ceux de Helm. Kustomize donne un YAML final plus lisible ; Helm reste pratique pour les logiciels tiers paramétrables.
* **Multi-cluster** : `argocd cluster add <contexte>` enregistre un second cluster ; un générateur **matrix** (clusters × git) produit une Application par couple (cluster, environnement). **Karmada**, à l'inverse, est un **plan de contrôle fédéré** : on applique les ressources une seule fois sur l'API Karmada, et des `PropagationPolicy` les répartissent (push) sur les clusters membres. Argo CD reste centré sur Git et la réconciliation (pull depuis le dépôt). Karmada est centré sur la planification et la répartition multi-cluster (basculement, pondération).
