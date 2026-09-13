# Experimentos (Fase 10)

Todos los experimentos usan el dataset sintético de prueba (75 imágenes:
15 libros × 5 condiciones de captura — ver [LIMITACIONES.md](LIMITACIONES.md)
sobre el carácter provisional de este dataset).

Reproducir:
```bash
python -m experiments.run_ocr_benchmark   # una sola vez, genera el CSV base
python -m experiments.run_all             # ejecuta y grafica los 5 experimentos
```

El primer comando ejecuta el OCR sobre las 75 imágenes en las 4
combinaciones (2 motores × con/sin preprocesado = 300 ejecuciones) y
guarda todo en `outputs/experiments/ocr_raw_results.csv`. Los 5
experimentos posteriores solo leen ese CSV, evitando repetir el trabajo
más costoso (la inferencia OCR) para cada análisis.

---

## Experimento 1 — Preprocesamiento OpenCV: con vs sin

**Pregunta.** ¿El preprocesamiento (`src/image_processing/preprocessing.py`:
deskew, CLAHE, denoise, autocrop) mejora realmente la calidad del OCR, o
solo añade tiempo de cómputo?

**Método.** Para cada imagen del dataset y cada motor OCR, se ejecuta el
OCR dos veces: sobre la imagen original y sobre la imagen preprocesada.
Se calcula CER y WER entre el texto reconocido y el texto de referencia
(`"{título} {autor}"` normalizado — el texto que literalmente está
impreso en la portada).

**Métrica.** CER y WER medios, agrupados por motor y por si se aplicó
preprocesado.

**Script.** `experiments/exp1_preprocessing.py`

---

## Experimento 2 — Comparación de motores OCR

**Pregunta.** Entre EasyOCR y PaddleOCR (los dos motores elegidos por
ser 100% instalables vía pip, ver §4 de
[DOCUMENTACION_TECNICA.md](DOCUMENTACION_TECNICA.md)), ¿cuál ofrece
mejor equilibrio entre precisión y velocidad?

**Método.** Se comparan ambos motores en su configuración de producción
(con preprocesado activado): CER, WER, confianza media reportada por el
propio motor, y tiempo medio de inferencia por imagen.

**Métrica.** CER/WER medios, tiempo medio (s) por imagen.

**Script.** `experiments/exp2_ocr_engines.py`

---

## Experimento 3 — Matching exacto vs fuzzy matching

**Pregunta.** ¿Cuánto mejora la tasa de reconocimiento correcto del
título al permitir similitud aproximada (RapidFuzz) frente a exigir una
coincidencia exacta de cadenas (tras normalizar mayúsculas/acentos)?

**Método.** Se toma como candidato de título el fragmento de texto más
largo detectado por el OCR en cada imagen, y se compara contra el título
real: `exact_match` (coincidencia == tras normalizar) frente a
`fuzzy_similarity >= 0.80` (umbral de aceptación fuzzy).

**Métrica.** Accuracy (proporción de aciertos) de cada estrategia,
global y por motor/preprocesado.

**Script.** `experiments/exp3_exact_vs_fuzzy.py`

---

## Experimento 4 — Solo título vs título + autor

**Pregunta.** ¿Cuánto aporta incluir el autor en el scoring de
identificación (Fase 6), frente a puntuar solo con el título?

**Método.** Para cada resultado OCR, se extraen los campos de título y
autor (`src/ocr/field_extraction.py`) y se calcula:
- `score_solo_titulo` = similitud(título_ocr, título_real)
- `score_titulo_autor` = 0.6·similitud(título) + 0.3·similitud(autor)

usando los pesos por defecto de `config.MatchingConfig`. Se mide qué
fracción de casos alcanza cada umbral de decisión (automático ≥0.90,
revisión ≥0.70) con cada estrategia.

**Nota de diseño.** Este experimento puntúa contra la ficha bibliográfica
REAL conocida del dataset sintético (no contra resultados en vivo de las
APIs), para aislar el efecto del campo usado en el scoring de la
variabilidad/límites de tasa de las APIs externas — ver el docstring de
`experiments/exp4_title_vs_title_author.py` para el detalle completo de
esta decisión.

**Métrica.** Tasa de identificación automática y automática+revisión,
para cada estrategia.

**Script.** `experiments/exp4_title_vs_title_author.py`

---

## Experimento 5 — Impacto de la condición de captura

**Pregunta.** ¿Cómo se degrada la calidad del OCR según la condición de
la fotografía (frontal, rotada, poca luz, perspectiva distorsionada,
reflejo)?

**Método.** Se agrupan los resultados del motor/configuración de
producción (EasyOCR + preprocesado) por condición de captura, calculando
CER, WER y confianza media del OCR en cada una.

**Métrica.** CER/WER y confianza media por condición.

**Script.** `experiments/exp5_image_quality.py`

---

Los resultados numéricos obtenidos al ejecutar estos experimentos se
documentan en [RESULTADOS.md](RESULTADOS.md), junto con las gráficas
correspondientes en `outputs/figures/`.
