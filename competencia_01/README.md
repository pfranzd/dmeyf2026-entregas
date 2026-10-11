# Competencia 01 — entrega final: e032_expansiva_auc_topk con 10500 envíos

Esta carpeta contiene todo lo necesario para replicar **exactamente** el archivo que se subió
como entrega final de la competencia:

| | |
|---|---|
| Archivo entregado | `20261009-232755_e032_expansiva_auc_topk_promedio_e10500.csv` (solo `numero_de_cliente`, sin encabezado, 10500 filas) |
| sha256 | `5d2713bd87934be94c6c54ca9ec59ba013104090faeb5fe982228516fc9180d1` |
| Puntaje público | 107.7175 M |
| Período objetivo | 202108 |

El dataset de la competencia **no está en el repositorio**: el script lo descarga solo desde la
URL pública de la cátedra (`competencia_01_crudo.csv`).

## Cómo replicarla

Hace falta Python 3.11 y unos 10 GB de RAM. Hay tres niveles; todos terminan comparando el
sha256 del CSV generado con el de arriba (`RESULTADO: OK, coincide`):

| Modo | Qué rehace | Tiempo aprox. |
|---|---|---|
| `predecir` (default) | Datos y features desde el crudo, y **predice** con los 24 modelos ya entrenados | ~16 min |
| `entrenar` | Lo anterior, pero **reentrena** los 24 modelos con los hiperparámetros de `competencia_1/definitiva/params.json` | ~80 min |
| `completo` | **Todo, incluida la optimización de hiperparámetros** (Optuna, 60 trials) que dio origen a esos hiperparámetros | ~1,5 h |

**Linux o VM de GCP:**

```bash
git clone https://github.com/pfranzd/dmeyf2026-entregas.git && cd dmeyf2026-entregas/competencia_01
bash reproducir.sh              # o: bash reproducir.sh entrenar | bash reproducir.sh completo
```

**Windows (PowerShell):**

```powershell
git clone https://github.com/pfranzd/dmeyf2026-entregas.git; cd dmeyf2026-entregas\competencia_01
py -3.11 -m venv .venv
.venv\Scripts\python -m pip install --upgrade pip
.venv\Scripts\python -m pip install -r requirements-lock.txt
$env:ENTREGA_MODO = "predecir"      # o "entrenar" / "completo"
.venv\Scripts\python entrega.py
```

**Verificación:** los tres modos se corrieron desde un clon limpio de este repositorio, con el entorno
instalado desde `requirements-lock.txt` y el dataset descargado de la URL pública, y los tres generan un CSV
con el sha256 indicado arriba (el 2026-10-10: `predecir` ~16 min, `completo` ~92 min).

También se puede abrir `entrega.ipynb` (es el mismo código que `entrega.py`) y ejecutarlo
completo; el modo se elige en la segunda celda.

El CSV generado queda en `work/entrega/runs/<run>/submits/` y el script compara su sha256 con
`competencia_1/definitiva/entrega.json`.

## Entorno

- `requirements-lock.txt`: **versiones exactas** de todas las librerías (incluidas las
  transitivas) con las que se generó la entrega. `requirements.txt` lista solo las directas.
- Python 3.11 (`.python-version`). El script avisa si alguna versión instalada difiere.
- La igualdad byte a byte depende de las versiones de LightGBM y DuckDB y de
  `lgbm.fijos.num_threads` (8) en `competencia_1/configs/base.yaml`; no los cambies.

## Qué hace el modelo

- LightGBM, objetivo `binary` sobre `BAJA+1 + BAJA+2`; la ganancia se mide sobre BAJA+2 (+780.000 / −20.000).
- Features: datos originales + familias temporales (lags, deltas, ventanas, tendencia) según `competencia_1/configs/exp/e032_expansiva_auc_topk.yaml`.
- Optimización: validación temporal con gap de 2 meses; objetivo `auc` promedio de los folds:
  - train [202103] → validación 202105
  - train [202103, 202104] → validación 202106
- Undersampling de CONTINUA en la optimización: 0.25 (el modelo final entrena sin undersampling).
- Modelo final entrenado con los meses [202103, 202104, 202105, 202106]; predice 202108.
- Ensamble final: 12 conjuntos de hiperparámetros × 2 semillas; se promedian las probabilidades.

## De dónde salen los hiperparámetros (optimización)

Los hiperparámetros **no están fijados a mano**: salen de una optimización bayesiana con Optuna, incluida en esta carpeta.

- Estudio: `optimizacion/estudio.db` (SQLite de Optuna, 60 trials, 37 completos y 23 podados) y `optimizacion/resumen.json`.
- La entrega usa los 12 mejores trials: #43, #41, #32, #22, #27, #44, #57, #31, #51, #42, #52, #21.
- Para rehacerla de punta a punta: `bash reproducir.sh completo` (corre `competencia_1/configs/exp/e032_expansiva_auc_topk.yaml`: Optuna → mejores trials → modelos → CSV) y comprueba el sha256.


## Contenido de la carpeta

```
entrega.py / entrega.ipynb     punto de entrada (mismo código)
reproducir.sh                  atajo para Linux / GCP
requirements*.txt, .python-version
competencia_1/                 pipeline (datos → features → Optuna → modelos → CSV)
  configs/base.yaml            defaults
  configs/exp/e032_expansiva_auc_topk.yaml   configuración del experimento (optimización + final)
  definitiva/                  config.yaml, params.json, entrega.json y modelos/ (entrega congelada)
dmeyf/, config/                métricas y semillas
optimizacion/                  evidencia de la optimización (estudio de Optuna + resumen)
```

## Trazabilidad

Generada el 2026-10-10 21:07 desde el repositorio de trabajo, commit `38666d6a17`, run
`20261009-232755_e032_expansiva_auc_topk`. El CSV de ese run coincide byte a byte con el que reproduce esta carpeta.
