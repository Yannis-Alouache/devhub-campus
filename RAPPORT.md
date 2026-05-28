# RAPPORT — TP 2 GitOps avec ArgoCD : DevHub Campus

## Etat de redaction

- [x] 0. Outillage
- [x] 1. GitOps en 1 page
- [x] 2. Glossaire ArgoCD
- [x] 3. Service choisi et containerisation
- [x] 4. Chart Helm
- [x] 5. Installation ArgoCD et premiere `Application`
- [x] 6. Pattern App of Apps
- [x] 7. `ApplicationSet` et previews
- [x] 8. Bestiaire ArgoCD
- [x] 9. Securite et observabilite d'ArgoCD
- [x] 11. Synthese obligatoire

## Journal d'avancement

- 2026-05-27 — Mise en place du socle local : outils prets, cluster `devhub` cree, ingress-nginx et ArgoCD installes, UI exposee sur `argocd.devhub.local`.
- 2026-05-27 — Phase GitOps stable en place : `annuaire-dev`, `planning-dev`, `notif-dev` et `root` obtenus en `Synced + Healthy`.
- 2026-05-27 — Etape previews reprise et documentee : lecture du poly et de la doc ApplicationSet, abandon de `scmProvider.github allBranches` pour un repo sur compte utilisateur, bascule des manifests preview vers `pullRequest`, root app elargie a `platform/apps/`.
- 2026-05-27 — Validation technique des previews : `kubectl apply --dry-run=server` passe sur les manifests preview et sur la root app mise a jour ; application temporaire des trois `ApplicationSet` sur le cluster pour verification controleur, avec `ParametersGenerated=True` et `ResourcesUpToDate=True`, puis nettoyage immediat pour ne pas laisser de drift manuel.
- 2026-05-27 — Limite reelle observee en cluster : le generateur `pullRequest` en acces anonyme a bute sur `403 API rate limit exceeded` cote GitHub. Correctif applique : `tokenRef` ajoute dans les trois `ApplicationSet` preview et `Secret` `github-token` a creer dans le namespace `argocd`, sans rien committer de sensible.
- 2026-05-27 — Demo live effectuee sur la PR `#1` (`feature/demo-preview-annuaire` -> `main`) : workflow `build-images` reussi, previews `annuaire-pr-1`, `planning-pr-1`, `notif-pr-1` generees, namespace `devhub-preview-pr-1` cree, `annuaire` observe avec 2 replicas et checks HTTP `/healthz` OK sur les trois hosts preview.
- 2026-05-27 — Nettoyage observe : fermeture de la PR `#1` + suppression de la branche distante = suppression automatique des trois `Application` preview. Le namespace partage est reste vide et a ete supprime manuellement ensuite.
- 2026-05-27 — Etat stable retrouve : plus aucune preview active hors PR ouverte ; `root`, `annuaire-dev`, `planning-dev` et `notif-dev` sont revenus en `Synced + Healthy`, tandis que les trois `ApplicationSet` preview restent installes et prets pour la prochaine PR.

## 0. Outillage

### Versions

- `docker version` : 28.5.2
- `kubectl version --client` : v1.35.0
- `helm version` : v4.2.0
- `argocd version --client` : v3.4.2 (CLI), v2.12.6 (serveur en cluster)
- `kind version` : v0.31.0
- `git --version` : 2.50.1

### Notes d'installation

- Docker Desktop installe avec le runtime Docker CLI.
- `kubectl`, `helm`, `argocd` et `kind` installes via Homebrew.
- Cluster local `devhub` cree avec `kind create cluster --config cluster/kind-config.yaml` (2 noeuds : control-plane + worker).
- Ingress-nginx installe via Helm sur le cluster.
- `/etc/hosts` renseigne avec les entrees `devhub.local` pour l'acces local.

## 1. GitOps en 1 page

### Schema personnel

```
  +----------------+         git push          +------------------+
  |  Dev / CI      |  ---------------------->  |  Depot Git       |
  |  (code + yaml) |     commit manifests      |  (GitHub)        |
  +----------------+                           +------------------+
                                                         |
                                                         | pull (periodique / webhook)
                                                         v
                                                +------------------+
                                                |  ArgoCD          |
                                                |  (in-cluster)    |
                                                +------------------+
                                                         |
                                                         | kubectl apply (auto-sync)
                                                         v
                                                +------------------+
                                                |  Cluster K8s     |
                                                |  (kind / devhub) |
                                                +------------------+
                                                         |
                                              +----------+----------+
                                              |          |          |
                                         annuaire    planning    notif
                                         (deploy)    (deploy)    (deploy)
```

Le principe : Git est l'unique source de verite. ArgoCD compare en permanence l'etat souhaite (dans Git) avec l'etat reel (dans le cluster) et corrige les ecarts automatiquement.

### Tableau push vs pull

| Question | Push (`kubectl apply` en CI) | Pull (ArgoCD) |
| --- | --- | --- |
| Qui a les droits sur le cluster ? | Le runner CI possede un kubeconfig avec droits eleves — surface d'attaque importante. | Seul ArgoCD (in-cluster) a les droits. Le CI n'a aucun acces au cluster. |
| Ou est l'historique des changements ? | Dans les logs CI (ephemeres) et les commits Git (manifests pas toujours commit). | Dans Git — chaque changement de manifest passe par un commit, l'historique est natif et auditable. |
| Que se passe-t-il si un dev modifie le cluster a la main ? | Rien ne detecte le drift. Le cluster derive silencieusement. | ArgoCD detecte le drift en moins de 3s (avec `selfHeal`) et re-synchronise automatiquement. |
| Comment ajouter un environnement de plus ? | Dupliquer le pipeline CI avec un nouveau kubeconfig. | Ajouter un dossier/branche dans Git + une `Application` ArgoCD. Pas de pipeline supplementaire. |
| Comment faire un rollback ? | `kubectl rollout undo` ou relancer le pipeline avec un tag precedent. | `git revert` — ArgoCD detecte le nouveau commit et converge. |
| Combien de pipelines pour 30 services ? | 30 pipelines minimum (un par service), plus les environnements. | 0 pipeline de deploiement. Un seul ArgoCD gere les 30 services via `App of Apps` ou `ApplicationSet`. |
| Qui voit en direct ce qui tourne ? | Il faut interroger le cluster (`kubectl get all`) ou un outil externe. | L'UI ArgoCD montre en temps reel l'etat de chaque application, ses ressources, la diff Git/cluster. |

