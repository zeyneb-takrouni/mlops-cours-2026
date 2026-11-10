# Rapport Lab 5 : serving du modèle de scoring sur KServe

## Résumé

L'image FastAPI + ONNX Runtime du Lab 4 (`scoring-api`) est déployée comme InferenceService `scoring-model` dans le namespace `scoring`, en mode Knative (Knative Serving + Kourier). Le service est appelé depuis le poste via la passerelle `http://localhost:30080` avec l'en-tête `Host`. Il sert le champion MLflow version 12, avec le seuil métier 0,3182, identique à `decision_threshold` du run d'entraînement. L'autoscaler KPA pilote le nombre de pods sur le débit (cible 200 req/s par pod, 0 à 5 pods). Une version 2.0.0 a été déployée en canary 90/10, puis promue à 100 %.

## Benchmark de latence

Locust rejoue de vraies lignes de `data/processed/test.csv` (attente de 0,1 à 0,3 s entre deux requêtes par utilisateur).

| Run | Requêtes | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Erreurs (%) |
|---|---|---|---|---|---|---|
| run_100 | 54721 | 438.9 | 7 | 32 | 71 | 0.00 |
| run_500 | 131047 | 680.2 | 370 | 530 | 630 | 0.00 |
| run_1000 | 126802 | 648.3 | 880 | 1200 | 2100 | 0.00 |

Le run 1000 compte 3 erreurs HTTP 502 (Bad Gateway) sur 126 802 requêtes, soit environ 0,002 % : l'arrondi à deux décimales affiche 0.00.

**Point de saturation.** Le RPS maximal mesuré est d'environ 680 req/s avec 5 pods, soit environ 136 req/s par pod. À 500 utilisateurs, la charge théorique serait d'environ 2 200 req/s : le débit mesuré est très inférieur, donc le service est déjà saturé (les utilisateurs attendent leurs réponses). De 500 à 1000 utilisateurs, le débit ne progresse plus (680 puis 648 req/s) et les latences explosent (P50 de 370 à 880 ms, P99 de 630 à 2100 ms). Le goulot n'est pas le modèle (environ 0,07 ms d'inférence ONNX) mais la pile HTTP, JSON et Python avec un seul worker uvicorn par pod, et `maxReplicas: 5` plafonne le nombre de pods. À 100 utilisateurs, le service tient sans saturer (P99 de 71 ms).

## Scale-up observé

- Run 100 : l'autoscaler atteint 4 pods (DESIREDSCALE 4 / ACTUALSCALE 4). Les pods supplémentaires apparaissent environ 1 minute après le début du run : la fenêtre de 60 s lisse le débit, et le mode panique ne se déclenche pas avec une montée progressive (spawn-rate 20).
- Runs 500 et 1000 : 5 pods, soit `maxReplicas`.
- Après le run 1000 : réduction de 5 à 3 pods environ 4 minutes après la fin du test.
- Scale-to-zero : sans trafic, le pod passe en Terminating après environ 73 s et disparaît après environ 106 s.
- Démarrage à froid : 6,69 s pour le premier appel après scale-to-zero (activator, démarrage du pod, chargement du modèle), contre 5 ms à chaud.
- CPU sous charge : non relevé. Les captures de `kubectl top` ont été prises après la fin des tests (2 à 3 m par conteneur), elles ne sont donc pas représentatives.

## Analyse canary

La révision 2.0.0 (route `/version` ajoutée, même modèle v12 et même seuil 0,3182) a été déployée avec `canaryTrafficPercent: 10`. L'ISVC a affiché PREV 90 / LATEST 10. Sur 200 requêtes à `/metadata`, on observe 183 réponses en `api_version` 1.0.0 et 17 en 2.0.0, soit 8,5 % pour la v2 : cohérent avec une cible de 10 % (écart type d'environ 4 requêtes sur 200 tirages). Les adresses dédiées `latest-` et `prev-` répondent respectivement 2.0.0 et 1.0.0. La promotion à 100 % (`kubectl patch`) a été vérifiée : 50 requêtes sur 50 servies par la 2.0.0. Cette décision et ce patch sont manuels, et l'état du cluster s'éloigne de Git : les Modules 6 et 7 automatisent la progression (GitOps) et la décision (métriques).

## Points d'amélioration

- Augmenter `maxReplicas` et/ou le nombre de workers uvicorn par pod, le goulot étant le serveur HTTP Python.
- Tester `scaleMetric: concurrency` ou abaisser `scaleTarget` : la cible de 200 req/s est proche de la capacité réelle d'un pod (environ 136 req/s), donc l'autoscaler réagit tard.
- Réduire le temps de scale-up (fenêtre de 60 s plus courte) pour un trafic en rampe.
- Batching des requêtes (jusqu'à 1000 demandes par appel) pour réduire le coût par prédiction.
- Garder `minReplicas: 1` aux heures ouvrées pour éviter le démarrage à froid de 6,7 s.
- Relever le CPU des pods pendant la charge, et ajouter un canary avec surveillance automatique et rollback.
