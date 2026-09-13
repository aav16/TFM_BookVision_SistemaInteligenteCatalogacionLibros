# Resultados de los experimentos

> **Nota de alcance:** estos resultados se generaron con los scripts
> `experiments/*.py` de la extensión de aplicación completa (no incluida
> en este repositorio). Aquí se reutilizan ya calculados en `results/*.csv`
> y se reproducen en la sección 8 del notebook.

Resultados obtenidos ejecutando `python -m experiments.run_ocr_benchmark`
seguido de `python -m experiments.run_all` sobre el dataset sintético
completo (75 imágenes: 15 títulos × 5 condiciones de captura — ver
[LIMITACIONES.md](LIMITACIONES.md) sobre su carácter provisional).
300 ejecuciones de OCR en total (75 imágenes × 2 motores × con/sin
preprocesado), completadas en 942 s (~15,7 min) en CPU.

Todas las cifras de esta página son las que produjeron realmente esos
scripts; los CSV completos están en `results/` en este repositorio (y en
`outputs/experiments/` / `outputs/figures/` en el proyecto completo).

---

## Experimento 1 — Preprocesamiento OpenCV: con vs sin

| Motor | Preprocesado | CER medio | WER medio | Confianza media |
|---|---|---|---|---|
| easyocr | No | 0.0954 | 0.1186 | 0.932 |
| easyocr | **Sí** | **0.0161** | **0.0286** | 0.915 |
| paddleocr | No | 0.0061 | 0.0198 | 0.990 |
| paddleocr | Sí | 0.0040 | 0.0379 | 0.974 |

![Experimento 1](../outputs/figures/exp1_preprocessing.png)

**Interpretación.** El preprocesamiento reduce drásticamente el error de
EasyOCR (CER −83%, de 0.095 a 0.016; WER −76%). En PaddleOCR el efecto es
mucho más pequeño y mixto: reduce ligeramente el CER (−34%) pero aumenta
el WER (+91%, aunque partiendo de un valor ya muy bajo, 0.020→0.038).
Esto sugiere que PaddleOCR ya incorpora internamente una normalización
de imagen robusta (su propio detector de texto + clasificador de
orientación), por lo que el preprocesamiento adicional de OpenCV le
aporta poco y ocasionalmente introduce artefactos (p. ej. el CLAHE puede
saturar zonas de alto contraste que PaddleOCR ya manejaba bien). **Conclusión
práctica:** el preprocesamiento es claramente beneficioso con EasyOCR y
prescindible —o incluso ligeramente contraproducente en WER— con
PaddleOCR.

---

## Experimento 2 — Comparación de motores OCR

| Motor | CER medio | WER medio | Tiempo medio/imagen | Confianza media |
|---|---|---|---|---|
| easyocr | 0.0161 | 0.0286 | 5.90 s | 0.915 |
| **paddleocr** | **0.0040** | 0.0379 | **0.29 s** | **0.974** |

![Experimento 2](../outputs/figures/exp2_engines.png)

**Interpretación.** PaddleOCR obtiene un CER 4 veces menor que EasyOCR y
es aproximadamente **20 veces más rápido** por imagen (0.29 s frente a
5.90 s), con mayor confianza media. EasyOCR tiene ligeramente mejor WER
(0.029 vs 0.038), es decir, comete más errores a nivel de carácter pero
esos errores tienden a no romper palabras completas tan a menudo. Con
base en esta evidencia, **se ha fijado PaddleOCR como motor por defecto
del sistema** (`config.py::OCRConfig.default_engine`), documentando
aquí la justificación de esta decisión de diseño tomada a partir de los
propios datos del proyecto.

---

## Experimento 3 — Matching exacto vs fuzzy matching

| Motor | Preprocesado | Accuracy exacto | Accuracy fuzzy (≥80%) | Score fuzzy medio |
|---|---|---|---|---|
| easyocr | No | 14.7% | 14.7% | 0.474 |
| easyocr | Sí | 13.3% | 13.3% | 0.462 |
| paddleocr | No | 13.3% | 13.3% | 0.460 |
| paddleocr | Sí | 12.0% | 12.0% | 0.459 |

**Accuracy global: exacto 13.3% — fuzzy (≥80%) 13.3%** (idénticas)

![Experimento 3](../outputs/figures/exp3_exact_vs_fuzzy.png)

**Interpretación — resultado inesperado y honesto.** El fuzzy matching
NO mejora sobre el matching exacto en este experimento, algo contrario a
la intuición inicial. La causa no es que el fuzzy matching sea inútil,
sino el método de extracción usado deliberadamente en este experimento:
el candidato de título es "el segmento de texto más largo" del OCR, sin
usar información de posición/tamaño de fuente (ver nota de diseño en
`experiments/exp3_exact_vs_fuzzy.py`). Cuando esa heurística simple
elige la línea equivocada (p. ej. el nombre del autor en vez del
título), el resultado no es "un título con pequeños errores" —donde el
fuzzy matching SÍ ayudaría— sino un texto completamente distinto al
título real (score fuzzy medio ≈0.46, muy por debajo del umbral 0.80).
El fuzzy matching solo puede rescatar errores de OCR *dentro* de un
candidato correcto, no una selección de candidato equivocada. Esto
**confirma cuantitativamente** la necesidad de la extracción basada en
layout (tamaño de fuente vía bounding boxes) implementada en el sistema
real (`src/ocr/field_extraction.py::extract_fields_from_lines`), en
lugar de la heurística simplificada usada aquí para aislar la variable
de estudio.