### Prise de position

ArgoCD est plus contraignant au demarrage (installation, configuration RBAC, comprehension du modele declaratif), mais ce surcout est inverse proportionnel au nombre de services : a partir de 5-10 services, le fait de ne plus maintenir de pipelines de deploiement et d'avoir un audit natif dans Git compense largement. L'operation la plus convaincante est la correction automatique du drift : un `kubectl scale` manuel est annule en quelques secondes sans intervention humaine.

## 2. Glossaire ArgoCD

| Terme | Definition avec mes mots | Exemple dans mon projet |
| --- | --- | --- |
| `Application` | Ressource Kubernetes qui lie une source Git (chart Helm, kustomize, manifests bruts) a une destination dans le cluster. C'est l'unite de deploiement ArgoCD. | `annuaire-dev` pointe vers `services/annuaire/chart` dans Git et deploie dans le namespace `devhub-dev`. |
| `AppProject` | Regroupement logique d'applications avec des garde-fous : quels repos, quels clusters, quels namespaces sont autorises. | Le projet `devhub` autorise uniquement les sources depuis le repo `Yannis-Alouache/devhub-campus` et les destinations dans `devhub-*`. |
| `Source` | L'emplacement dans Git ou ArgoCD va chercher les manifests a deployer (repo URL + branche + chemin). | `repoURL: https://github.com/Yannis-Alouache/devhub-campus.git`, `path: services/annuaire/chart`, `targetRevision: main`. |
| `Destination` | Le cluster et le namespace cibles ou les ressources seront creees. | `server: https://kubernetes.default.svc`, `namespace: devhub-dev`. |
| `Sync` | L'operation qui rapproche l'etat reel du cluster de l'etat souhaite dans Git. Peut etre manuelle ou automatique (`auto-sync`). | Quand un commit modifie `values-dev.yaml`, ArgoCD detecte le changement et lance un sync pour mettre a jour le `Deployment`. |
| `Prune` | Suppression des ressources cluster qui ne sont plus presentes dans Git. Active avec `syncPolicy.automated.prune: true`. | Quand le fichier `service.yaml` a ete retire du chart, `prune: true` a supprime le Service du cluster. |
| `App of Apps` | Pattern ou une `Application` ArgoCD (dite "root") surveille un dossier Git contenant d'autres manifests `Application`, creant ainsi un arbre d'applications gere par ArgoCD. | `root` surveille `platform/apps/dev/` et cree `annuaire-dev`, `planning-dev`, `notif-dev`. |
| `ApplicationSet` | CRD qui genere dynamiquement des `Application` a partir de generateurs (liste, git, pullRequest). Permet le deploiement automatique par branche/PR. | Les trois `ApplicationSet` preview utilisent le generateur `pullRequest` pour creer `annuaire-pr-1`, `planning-pr-1`, `notif-pr-1` quand une PR est ouverte. |
| `Sync wave` | Mecanisme pour ordonner les ressources lors d'un sync. Les ressources de wave `-1` sont appliquees avant celles de wave `0`, etc. | Le `ConfigMap` est en wave `-1`, le `Deployment` en wave `0`, et le Job `PreSync` s'execute avant tout. |
| `Hook` | Ressource annotee `argocd.argoproj.io/hook` qui s'execute a un moment precis du cycle de sync (`PreSync`, `PostSync`, `SyncFail`). | Le Job `annuaire-dev-annuaire-migration` avec `hook: PreSync` s'execute avant le deploiement et bloque la sync s'il echoue. |

## 3. Service choisi et containerisation

- Service prioritaire retenu : `annuaire`.
- Contraintes a documenter : multi-stage, non-root, tag SHA, endpoint `/healthz`, `LOG_LEVEL`, label OCI source.
- Resultats a consigner : image produite, commande de test locale, taille de l'image, remarques.

### Dockerfile

Le Dockerfile (`services/annuaire/Dockerfile`) est en deux stages :

1. **Stage `build`** (base `node:20.19.5-alpine3.22`) :
   - Copie `.npmrc`, `package*.json`, puis `npm ci --omit=dev` pour installer uniquement les dependances de production.
   - Copie le code source (`src/`).

2. **Stage `runtime`** (meme base) :
   - `LABEL org.opencontainers.image.source` pointant vers le depot GitHub.
   - Copie les artefacts du stage de build uniquement (pas de compilateur ni devDependencies).
   - Cree un utilisateur non-root `app:1001:1001` via `adduser`/`addgroup`.
   - Variables d'environnement : `PORT=8080`, `LOG_LEVEL=info`, `NODE_ENV=production`.
   - `USER 1001:1001` avant le `CMD`.

### Tests locaux

```bash
docker build -t annuaire:test .
docker run -d -p 8080:8080 annuaire:test
curl http://localhost:8080/healthz    # => 200 OK
curl http://localhost:8080/students   # => JSON
```

- Taille de l'image : environ 132 MiB (sous la limite de 200 Mo).
- L'image est poussee sur GHCR : `ghcr.io/yannis-alouache/annuaire:<sha>`.

### Etat courant

- Dockerfile `annuaire` finalise en multi-stage.
- Utilisateur runtime non-root : `1001:1001`.
- Lockfile rendu portable via un `.npmrc` de projet pointant vers `https://registry.npmjs.org/`.
- Validation locale :
  - build Docker OK ;
  - `GET /healthz` OK ;
  - `GET /students` OK ;
  - taille image locale de validation : `138383801` octets (environ 132 MiB).

