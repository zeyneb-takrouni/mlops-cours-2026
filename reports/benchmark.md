# Benchmark de latence — german-credit, modèle v12

1000 appels par backend, après 10 appels de chauffe · x86_64 · Python 3.12.15

| Backend | P50 (ms) | P95 (ms) | P99 (ms) |
|---|---|---|---|
| sklearn_pipeline | 5.969 | 10.323 | 16.828 |
| onnx_python | 0.082 | 0.209 | 0.400 |
| onnx_batch_100 | 0.384 | 0.707 | 1.056 |

Accélération ONNX vs scikit-learn (P50, 1 ligne) : x72.5
