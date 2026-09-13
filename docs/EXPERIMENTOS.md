# Experimentos (Fase 10)

> **Nota de alcance:** este documento describe la metodología con la que se
> generaron los resultados que este repositorio reutiliza ya calculados en
> `results/*.csv`, cargados y graficados en la sección 8 del notebook
> (`TFM_BookVision_Nucleo_Algoritmico.ipynb`).

Todos los experimentos usan el dataset sintético de prueba (75 imágenes:
15 libros × 5 condiciones de captura — ver [LIMITACIONES.md](LIMITACIONES.md)
sobre el carácter provisional de este dataset).

El OCR se ejecutó sobre las 75 imágenes en las 4 combinaciones (2 motores
× con/sin preprocesado = 300 ejecuciones), guardando los resultados en
`results/ocr_raw_results.csv`. Los 5 experimentos posteriores solo leen
ese CSV, evitando repetir el trabajo más costoso (la inferencia OCR) para
cada análisis.

---

## Experimento 1 — Preprocesamiento OpenCV: con vs sin

**Pregunta.** ¿El preprocesamiento (deskew, CLAHE, denoise, autocrop —
implementado también en la sección 1 de este notebook) mejora realmente
la calidad del OCR, o solo añade tiempo de cómputo?

**Método.** Para cada imagen del dataset y cada motor OCR, se ejecuta el
OCR dos veces: sobre la imagen original y sobre la imagen preprocesada.
Se calcula CER y WER entre el texto reconocido y el texto de referencia
(`"{título} {autor}"` normalizado — el texto que literalmente está
impreso en la portada).

**Métrica.** CER y WER medios, agrupados por motor y por si se aplicó
preprocesado.

---

## Experimento 2 — Comparación de motores OCR

**Pregunta.** Entre EasyOCR y PaddleOCR (los dos motores elegidos por
ser 100% instalables vía pip, ver §4 de
[DOCUMENTACION_TECNICA.md](DOCUMENTACION_TECNICA.md)), ¿cuál ofrece
mejor equilibrio entre precisión y velocidad?

**Método.** Se comparan ambos motores con preprocesado activado: CER,
WER, confianza media reportada por el propio motor, y tiempo medio de
inferencia por imagen.

**Métrica.** CER/WER medios, tiempo medio (s) por imagen.

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

---

## Experimento 4 — Solo título vs título + autor

**Pregunta.** ¿Cuánto aporta incluir el autor en el scoring de
identificación (Fase 6), frente a puntuar solo con el título?

**Método.** Para cada resultado OCR, se extraen los campos de título y
autor (equivalente a `extract_fields()` en la sección 3 de este notebook)
y se calcula:
- `score_solo_titulo` = similitud(título_ocr, título_real)
- `score_titulo_autor` = 0.6·similitud(título) + 0.3·similitud(autor)

usando los pesos por defecto (`WEIGHT_TITLE`/`WEIGHT_AUTHOR` en la
sección 6 del notebook). Se mide qué fracción de casos alcanza cada
umbral de decisión (automático ≥0.90, revisión ≥0.70) con cada estrategia.

**Nota de diseño.** Este experimento puntúa contra la ficha bibliográfica
REAL conocida del dataset sintético (no contra resultados en vivo de las
APIs), para aislar el efecto del campo usado en el scoring de la
variabilidad/límites de tasa de las APIs externas.

**Métrica.** Tasa de identificación automática y automática+revisión,
para cada estrategia.

---

## Experimento 5 — Impacto de la condición de captura

**Pregunta.** ¿Cómo se degrada la calidad del OCR según la condición de
la fotografía (frontal, rotada, poca luz, perspectiva distorsionada,
reflejo)?

**Método.** Se agrupan los resultados con EasyOCR y preprocesado activado
por condición de captura, calculando CER, WER y confianza media del OCR
en cada una. Nota de coherencia: este experimento se ejecutó con EasyOCR
antes de adoptar PaddleOCR como motor por defecto (decisión que se toma
precisamente a partir del Experimento 2); no se repitió después con
PaddleOCR por no aportar información adicional a la pregunta de este
experimento (el efecto relativo de la condición de captura).

**Métrica.** CER/WER y confianza media por condición.

---

Los resultados numéricos obtenidos al ejecutar estos experimentos se
documentan en [RESULTADOS.md](RESULTADOS.md), junto con las gráficas
correspondientes en `figures/`.