## 4. Chart Helm

### Structure

```
services/annuaire/chart/
├── Chart.yaml              # apiVersion v2, nom, version, maintainer
├── values.yaml             # valeurs par defaut (repliques, image, probes, resources)
├── values-dev.yaml         # surcharges pour dev (ingress active, 1 replica, LOG_LEVEL=debug)
├── values-staging.yaml     # surcharges pour staging (ingress actif, host different)
├── values-preview.yaml     # surcharges pour previews (1 replica, resources reduits, host injecte par ApplicationSet)
└── templates/
    ├── _helpers.tpl        # helpers : annuaire.name, fullname, labels (4 labels obligatoires)
    ├── configmap.yaml      # ConfigMap pour LOG_LEVEL (avec checksum de deploiement)
    ├── deployment.yaml     # Deployment avec probes, securityContext, resources
    ├── ingress.yaml        # Ingress conditionnel (ingress.enabled)
    ├── presync-migration-job.yaml  # Job PreSync conditionnel (hooks.preSyncMigration.enabled)
    └── service.yaml        # Service ClusterIP sur le port 80
```

### Labels obligatoires

Le helper `annuaire.labels` injecte les 4 labels recommandes :

```yaml
app.kubernetes.io/name: annuaire
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/part-of: devhub-campus
app.kubernetes.io/managed-by: Helm
```

### Probes

Le Deployment expose deux probes HTTP sur `/healthz` (port 8080) :
- **readinessProbe** : `initialDelaySeconds: 2`, `periodSeconds: 5` — le pod est pret a recevoir du trafic rapidement.
- **livenessProbe** : `initialDelaySeconds: 10`, `periodSeconds: 10` — redemarrage automatique si le service devient instable.

### Ingress

L'Ingress est conditionnel (`ingress.enabled`). En dev, il est active avec le host `annuaire.devhub.local`. En preview, le host est surcharge dynamiquement par l'ApplicationSet (ex: `annuaire-pr-1.devhub.local`).

### Differences entre environnements

| Parametre | Defaut (`values.yaml`) | Dev | Preview | Staging |
| --- | --- | --- | --- | --- |
| `replicaCount` | 2 | 1 | 1 | 2 (defaut) |
| `ingress.enabled` | false | true | true | true |
| `ingress.host` | annuaire.devhub.local | annuaire.devhub.local | injecte par ApplicationSet | annuaire.staging.devhub.local |
| `env.LOG_LEVEL` | info | debug | debug | info (defaut) |
| `resources.requests.cpu` | 50m | 50m (defaut) | 25m | 50m (defaut) |
| `resources.limits.memory` | 128Mi | 128Mi (defaut) | 96Mi | 128Mi (defaut) |
| `hooks.preSyncMigration.enabled` | false | true | false | false |

## 5. Installation ArgoCD et premiere `Application`

### Installation

- ingress-nginx installe via Helm sur le cluster `devhub`.
- ArgoCD installe via le chart officiel `argo/argo-cd` avec `helm install argocd argo/argo-cd -n argocd -f platform/argocd/values.yaml`.
- Tous les pods du namespace `argocd` sont en `Running` (controller, server, repo-server, redis, dex, notifications, applicationset-controller).
- Ingress ArgoCD expose sur `argocd.devhub.local` (TLS desactive en interne, ingress-nginx en frontal).
- Verification HTTP effectuee via `curl -H "Host: argocd.devhub.local" http://localhost/`.
- Mot de passe initial admin recupere depuis le Secret `argocd-initial-admin-secret`.

### Premiere Application

- Depot GitHub public du projet : `https://github.com/Yannis-Alouache/devhub-campus`.
- `AppProject` `devhub` cree avec comme source autorisee le repo GitHub et comme destinations `devhub-*` + `argocd`.
- `Application` `annuaire-dev` creee d'abord en mode manuel, synchronisee, puis basculee en auto-sync avec `selfHeal: true` et `prune: false`.
- Etat constate : `Synced + Healthy`.

### Comparaison `selfHeal` vs `prune`

| Option | Effet | Exemple dangereux |
| --- | --- | --- |
| `selfHeal: true` | ArgoCD re-synchronise automatiquement quand un drift est detecte (une ressource cluster ne correspond plus au manifest Git). | Un operateur fait un `kubectl scale deploy X --replicas=10` pour absorber un pic de charge. ArgoCD annule le scaling en quelques secondes, ce qui provoque une chute de capacite en pleine crise. |
| `prune: true` | ArgoCD supprime du cluster les ressources qui ne sont plus dans Git. | Quelqu'un supprime accidentellement un fichier `service.yaml` du chart dans Git (ou merge une branche qui le retire). ArgoCD supprime le Service du cluster, coupant l'acces a l'application sans qu'aucune commande Kubernetes n'ait ete lancee. |

En dev, `selfHeal: true` est active (correction du drift) mais `prune: false` (pas de suppression automatique). En preview, `prune: true` est necessaire pour que les environnements temporaires soient nettoyes quand la PR est fermee.

## 6. Pattern App of Apps

### Principe

La `root` Application (definie dans `platform/bootstrap/root-app.yaml`) surveille le dossier `platform/apps/` dans Git. Ce dossier contient les manifests des `Application` enfants (`annuaire-dev`, `planning-dev`, `notif-dev`) et des `ApplicationSet` preview. Quand une nouvelle `Application` apparait dans ce dossier Git, ArgoCD la cree automatiquement dans le cluster. Le bootstrap initial se resume a :

```bash
kubectl apply -f platform/bootstrap/root-app.yaml
```

Apres cette commande, ArgoCD prend le relais et cree les applications enfants.

### Etat courant

- `root` creee depuis `platform/bootstrap/root-app.yaml`.
- `AppProject` `devhub` ajuste pour autoriser `devhub-*` et `argocd`.
- Trois applications enfants presentes sous ArgoCD :
  - `annuaire-dev`
  - `planning-dev`
  - `notif-dev`
