# RAPPORT — TP 2 GitOps avec ArgoCD : DevHub Campus

## Etat de redaction

- [ ] 0. Outillage
- [ ] 1. GitOps en 1 page
- [ ] 2. Glossaire ArgoCD
- [ ] 3. Service choisi et containerisation
- [ ] 4. Chart Helm
- [ ] 5. Installation ArgoCD et premiere `Application`
- [ ] 6. Pattern App of Apps
- [ ] 7. `ApplicationSet` et previews
- [ ] 8. Bestiaire ArgoCD
- [ ] 9. Securite et observabilite d'ArgoCD
- [ ] 11. Synthese obligatoire

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

- `docker version` :
- `kubectl version --client` :
- `helm version` :
- `argocd version --client` :

### Notes d'installation

- A completer pendant la mise en place du poste.

## 1. GitOps en 1 page

### Schema personnel

- A completer.

### Tableau push vs pull

| Question | Push (`kubectl apply` en CI) | Pull (ArgoCD) |
| --- | --- | --- |
| Qui a les droits sur le cluster ? | A completer | A completer |
| Ou est l'historique des changements ? | A completer | A completer |
| Que se passe-t-il si un dev modifie le cluster a la main ? | A completer | A completer |
| Comment ajouter un environnement de plus ? | A completer | A completer |
| Comment faire un rollback ? | A completer | A completer |
| Combien de pipelines pour 30 services ? | A completer | A completer |
| Qui voit en direct ce qui tourne ? | A completer | A completer |

### Prise de position

- A completer.

## 2. Glossaire ArgoCD

| Terme | Definition avec mes mots | Exemple dans mon projet |
| --- | --- | --- |
| `Application` | A completer | A completer |
| `AppProject` | A completer | A completer |
| `Source` | A completer | A completer |
| `Destination` | A completer | A completer |
| `Sync` | A completer | A completer |
| `Prune` | A completer | A completer |
| `App of Apps` | A completer | A completer |
| `ApplicationSet` | A completer | A completer |
| `Sync wave` | A completer | A completer |
| `Hook` | A completer | A completer |

## 3. Service choisi et containerisation

- Service prioritaire retenu : `annuaire`.
- Contraintes a documenter : multi-stage, non-root, tag SHA, endpoint `/healthz`, `LOG_LEVEL`, label OCI source.
- Resultats a consigner : image produite, commande de test locale, taille de l'image, remarques.

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

- Fichiers a decrire : `Chart.yaml`, `values.yaml`, `values-dev.yaml`, `values-staging.yaml`, `values-preview.yaml`, `templates/`.
- Points a documenter : labels obligatoires, probes, ingress, differences entre environnements.

## 5. Installation ArgoCD et premiere `Application`

- Notes a completer pendant l'etape 5.
- Captures a ajouter : UI ArgoCD, `Application` Healthy + Synced, arbre des ressources.
- Comparaison a rediger : `selfHeal: true` vs `prune: true` avec un exemple dangereux pour chacun.

### Etat courant

- ingress-nginx installe sur le cluster local.
- ArgoCD installe via le chart officiel `argo/argo-cd`.
- Tous les pods du namespace `argocd` sont en `Running`.
- Ingress ArgoCD expose sur `argocd.devhub.local`.
- Verification HTTP effectuee via `curl` avec header `Host`.
- Mot de passe initial admin recupere.
- Depot GitHub public du projet : `https://github.com/Yannis-Alouache/devhub-campus`.
- `AppProject` `devhub` cree.
- `Application` `annuaire-dev` creee d'abord en mode manuel, synchronisee, puis basculee en auto-sync avec `selfHeal: true` et `prune: false`.
- Etat constate : `Synced + Healthy`.

## 6. Pattern App of Apps

- Notes a completer pendant l'etape 6.
- Captures a ajouter : root `Application` + trois enfants.
- Question a traiter : pourquoi App of Apps n'est pas equivalent a un simple `kubectl apply -f apps/dev/` ?

### Etat courant

- `root` creee depuis `platform/bootstrap/root-app.yaml`.
- `AppProject` `devhub` ajuste pour autoriser `devhub-*` et `argocd`.
- Trois applications enfants presentes sous ArgoCD :
  - `annuaire-dev`
  - `planning-dev`
  - `notif-dev`
- Etat constate apres correction de l'erreur de comparaison ArgoCD : `root`, `annuaire-dev`, `planning-dev` et `notif-dev` sont en `Synced + Healthy`.
- Verification HTTP faite via les ingresses locaux avec header `Host` :
  - `annuaire.devhub.local`
  - `planning.devhub.local`
  - `notif.devhub.local`

## 7. `ApplicationSet` et previews

- Notes a completer pendant l'etape 7.
- Demo attendue : creation d'une PR issue de `feature/*`, apparition de la preview, suppression de la preview a la fermeture/suppression de la PR.
- Choix final du generateur : `pullRequest`. Le poly demande de choisir et justifier entre `git` et `pullRequest`, sans imposer une option unique. Ici, le repo est public mais heberge sur un compte utilisateur GitHub ; `pullRequest` est donc l'option supportee qui reste simple a exploiter.

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

- RBAC : A completer.
- Notifications : A completer.
- Metriques Prometheus :
  - metrique 1 :
  - metrique 2 :
  - metrique 3 :

## 11. Synthese obligatoire

### Retour TP 1 -> TP 2

- A completer.

### Ce qu'ArgoCD ne sait pas faire

| Theme | Risque concret | Outil complementaire | Reference |
| --- | --- | --- | --- |
| Deploiement progressif | A completer | A completer | A completer |
| Validation des manifests avant sync | A completer | A completer | A completer |
| Gestion des secrets dans Git | A completer | A completer | A completer |
| Signature et provenance des images | A completer | A completer | A completer |
| RBAC multi-equipe sur ArgoCD | A completer | A completer | A completer |
| Disaster recovery applicatif | A completer | A completer | A completer |
| Multi-cluster | A completer | A completer | A completer |
