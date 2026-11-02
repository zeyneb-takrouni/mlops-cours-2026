# Rapport — Lab 4 : packaging du modèle

Binôme : Garali_Fatma · Dataset : german-credit · Modèle : `scoring-model` v12

## 1. Comparatif des tailles

| Image | Taille | CVE HIGH + CRITICAL |
|---|---|---|
| `scoring-api:naive` | 2,19 Go | 489 (473 HIGH, 16 CRITICAL) |
| `scoring-api:optimized` | 429 Mo | 45 (45 HIGH, 0 CRITICAL) |
| `scoring-api:bentoml` | 1,41 Go | 126 (125 HIGH, 1 CRITICAL) |

L'image optimisée est 5 fois plus petite que la baseline. Elle tourne en utilisateur
non-root (`uid=10001`) ; la baseline tourne en root (`uid=0`) et embarque `.env` et `.git`.

## 2. Comparatif des latences

1000 appels par backend après 10 appels de chauffe, x86_64, Python 3.12.15.

| Backend | P50 (ms) | P95 (ms) | P99 (ms) |
|---|---|---|---|
| sklearn_pipeline | 5,969 | 10,323 | 16,828 |
| onnx_python | 0,082 | 0,209 | 0,400 |
| onnx_batch_100 | 0,384 | 0,707 | 1,056 |

Accélération ONNX contre scikit-learn (P50, une ligne) : x72,5.

## 3. Interprétation

- **Taille.** La baseline part de `python:3.12` complet et copie tout le dépôt (`.git`,
  `data/`, `.env`). L'image optimisée part de `python:3.12-slim`, utilise un build en deux
  stages, n'installe que ce que l'API exécute (ni MLflow ni scikit-learn : elle ne fait
  qu'exécuter le graphe ONNX), supprime les dossiers `tests` et le bytecode des
  dépendances, et n'embarque que le code, le contrat et le modèle (liste blanche du
  `.dockerignore`). Ce dernier nettoyage a fait passer l'image de 535 à 429 Mo.
- **Latence.** Pour une ligne, scikit-learn coûte environ 6 ms contre 0,08 ms pour ONNX
  Runtime. Notre explication, que nous n'avons pas profilée : le pipeline scikit-learn
  paie un surcoût fixe par appel (validation du DataFrame, un appel Python par
  transformateur), alors qu'ONNX exécute un graphe unique. Le calcul lui-même est
  minuscule. Un lot de 100 lignes coûte 0,38 ms, soit environ 4 µs par ligne.
- **Scans.** Trivy classe les images dans le même ordre que leur taille : 489 CVE pour la
  baseline, 126 pour le Bento, 45 pour l'image optimisée, sans aucune CRITICAL. Moins de
  paquets installés signifie moins de surface d'attaque ; nous n'avons pas détaillé la
  répartition entre paquets système et paquets Python. ModelScan ne trouve rien dans
  l'ONNX (graphe déclaratif sans code) mais détecte `os.system` dans le pickle piégé de
  démonstration : charger un pickle exécute du code arbitraire. Le modèle du registry est
  au format skops, qui ne demande que le type `numpy.dtype`.

## 4. Choix recommandé

Nous retiendrions `scoring-api:optimized` pour la production : la plus petite image, la
latence la plus basse, aucune CVE CRITICAL, un utilisateur non-root, un modèle ONNX sans
code exécutable et un schéma d'entrée généré depuis `dataset.yaml`. BentoML apporte son
outillage (build, packaging) mais exécute le pipeline scikit-learn sérialisé en pickle,
pour une image trois fois plus lourde et 126 CVE ; nous le réserverions à une équipe qui
l'utilise déjà.

## 5. Pistes d'amélioration

- Image distroless ou base plus récente pour réduire les 45 CVE HIGH restantes ; épingler
  la base par son digest.
- Scan Trivy dans la CI (Lab 6), avec échec du build si une CVE CRITICAL apparaît.
- Quantization INT8 : peu d'intérêt ici (le modèle fait 11 ko), utile pour un modèle plus
  gros.
- Plusieurs workers uvicorn et préchargement du modèle dans un init container.
- Limites de notre comparaison : une seule machine, 1000 appels, une exécution ; et
  ModelScan signale une erreur de son scanner NumPy avec NumPy 2.5, donc les fichiers
  `.npy` ne sont pas vérifiés (notre modèle n'en contient pas).