- Etat constate : `root`, `annuaire-dev`, `planning-dev` et `notif-dev` en `Synced + Healthy`.
- Verification HTTP faite via les ingresses locaux avec header `Host` :
  - `annuaire.devhub.local`
  - `planning.devhub.local`
  - `notif.devhub.local`

### Pourquoi App of Apps n'est pas equivalent a `kubectl apply -f apps/dev/` ?

Un `kubectl apply -f apps/dev/` applique les manifests une seule fois, de maniere imperative. Si quelqu'un modifie le cluster ensuite (drift), rien ne se passe. Si un nouveau fichier est ajoute dans le dossier Git, il faut relancer manuellement la commande.

Avec le pattern App of Apps, la `root` Application est en auto-sync : ArgoCD surveille en permanence le dossier Git. Si un nouveau manifest `Application` est ajoute, il est detecte et applique automatiquement. Si un manifest est modifie, la modification est propagee. Si une application est supprimee de Git, elle est supprimee du cluster. Le cycle de vie complet est gere de maniere declarative, sans intervention manuelle.

## 7. `ApplicationSet` et previews

### Choix du generateur et justification

Le polycopie demande de choisir entre le generateur `git` (branches) et `pullRequest` (PRs ouvertes).

- **`git`** : detecte les branches qui matchent un pattern et genere une `Application` par branche. Plus simple conceptuellement, mais ne distingue pas les branches "en cours de revue" des branches mortes. De plus, la version d'ArgoCD installee (v2.12.6) ne supporte pas `git.branches` pour les sous-chemins.
- **`pullRequest`** : detecte les PR ouvertes via l'API GitHub et genere une `Application` par PR. Plus precis (seules les PR actives sont prises en compte), mais requiert un token GitHub pour eviter le rate limiting.

**Decision : `pullRequest`**. Le repo est heberge sur un compte utilisateur GitHub (`Yannis-Alouache`), pas dans une organisation. Le generateur `scmProvider.github` avec `allBranches` ne fonctionne que pour les organisations. Le generateur `pullRequest` est donc la seule option viable pour ce repo. En pratique, meme avec un repo public, le rate limiting GitHub impose l'usage d'un `tokenRef`.

### Bilan du cycle demo

1. **Ouverture** : creation de la PR `#1` depuis `feature/demo-preview-annuaire` vers `main`. ArgoCD detecte la PR et genere trois `Application` preview (`annuaire-pr-1`, `planning-pr-1`, `notif-pr-1`) dans le namespace `devhub-preview-pr-1`.
2. **Convergence** : les trois applications passent en `Synced + Healthy`. L'ingress est accessible via les hosts `annuaire-pr-1.devhub.local`, etc.
3. **Fermeture** : la PR est fermee et la branche supprimee. Les trois `Application` preview sont prunees automatiquement par ArgoCD.
4. **Nettoyage** : le namespace `devhub-preview-pr-1` reste vide mais n'est pas supprime automatiquement (ArgoCD ne gere le cycle de vie que des ressources qu'il a creees, pas du namespace lui-meme). Suppression manuelle avec `kubectl delete namespace`.

### Demo realisee

- PR de demonstration : `#1` — `feature/demo-preview-annuaire` -> `main`
- Effet volontaire sur la branche : `services/annuaire/chart/values-preview.yaml` passe `replicaCount` de `1` a `2`
- Build associe : workflow GitHub Actions `build-images` execute avec succes avant la stabilisation des previews
- Applications generees par ArgoCD :
  - `annuaire-pr-1`
  - `planning-pr-1`
  - `notif-pr-1`
- Namespace cree : `devhub-preview-pr-1`
- Hosts verifies :
  - `annuaire-pr-1.devhub.local`
  - `planning-pr-1.devhub.local`
  - `notif-pr-1.devhub.local`

### Etat courant

- Le poly a bien ete relu : il impose de choisir et justifier un generateur, mais n'impose pas une option unique.
- Limitation constatee sur la version d'ArgoCD installee :
  - la voie `git.branches` n'est pas exploitable ici ;
  - `scmProvider.github allBranches` scanne des organisations GitHub, pas un compte utilisateur comme `Yannis-Alouache`.
- Strategie retenue :
  - `platform/apps/preview/*.yaml` utilise maintenant `pullRequest.github.owner/repo` sur `Yannis-Alouache/devhub-campus` ;
  - le poly conseille de preparer un token GitHub ; la doc ApplicationSet autorise un repo public sans `tokenRef`, mais dans notre cas le controleur a effectivement rencontre un `403 API rate limit exceeded`, donc `tokenRef` devient necessaire en pratique ;
  - les previews sont filtrees sur les PR dont la branche source matche `^feature/.*` et la branche cible `^main$` ;
  - chaque preview partage le namespace `devhub-preview-pr-<numero>` et expose des hosts dedies (`annuaire-pr-<numero>.devhub.local`, etc.) ;
  - le tag d'image injecte correspond a `head_short_sha_7`, ce qui colle a la CI GitHub Actions deja en place.
- Cablage plateforme :
  - `platform/bootstrap/root-app.yaml` doit maintenant charger `platform/apps/` et non plus seulement `platform/apps/dev/`, afin que les `ApplicationSet` preview soient crees par la root app.
- Secret externe au Git :
  - un `Secret` Kubernetes `github-token` dans `argocd` fournit le PAT au generateur `pullRequest` ;
  - le token n'est jamais commite dans le repo.
