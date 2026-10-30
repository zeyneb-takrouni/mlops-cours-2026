# Rapport comparatif — Kubeflow Pipelines vs Airflow (Lab 2)

**Binôme :** Fatma Garali · **Date :** 20 octobre 2026 · **Machine :** 12 threads CPU, 11 Go de RAM, Ubuntu sous Windows (WSL) ·
**Dataset :** german-credit

## 1. Contexte

Nous avons migré le pipeline DVC du Lab 1 (credit scoring, 1000 demandes) vers deux orchestrateurs : Kubeflow Pipelines (KFP 2.17) et Airflow 3.3. Le pipeline compte 6 steps : ingest, validate, prepare, train, evaluate, register. Chaque step est un pod Kubernetes lancé dans le cluster kind. Les deux orchestrateurs utilisent la même image `scoring-pipeline:1.0.0`, le même code `src/` et le même MLflow : seul le transport des données diffère.

## 2. Résultats quantitatifs

| Mesure | KFP | Airflow |
|---|---|---|
| Installation de zéro (min) | environ 30 | environ 45 |
| Temps jusqu'au premier run réussi (min) | non mesuré | non mesuré |
| Durée d'un run complet (min) | 3 min 59 s | 2 min 30 s |
| Relance avec les mêmes paramètres (min) | 2 min 18 s (cache) | pas de cache |
| Hot-reload : modifier un step et voir le résultat (min) | non mesuré | non mesuré |
| Lignes de code propres à l'orchestrateur | 277 | 70 |
| Mémoire consommée par l'outil (Go, `docker stats`) | non isolable (cluster kind entier : 4,5) | environ 2,2 |

Le `stage.py` (environ 140 lignes) est partagé et sert surtout à Airflow (S3, XCom).

## 3. Analyse qualitative

### Developer experience

KFP demande plus de code (277 lignes contre 70), mais il fournit des artefacts typés (Dataset, Model, Metrics), une matrice de confusion et un lineage visibles dans l'UI. Airflow est plus court à écrire, mais il ne transporte aucun fichier : il a fallu un dossier de travail S3 et du code dans `stage.py`. Le test local `kfp.local` permet de déboguer en 30 secondes sans cluster.

### Scaling

Chez KFP, un step est un pod et le parallélisme est natif. Le cache a réduit le run de 3 min 59 s à 2 min 18 s, car ingest, validate et prepare sont réutilisés. Airflow n'a pas de cache : tout est ré-exécuté. Nous n'avons pas testé de forte charge.

### Debugging

Sur le cluster, les pods driver et impl de KFP rendent le suivi détaillé mais verbeux. Nos vrais blocages venaient de l'infrastructure : téléchargements d'images coupés (`unexpected EOF`), cible `make` tuée par son propre `pkill`, et vérification Airflow lente dans `check-m2`. Les logs de task d'Airflow sont simples à lire dans l'UI.

## 4. Recommandation

KFP convient à des pipelines ML (artefacts, métriques, cache) sur Kubernetes, si l'équipe accepte son poids : 14 pods et une installation longue. Airflow convient mieux à de l'orchestration générale (ETL, planification cron) et à une équipe qui le connaît déjà, à condition de gérer soi-même le transport des fichiers.

## 5. Limites de la comparaison

Un seul run par outil, sur un petit dataset et un seul poste. Pas de KFP sur un vrai cluster managé, pas de KubernetesExecutor d'Airflow, pas de mesure du hot-reload ni de la charge. Les temps d'installation dépendent de notre réseau instable.
