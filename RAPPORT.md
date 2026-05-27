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

## 6. Pattern App of Apps

- Notes a completer pendant l'etape 6.
- Captures a ajouter : root `Application` + trois enfants.
- Question a traiter : pourquoi App of Apps n'est pas equivalent a un simple `kubectl apply -f apps/dev/` ?

## 7. `ApplicationSet` et previews

- Notes a completer pendant l'etape 7.
- Demo attendue : creation d'une branche `feature/*`, apparition de la preview, suppression de la preview a la suppression de la branche.
- Choix du generateur : A completer plus tard (`git` ou `pullRequest`).

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