- Validation faite :
  - `kubectl apply --dry-run=server -f platform/apps/preview` passe ;
  - `kubectl apply --dry-run=server -f platform/bootstrap/root-app.yaml` passe ;
  - une application temporaire des trois `ApplicationSet` sur le cluster a confirme des conditions `ParametersGenerated=True` et `ResourcesUpToDate=True` ;
  - une fois la PR `#1` ouverte, ArgoCD a genere `annuaire-pr-1`, `planning-pr-1` et `notif-pr-1` dans `devhub-preview-pr-1` ;
  - `annuaire-pr-1` a bien deploye `2` replicas comme attendu ;
  - les trois applications de preview sont passees en `Synced + Healthy` ;
  - les endpoints `/healthz` ont repondu via `curl` avec le header `Host` sur les trois hosts preview ;
  - a la fermeture de la PR `#1` et a la suppression de la branche, les trois `Application` preview ont ete prunees automatiquement ;
  - le namespace partage `devhub-preview-pr-1` est reste vide apres le prune et a ete supprime manuellement ; avec `CreateNamespace=true`, ce point reste a connaitre/documenter.

## 8. Bestiaire ArgoCD

Pour chaque scenario : capture, observation, hypothese, conclusion.

### 1. Drift sur `replicaCount`

- **Manipulation.** `kubectl scale deploy annuaire-dev-annuaire -n devhub-dev --replicas=7`, puis refresh force de l'application pour voir passer l'etat dans ArgoCD.
- **Observation.** A `t+1s`, `annuaire-dev` passe en `OutOfSync + Progressing` avec `replicas=7/1`. A `t+3s`, ArgoCD a deja reapplique le desired state Git et le `Deployment` revient a `1/1`, avec l'operation auto-sync en succes.
- **Hypothese.** Comme `selfHeal: true` est active, ArgoCD ne se contente pas de signaler le drift : il re-synchronise aussitot la ressource modifiee a la main.
- **Verification.** Le `Deployment` repasse de `spec.replicas=7` a `1`, et `annuaire-dev` revient en `Synced + Healthy` sans aucun commit Git.
- **Conclusion.** Un changement manuel dans le cluster ne "gagne" pas contre Git. En pratique, sur cette app, la correction est quasi immediate.

### 2. Tag image inexistant

- **Manipulation.** Commit `26dc5aa` sur `main`, en forcant `image.tag=does-not-exist-step8` dans `platform/apps/dev/annuaire.yaml`.
- **Observation.** ArgoCD accepte la sync du manifeste et cree un nouveau ReplicaSet avec l'image invalide. Le pod associe passe en `ErrImagePull` puis `ImagePullBackOff`. Avec la strategie de rollout par defaut, l'ancien pod sain reste toutefois en place, ce qui laisse l'application en `Synced + Progressing` au lieu d'un `Degraded` immediat.
- **Hypothese.** ArgoCD considere le rendu Git comme valide et synchronise correctement les manifests ; l'echec apparait seulement a l'execution, au niveau kubelet / registry.
- **Verification.** Le `Deployment` pointe bien vers `ghcr.io/yannis-alouache/annuaire:does-not-exist-step8`, et Kubernetes cree un nouveau ReplicaSet en echec pendant que l'ancien sert encore le trafic.
- **Conclusion.** `Synced` ne veut pas dire "deploie avec succes". Si une image n'existe pas, il faut regarder le `Deployment`, les `ReplicaSet` et les events de pod, pas seulement le badge ArgoCD.

### 3. Rollback par `git revert`

- **Manipulation.** `git revert` du commit precedent, produit dans le commit `69abbf4`.
- **Observation.** A `t+0s`, `annuaire-dev` est encore `Progressing` avec l'image invalide. A `t+6s`, l'image revient sur `ccb0f03`, le ReplicaSet fautif disparait et l'application repasse en `Synced + Healthy`.
- **Hypothese.** Un revert Git suffit : ArgoCD voit le nouveau commit, resynchronise automatiquement et le cluster converge sans action imperative supplementaire.
- **Verification.** Le `Deployment` retrouve `ghcr.io/yannis-alouache/annuaire:ccb0f03` et un seul pod sain reste en place.
- **Conclusion.** Le rollback GitOps tient bien dans Git lui-meme. Ici, revenir a l'etat sain a pris environ 6 secondes.

### 4. Hook `PreSync`

- **Manipulation.** Commit `beadd30` sur `main`, ajoutant un job `annuaire-dev-annuaire-migration` avec `argocd.argoproj.io/hook: PreSync`.
- **Observation.** Au sync suivant, le job de migration apparait avant le deploiement principal, s'execute en 6 secondes, puis logge `migration ok`. L'application ne revient `Healthy` qu'apres ce job.
- **Hypothese.** Un hook `PreSync` bloque la sync tant qu'il n'a pas reussi ; c'est le bon mecanisme pour une migration ou un pre-check obligatoire.
- **Verification.** Le `syncResult` de l'application liste d'abord le `Job` en phase `PreSync`, puis les autres ressources. Les logs du job contiennent bien `migration ok`.
- **Conclusion.** Pour une migration de schema ou un pre-requis fort, `PreSync` donne un vrai garde-fou : si le job echoue, le deploiement applicatif n'avance pas.

### 5. Sync waves

- **Manipulation.** Le meme commit `beadd30` introduit un `ConfigMap` annote `argocd.argoproj.io/sync-wave: "-1"` et un `Deployment` annote `argocd.argoproj.io/sync-wave: "0"`. Ensuite, le commit `fb68e13` retire la cle `LOG_LEVEL` du `ConfigMap` (`data: {}`) et force un rollout visible avec `maxUnavailable: 1` et `maxSurge: 0`.
- **Observation.** Le `syncResult` ArgoCD applique d'abord le `Job` `PreSync`, puis le `ConfigMap`, puis le `Deployment`. Quand le `ConfigMap` ne contient plus `LOG_LEVEL`, le pod applicatif passe en `CreateContainerConfigError` avec l'event Kubernetes `couldn't find key LOG_LEVEL in ConfigMap`. ArgoCD reste `Synced + Progressing` tant que le deadline de rollout n'est pas depasse, mais le deploiement, lui, ne demarre plus.
- **Hypothese.** Les sync waves controlent bien l'ordre d'application, mais pas la validite semantique de ce qui est applique : si la ressource de wave `-1` est mauvaise, la wave `0` la consomme et echoue.
- **Verification.** Le `ConfigMap` rendu en cluster porte bien la wave `-1`, le `Deployment` la wave `0`, et le pod `annuaire-dev-annuaire-75447b79fc-bgf9n` reste bloque en `CreateContainerConfigError`.
- **Conclusion.** Les sync waves permettent d'imposer l'ordre, pas de "magiquement" securiser les ressources. Une config invalide placee tot dans la sequence casse tout ce qui arrive apres.