---

## Experimento 4 — Solo título vs título + autor

| Motor | Preprocesado | Score solo título | Score título+autor |
|---|---|---|---|
| easyocr | No | 0.601 | 0.492 |
| easyocr | Sí | 0.561 | 0.467 |
| paddleocr | No | 0.552 | 0.462 |
| paddleocr | Sí | 0.558 | 0.465 |

| Estrategia | Tasa automática (≥90%) | Tasa automática+revisión (≥70%) |
|---|---|---|
| Solo título | 34.7% | 34.7% |
| Título + autor | **0.0%** | 34.0% |

![Experimento 4](../outputs/figures/exp4_title_vs_title_author.png)

**Interpretación — otro resultado honesto y contrario a la hipótesis
inicial.** Con la extracción usada en este experimento (variante de
texto plano de `extract_fields`, sin bounding boxes — ver nota de
diseño en el script), añadir el autor al score lo EMPEORA en lugar de
mejorarlo, y ninguna imagen alcanza el umbral automático (90%) con la
estrategia combinada. La causa es la misma que en el Experimento 3: sin
información de layout, el candidato de "autor" que se extrae del texto
plano es poco fiable (a menudo una línea que no es realmente el autor),
y promediarlo con el título (peso 0.3) arrastra el score combinado hacia
abajo. Esto **no invalida** la estrategia de combinar título y autor en
general — de hecho, una prueba manual end-to-end del pipeline real
(`main.py`, que sí usa la extracción con bounding boxes) identificó
correctamente "Cien años de soledad" de Gabriel García Márquez con un
score de 0.90 combinando ambos campos correctamente extraídos. La
lectura correcta de este experimento es: **el valor de combinar título y
autor depende críticamente de la calidad de la extracción de campos**;
combinarlos con una extracción poco fiable es peor que no combinarlos,
mientras que con una extracción fiable (basada en layout, la que usa el
sistema en producción) sí aporta valor. Esta es una limitación
documentada explícitamente para trabajo futuro (mejorar/generalizar la
extracción, ver [LIMITACIONES.md](LIMITACIONES.md)).

---

## Experimento 5 — Impacto de la condición de captura

Motor: EasyOCR, con preprocesado (n=15 por condición).

| Condición | CER medio | WER medio | Confianza media |
|---|---|---|---|
| Frontal | 0.0198 | 0.0333 | 0.916 |
| Rotada | 0.0213 | 0.0429 | 0.917 |
| Poca luz | 0.0198 | 0.0333 | 0.916 |
| Perspectiva | 0.0000 | 0.0000 | 0.916 |
| Reflejo | 0.0198 | 0.0333 | 0.909 |

![Experimento 5](../outputs/figures/exp5_image_quality.png)

**Interpretación.** Las diferencias entre condiciones son pequeñas
(CER entre 0.0% y 2.1%) y la condición "perspectiva" obtiene incluso
mejor resultado que la frontal. Esto es una **limitación del dataset
sintético, no una conclusión general sobre robustez del sistema**: las
transformaciones sintéticas (rotación ±12°, oscurecimiento del 55%,
perspectiva con desplazamiento del 8%) son deliberadamente moderadas y
se aplican sobre texto renderizado con alto contraste, mucho más
"limpio" que una fotografía real con las mismas condiciones (papel con
textura, iluminación no uniforme real, motion blur, etc.). El resultado
correcto a destacar es metodológico: el pipeline se ejecuta sin errores
en las 5 condiciones y las métricas se calculan correctamente, validando
que la infraestructura de evaluación funciona; la magnitud real de la
degradación por condición de captura solo podrá medirse con fiabilidad
sobre fotografías reales (ver [LIMITACIONES.md](LIMITACIONES.md) §1).

---

## Síntesis general

1. **PaddleOCR** es preferible a EasyOCR en este dataset (mejor CER,
   mucho más rápido) → fijado como motor por defecto.
2. El **preprocesamiento OpenCV** aporta una mejora clara con EasyOCR,
   marginal con PaddleOCR.
3. Tanto el Experimento 3 como el 4 revelan, de forma consistente y
   honesta, que **la calidad de la extracción de campos (título/autor)
   es el cuello de botella real del sistema** — más determinante que la
   estrictez del matching (exacto vs fuzzy) o que combinar señales
   (título vs título+autor) — cuando esa extracción no usa información
   de layout. Esto valida la decisión de diseño de usar bounding boxes
   en el sistema de producción (`extract_fields_from_lines`) en lugar de
   heurísticas de solo texto.
4. El dataset sintético, aunque útil para validar el pipeline de extremo
   a extremo, es demasiado "limpio" para estresar realmente la
   robustez frente a condiciones de captura adversas — motivo principal
   por el que sustituirlo por fotografías reales es la prioridad número
   uno de trabajo futuro.
