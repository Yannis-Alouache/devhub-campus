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
- 2026-05-27 — Point restant pour cloturer l'etape 7 : ouvrir une PR `feature/*` vers `main`, montrer l'apparition de la preview dans ArgoCD, puis documenter la suppression et ajouter les captures.

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

### Etat courant

- Le poly a bien ete relu : il impose de choisir et justifier un generateur, mais n'impose pas une option unique.
- Limitation constatee sur la version d'ArgoCD installee :
  - la voie `git.branches` n'est pas exploitable ici ;
  - `scmProvider.github allBranches` scanne des organisations GitHub, pas un compte utilisateur comme `Yannis-Alouache`.
- Strategie retenue :
  - `platform/apps/preview/*.yaml` utilise maintenant `pullRequest.github.owner/repo` sur `Yannis-Alouache/devhub-campus` ;
  - le poly conseille de preparer un token GitHub ; la doc ApplicationSet indique toutefois que `tokenRef` reste optionnel sur un depot public, avec seulement une limite de requetes plus basse ;
  - les previews sont filtrees sur les PR dont la branche source matche `^feature/.*` et la branche cible `^main$` ;
  - chaque preview partage le namespace `devhub-preview-pr-<numero>` et expose des hosts dedies (`annuaire-pr-<numero>.devhub.local`, etc.) ;
  - le tag d'image injecte correspond a `head_short_sha_7`, ce qui colle a la CI GitHub Actions deja en place.
- Cablage plateforme :
  - `platform/bootstrap/root-app.yaml` doit maintenant charger `platform/apps/` et non plus seulement `platform/apps/dev/`, afin que les `ApplicationSet` preview soient crees par la root app.
- Validation faite :
  - `kubectl apply --dry-run=server -f platform/apps/preview` passe ;
  - `kubectl apply --dry-run=server -f platform/bootstrap/root-app.yaml` passe ;
  - une application temporaire des trois `ApplicationSet` sur le cluster a confirme des conditions `ParametersGenerated=True` et `ResourcesUpToDate=True` ;
  - aucune preview n'est generee tant qu'aucune PR `feature/*` n'est ouverte, ce qui est conforme au generateur choisi ;
  - les `ApplicationSet` temporaires ont ete supprimes juste apres le controle pour garder le cluster aligne avec Git en attendant le push des manifests ;
  - il reste a ouvrir une PR `feature/*` vers `main` pour produire la preuve finale dans l'UI ArgoCD et completer les captures.

## 8. Bestiaire ArgoCD

Pour chaque scenario : capture, observation, hypothese, conclusion.

1. Drift sur `replicaCount`
2. Tag image inexistant
3. Rollback par `git revert`
4. Hook `PreSync`
5. Sync waves
6. Suppression par `prune`

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