### 6. Suppression par `prune`

- **Manipulation.** Commit `3cc2f3a` sur `main`, avec `prune: true` active pour `annuaire-dev` et suppression du fichier `services/annuaire/chart/templates/service.yaml`.
- **Observation.** Le `Service/annuaire-dev-annuaire` disparait du cluster a `t+15s`. L'application reste `Healthy`, car le `Deployment` et l'`Ingress` existent toujours, meme si l'ingress n'a plus de backend cible operationnel. Le commit de revert `8fa80f8` recree ensuite le `Service` et ramene l'app a un etat normal en 16 secondes.
- **Hypothese.** Avec `prune: true`, ArgoCD supprime bien du cluster les ressources retirees du Git, meme si l'on n'a pas touche au cluster a la main.
- **Verification.** Le `Service` est visible avant le commit, absent apres la sync, puis de nouveau present apres le revert.
- **Conclusion.** `prune` est tres puissant et donc dangereux : une suppression de fichier dans Git se traduit par une vraie suppression Kubernetes.

## 9. Securite et observabilite d'ArgoCD

### RBAC

#### Configuration appliquee

Le fichier `platform/argocd/values.yaml` declare :

- un compte local `developer` avec `accounts.developer: login, apiKey` dans le ConfigMap `argocd-cm` ;
- un role `platform-admin` avec tous les droits (`p, role:platform-admin, *, *, *, allow`) ;
- un role `developer` borne au service annuaire :
  - `get` sur toutes les `Application` du projet `devhub` ;
  - `get` et `logs` sur le projet `devhub` ;
  - `sync` uniquement sur `devhub/annuaire*` (syntaxe glob, pas regex POSIX) ;
- les mappings `g, admin, role:platform-admin` et `g, developer, role:developer` ;
- `policy.default: role:authenticated` pour que tout utilisateur authentifie puisse au moins lire les apps ;
- `policy.matchMode: glob` (defaut ArgoCD).

#### Tests realises

1. **Creation du mot de passe.** Le mot de passe du compte `developer` a ete defini via `argocd account update-password --account developer` (execute dans le pod `argocd-server`).
2. **Login developer.** Connexion reussie via l'API (`POST /api/v1/session` avec `username: developer`).
3. **Lecture autorisee.** `GET /api/v1/applications` en tant que `developer` retourne les 4 applications (`annuaire-dev`, `notif-dev`, `planning-dev`, `root`).
4. **Sync autorisee.** `POST /api/v1/applications/annuaire-dev/sync` en tant que `developer` est acceptee — la sync s'execute (elle echoue ensuite a cause du Job PreSync residual de l'etape 8, mais l'autorisation RBAC est bien accordee).
5. **Sync refusee.** `POST /api/v1/applications/planning-dev/sync` en tant que `developer` retourne :
   ```json
   {"error":"permission denied: applications, sync, devhub/planning-dev, sub: developer, iat: 2026-05-28T23:27:42Z","code":7}
   ```
   Le role `developer` ne peut pas `sync` une application qui ne matche pas `devhub/annuaire*`.

#### Points de vigilance

- Le ConfigMap `argocd-rbac-cm` utilise `glob` par defaut : `annuaire*` matche `annuaire-dev`, `annuaire-pr-1`, etc., mais pas `planning-dev`.
- Le nom du champ `policy.csv` dans le ConfigMap doit etre exactement `policy.csv` (avec le pipe `|` pour le bloc YAML).
- Le role `role:authenticated` donne un droit de lecture par defaut a tous les utilisateurs connectes.

### Notifications

#### Configuration appliquee

Le chart active `notifications.enabled: true` dans `platform/argocd/values.yaml`. Le ConfigMap `argocd-notifications-cm` contient :

- un service webhook `webhook-site` pointant vers `https://webhook.site/7950c1c3-6599-400f-9bd2-56e27479e7e8` ;
- un template `sync-failed-webhook` qui envoie un JSON avec `application`, `revision` et `error` ;
- un trigger `on-sync-failed-webhook` qui se declenche quand `app.status.operationState.phase` est `Error` ou `Failed` ;
- l'annotation `notifications.argoproj.io/subscribe.on-sync-failed-webhook.webhook-site: ""` posee sur `annuaire-dev` pour souscrire aux notifications.

#### Test realise

Le test RBAC ci-dessus a declenche un `sync` sur `annuaire-dev` qui a echoue (Job PreSync residual `annuaire-dev-annuaire-migration` en `BackoffLimitExceeded`). Le notifications controller a detecte l'echec et envoye la notification :

```
level=info msg="Trigger on-sync-failed-webhook result: [{... true}]"
level=info msg="Sending notification about condition 'on-sync-failed-webhook.[0].H9WjsqG1dKYm6njOZ7yUQYOA1Wk' to '{webhook-site }'"
DEBUG POST https://webhook.site/7950c1c3-6599-400f-9bd2-56e27479e7e8
```

Le payload envoye contient :
```json
{
  "application": "annuaire-dev",
  "revision": "<sha-du-commit>",
  "error": "one or more synchronization tasks completed unsuccessfully"
}
```

La notification a ete recue avec succes par webhook.site (HTTP 200 du POST). L'annotation `notified.notifications.argoproj.io` sur l'Application empeche les envois en doublon tant que l'etat ne change pas.

### Metriques Prometheus

Les trois Services suivants exposent les metriques d'ArggoCD :

