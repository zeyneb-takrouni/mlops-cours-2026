# Rapport — Lab 6 : CI/CD/CT de bout en bout

Binôme : Fatma Garali · Fork : `zeyneb-takrouni/mlops-cours-2026` · Dataset : german-credit ·
Image en prod à la fin du lab : `ghcr.io/zeyneb-takrouni/mlops-cours-2026/scoring-api:2.1.8` ·
`@champion` : v14

## 1. Architecture

    Poste (webhook CT)                      GitHub                         Cluster kind
    ------------------                      ------                         ------------
    Evidently (detect) --POST /drift-->  webhook (127.0.0.1:8095)
    webhook --crée un run--> KFP ct-pipeline --écrit--> MLflow (@challenger, @champion)
    webhook --export ONNX + git push--> main (packaging/model/)
                                          GitHub Actions : tests, build, smoke test
                                          --push image--> GHCR
                                          --commit du tag--> main (overlay staging)
    Argo CD (dans le cluster) --tire main toutes les 60 s--> staging (scoring-staging)
    webhook --gitops : canary 10 % puis 100 %--> main (overlay prod)
    Argo CD --tire main--> prod (scoring, canary KServe)
    Les nœuds kind --tirent l'image--> GHCR (package public)

Qui pousse : le webhook (modèle et prod, via Git), la CI (image sur GHCR, tag de staging).
Qui tire : Argo CD (Git), les nœuds (GHCR), le pipeline KFP (lot de données dans MLflow).
GitHub n'atteint jamais le cluster : tout passe par Git.

## 2. Temps du cycle

Démo de l'étape 9 (heures locales, source : `date`, `git log --format='%ci %s'`, UI KFP).

| Étape | Début | Fin | Durée |
|---|---|---|---|
| Dérive détectée → run KFP lancé | 10:43:58 | 10:44:07 | 9 s |
| Pipeline CT (ingest → gate) + export, jusqu'au commit du modèle | 10:44:07 | 10:48:23 | ≈ 4 min 16 s |
| Commit du modèle → image publiée (CI) et tag de staging | 10:48:23 | 10:50:12 | 1 min 49 s |
| Image publiée → staging servi et vérifié | 10:50:12 | 10:52:19 | 2 min 7 s |
| Canary 10 % → promotion 100 % | 10:52:19 | 10:53:10 | 51 s |
| **Cycle complet** | 10:43:58 | 10:53:10 | **9 min 12 s** |

La fin exacte du run KFP n'a pas été relevée (dernier log de la gate à 10:47:51) : la durée du
pipeline CT est approximative. Il faut ajouter jusqu'à 70 s pour que Argo CD applique la
promotion finale.

L'étape la plus longue est le pipeline CT (≈ 4 min), surtout du démarrage de pods, car le modèle
est petit. On pourrait la raccourcir avec des pods plus chauds ou une image plus légère. Le
délai Argo CD (≤ 70 s par sondage) se supprimerait avec un webhook GitHub → Argo CD, impossible
ici car GitHub ne voit pas le poste. Le premier tirage de l'image depuis GHCR a pris plus d'une
minute.

## 3. Gates et décisions

- Gate de données (premier essai, dérive ×2) : refusée. `duration` et `age` sortaient des
  bornes du contrat (2 contrôles sur 67). Le pipeline s'est arrêté avant tout entraînement.
- Gate de promotion, premier ré-entraînement (v13 contre v12) : `roc_auc` 0.8094 contre
  0.8050 (seuil 0.7), coût métier 108 contre 124. Promue.
- Gate de promotion, démo (v14 contre v13) : `roc_auc` 0.8094 pour les deux, coût 108 (règle
  `business_cost <= 108`). Promue par égalité : le v14 est identique au v13 (même lot, même code).
- Canary : 10 % de `2.1.8`, analyse naïve (50 prédictions sans erreur), puis promotion à 100 %.
  Aucun retour arrière nécessaire.

## 4. Points d'attention

- Webhook : jeton Bearer exigé (401 sans jeton, 503 s'il n'est pas défini côté serveur),
  stocké dans `.env` hors de Git, écoute sur 127.0.0.1 seulement, un seul run à la fois et
  10 min minimum entre deux.
- Gate automatique : elle bloque un modèle sous le seuil ou plus coûteux, mais pas un modèle
  identique : le v14 a été promu sans gain. Il faudrait exiger une amélioration minimale ou
  ne pas ré-entraîner si les données n'ont pas changé. Pour un modèle de crédit, une
  validation humaine (PR plutôt que commit direct) serait nécessaire.
- Retour arrière : trois niveaux, tous par commit. `gitops canary 0` coupe le trafic vers la
  nouvelle révision, `gitops rollback` restaure aussi le `@champion` (tag
  `previous_champion`), et `git revert` annule le commit de modèle.
- Secrets : le jeton du webhook est dans `.env` (poste), le `GITHUB_TOKEN` est géré par
  GitHub Actions. Le package GHCR est public, donc aucun identifiant de lecture dans le cluster.
- Incidents : KFP était à 0 réplique (remonté à la main) et les `pkill` des cibles `make`
  tuaient `make` lui-même.

## 5. Différences avec la prod

Secrets gérés par un coffre, bot dans le cluster avec un jeton limité au dépôt, PR relue
plutôt que commit direct sur `main`, environnements et clusters séparés, sauvegarde et
reprise après sinistre, audit des déploiements, analyse du canary sur de vraies métriques
(Lab 7), données réellement récentes et étiquetées au lieu d'un lot simulé, webhook GitHub →
Argo CD pour une synchronisation instantanée.

## 6. Captures d'écran

Fichiers dans `reports/captures/` :

- `argocd.png` : Applications `scoring-staging` et `scoring-prod` Synced / Healthy ;
- `kfp-gate.png` : run `ct-evidently-drift-20261117-104407` réussi, rapport de la gate ;
- `actions.png` : onglet Actions, run vert ;
- `canary.png` : `kubectl get isvc -n scoring` pendant le canary.