| Service | Port | Composant |
| --- | --- | --- |
| `argocd-application-controller-metrics` | 8082 | Controleur de reconciliation |
| `argocd-server-metrics` | 8083 | API server |
| `argocd-repo-server-metrics` | 8084 | Repo server (git fetch, helm template) |

Verifie via `kubectl port-forward` + `curl /metrics`.

#### Trois metriques utiles

1. **`argocd_app_info`** (gauge, sans unite — valeur toujours 1)
   - Labels : `sync_status`, `health_status`, `name`, `project`, `dest_namespace`.
   - **Interpretation en incident :** permet de savoir combien d'applications sont `OutOfSync` ou `Degraded`. Si `sync_status="OutOfSync"` passe de 0 a N, le controleur n'arrive plus a reconcilier ou le repo Git est injoignable.
   - Requete typique : `count(argocd_app_info{sync_status="OutOfSync"})`.

2. **`argocd_app_sync_total`** (counter, nombre de syncs)
   - Labels : `name`, `phase` (`Succeeded`, `Failed`, `Error`), `project`.
   - **Interpretation en incident :** `rate(argocd_app_sync_total{phase="Failed"}[5m]) > 0` signale des syncs qui echouent en continu — image introuvable, manifest invalide, ou droit manque. C'est la metrique qui justifie a elle seule les notifications.
   - Requete typique : `increase(argocd_app_sync_total{phase="Failed"}[1h])`.

3. **`argocd_git_request_duration_seconds`** (histogram, secondes)
   - Labels : `repo`, `request_type` (`fetch`, `ls-remote`).
   - **Interpretation en incident :** si le 99e centile de `argocd_git_request_duration_seconds{request_type="ls-remote"}` depasse quelques secondes, le repo Git est lent ou le reseau est degrade — ArgoCD mettra plus de temps a detecter les changements et les syncs seront retardes.
   - Requete typique : `histogram_quantile(0.99, rate(argocd_git_request_duration_seconds_bucket[5m]))`.

## 11. Synthese obligatoire

### Retrospective TP 1 -> TP 2

| Operation du quotidien | Vecu reel avec ArgoCD |
| --- | --- |
| Deployer un service pour la premiere fois | Plus long au demarrage (installer ArgoCD, creer l'AppProject, la root Application), mais une fois le socle en place, deployer un 2e ou 3e service se reduit a un commit dans `platform/apps/dev/`. |
| Deployer une nouvelle version | Totalement transparent : le CI build l'image, pousse le tag, le commit qui met a jour `image.tag` dans Git suffit. Pas de pipeline de deploiement a maintenir. |
| Faire un rollback | `git revert` + 6 secondes de convergence. Pas besoin de relancer la CI ni de chercher l'ancien tag. C'est l'operation ou le gain est le plus flagrant. |
| Ouvrir un environnement de plus | Ajouter une `Application` dans le dossier Git. Pas de kubeconfig supplementaire, pas de pipeline dedie. |
| Donner un env perso a chaque dev | Les previews par PR ont fonctionne immediatement. Un dev ouvre une PR, un namespace isole apparait, il peut tester sans impacter les autres. |
| Voir ce qui tourne en ce moment | L'UI ArgoCD montre l'etat de chaque application en un coup d'oeil. Pas besoin de `kubectl get all -A`. |
| Detecter qu'un dev a fait un `kubectl edit` en douce | ArgoCD passe en `OutOfSync` en moins de 3 secondes et re-synchronise avec `selfHeal`. Le drift est corrige avant meme que le dev ait ferme son terminal. |
| Auto-reparer un drift | `selfHeal: true` fait exactement ce qu'on attend. Le test `kubectl scale` a ete corrige en 3 secondes. |
| Donner les droits a un nouveau dev | Le RBAC ArgoCD fonctionne mais la configuration est plus verbeuse qu'un kubeconfig. Le mini-DSL CSV du ConfigMap `argocd-rbac-cm` necessite une relecture de la doc. |
| Hotfix en urgence a 3h du matin | Pas besoin de SSH ni de `kubectl edit`. Une PR sur le repo, un merge, et ArgoCD converge. L'operation est tracee dans Git. |
| Auditer ce qui a change sur 6 mois | `git log` sur le repo `platform/`. Chaque changement de manifest est un commit avec auteur, date et message. |
| Re-deployer le cluster from scratch | `kind create cluster` + `helm install argocd` + `kubectl apply -f root-app.yaml`. Tout le reste converge automatiquement. |
| Desinstaller un service | Supprimer le fichier dans Git, `prune: true` fait le reste. Mais `prune` est dangereux — on l'a vu quand le `Service` a disparu apres la suppression de `service.yaml`. |
| Tester un changement risque | Les previews isolees par branche sont un confort considerable. On teste sur `annuaire-pr-1.devhub.local` sans toucher au dev partagé. |

#### Operations ou ArgoCD est plus contraignant

1. **Depliage initial d'un service** — Au TP 1, `kubectl apply -f` est immediat. Avec ArgoCD, il faut installer ArgoCD, configurer l'AppProject, creer la root Application, puis attendre la detection. Cette contrainte est toutefois justifiee car elle force a structurer la plateforme et a rendre le deploiement reproductible.

2. **Donner les droits a un nouveau dev** — Au TP 1, on distribue un kubeconfig. Avec ArgoCD, il faut creer un compte local, configurer le RBAC dans le ConfigMap avec la syntaxe CSV, et potentiellement redemarrer le serveur. Cette contrainte est justifiee car elle evite de donner des droits cluster-wide a un developpeur qui n'en a pas besoin.

3. **Correction d'un pic de charge** — Si un operateur fait `kubectl scale --replicas=10` pour absorber un pic, `selfHeal` annule le changement en quelques secondes. ArgoCD ne distingue pas un "bon" drift d'un "mauvais" drift. Cette contrainte est partiellement justifiee (le scaling devrait etre declare dans Git), mais en situation d'urgence, il faut desactiver temporairement le self-heal ou utiliser une ressource `HorizontalPodAutoscaler`.

#### L'operation qui justifie l'adoption d'ArgoCD

**La detection et correction automatique du drift** (`selfHeal`). En production, un `kubectl edit` fait en astreinte a 3h du matin derive le cluster de son etat souhaite sans que personne ne le sache. Avec ArgoCD, le drift est detecte en quelques secondes et corrige automatiquement. Cette seule capacite — garantir que le cluster correspond toujours a ce qui est dans Git — justifie l'adoption.

### Ce qu'ArgoCD ne sait pas faire

#### 1. Deploiement progressif (canary, blue/green)

**Risque concret :** ArgoCD applique les manifests de maniere atomique. Une mise a jour du Deployment declenche un `RollingUpdate` Kubernetes natif (maxSurge/maxUnavailable), mais sans analysis de metriques en temps reel. Si la nouvelle version est bogueuse, elle sera progressivement deployee a 100% du trafic sans interruption automatique. En prod, cela signifie qu'un deploiement defaillant touche tous les utilisateurs avant que quelqu'un intervienne.

**Outil complementaire :** Argo Rollouts (du meme ecosysteme). Il remplace le `Deployment` par un `Rollout` et permet de definir des etapes (canary 10% -> 20% -> 100%) avec des analyses basees sur Prometheus, les metriques de la pod health, ou des webhooks externes.

**Reference :** https://argoproj.github.io/argo-rollouts/

#### 2. Validation des manifests avant sync

**Risque concret :** ArgoCD sync tout manifest qui est dans Git, meme s'il contient des erreurs (mauvaise image, resource requests absentes, label manquant). Le test `image.tag=does-not-exist` de l'etape 8 l'a montre : ArgoCD passe en `Synced` mais le pod est en `ImagePullBackOff`. En prod, un manifest invalide peut rendre un service indisponible.

**Outil complementaire :** Kyverno (AdmissionPolicy) ou OPA Gatekeeper. Ils agissent comme des webhooks d'admission Kubernetes et rejettent les ressources qui violent les politiques (pas de `:latest`, ressources obligatoires, labels requis). Conftest peut aussi valider les manifests en CI avant le push.

**Reference :** https://kyverno.io/docs/ et https://www.openpolicyagent.org/docs/v1.0/kubernetes/

#### 3. Gestion des secrets dans Git

**Risque concret :** Les manifests du repo `platform/` contiennent des references a des secrets (mot de passe admin ArgoCD, token GitHub). Actuellement, le token GitHub est dans un Secret Kubernetes hors Git, ce qui n'est pas reproductible. Si le cluster est detruit, le secret doit etre recree manuellement. Un developpeur pourrait accidentellement commiter un secret en clair.

**Outil complementaire :** Sealed Secrets (Bitnami) ou External Secrets Operator. Sealed Secrets chiffre le secret avec une cle publique et le commit dans Git. Seul le controleur dans le cluster peut le dechiffrer. External Secrets Operator synchronise les secrets depuis un gestionnaire externe (AWS Secrets Manager, Vault).

**Reference :** https://sealed-secrets.netlify.app/ et https://external-secrets.io/

#### 4. Signature et provenance des images

**Risque concret :** N'importe quelle image peut etre deployee tant qu'elle existe dans le registry. Si le registry GHCR est compromis ou si une image malveillante est poussee avec un tag existant, ArgoCD la deployera sans verification. Le TP a montre qu'un `does-not-exist` passe le sync — a l'inverse, une image malveillante bien presente sera deployee silencieusement.

**Outil complementaire :** Sigstore/cosign pour signer les images a la build et une admission policy (Kyverno) pour verifier la signature avant le deploiement. Le workflow CI (`build.yml`) inclus deja un placeholder cosign dans le squelette.

**Reference :** https://docs.sigstore.dev/ et https://kyverno.io/policies/other/verify_image/

#### 5. RBAC multi-equipe sur ArgoCD

**Risque concret :** Le RBAC actuel utilise des comptes locaux et une syntaxe CSV dans un ConfigMap. Pour 3-4 comptes, ca fonctionne. Pour 50 developpeurs dans 5 equipes, maintenir le fichier `policy.csv` a la main est ingérable. De plus, les comptes locaux n'ont pas de SSO, pas de rotation de mot de passe, pas d'audit de connexion.

**Outil complementaire :** Dex (deja inclus dans le chart ArgoCD) ou Keycloak pour l'authentification OIDC/SSO. Les groupes OIDC mappent vers les roles RBAC ArgoCD. Pour le TP, Dex en mode statique avec deux utilisateurs aurait suffi.

**Reference :** https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/

#### 6. Disaster recovery applicatif

**Risque concret :** ArgoCD sait recreer les ressources Kubernetes depuis Git, mais il ne sauvegarde pas les donnees persistantes. Si un PVC est supprime ou corrompu, les donnees sont perdues. Les ConfigMaps et Secrets hors Git (comme le token GitHub) ne sont pas non plus sauvegardes.

**Outil complementaire :** Velero pour les snapshots de ressources Kubernetes et de volumes PVC. Pour les bases de donnees, des dumps reguliers vers un stockage objet (S3, MinIO). Velero peut aussi migrer un cluster vers un autre.

**Reference :** https://velero.io/docs/

#### 7. Multi-cluster

**Risque concret :** L'architecture actuelle est mono-cluster (`kind-devhub`). En prod, on aurait au minimum un cluster `dev` et un cluster `staging`/`prod`. ArgoCD peut gerer plusieurs clusters, mais il faut les enregistrer, gerer les kubeconfigs, et adapter les `Application` pour pointer vers les bons clusters. Le pattern `ApplicationSet` avec le generateur `cluster` est fait pour ca.

**Outil complementaire :** ArgoCD supporte nativement le multi-cluster via le `cluster generator` d'ApplicationSet. Pour la gestion du cycle de vie des clusters eux-memes, Cluster API (CAPI) ou Crossplane completent ArgoCD.

**Reference :** https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-management/ et https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster/
