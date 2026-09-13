# Limitaciones y trabajo futuro

## Limitaciones actuales

### 1. Dataset de evaluación sintético (provisional)
El dataset usado para desarrollar y evaluar el pipeline (`data/test/synthetic/`,
75 imágenes) se genera programáticamente con PIL, renderizando título y
autor sobre fondos de color con distintas transformaciones (rotación,
oscurecimiento, perspectiva, reflejo simulado). **No son fotografías
reales de portadas comerciales.**

- **Por qué**: evita cualquier problema de derechos de autor sobre
  diseños de portada reales mientras se desarrollaba el sistema, y
  permite tener ground truth exacto (título/autor conocidos con
  certeza) para calcular métricas automáticamente.
- **Impacto**: las condiciones sintéticas (rotación, oscurecimiento,
  perspectiva) son más "limpias" que las de una fotografía real: no
  capturan variabilidad tipográfica entre editoriales, ilustraciones
  complejas de fondo, texturas de papel, ni reflejos realistas de
  cámara de móvil.
- **Plan de sustitución**: el sistema está preparado para recibir
  fotografías reales sin cambios de código — basta con colocarlas en
  `data/raw/` o subirlas por la interfaz Streamlit.
- **Actualización — ya se completó esta validación**: se evaluó el
  pipeline con 253 fotografías reales aportadas por el autor (34 en una
  primera tanda, ampliadas después a 253). Resultados completos, con
  hallazgos honestos (incluidos dos falsos positivos reales, la
  refutación con datos de la hipótesis inicial sobre el impacto de la
  rotación, y la confirmación a escala del cuello de botella por límite
  de peticiones de Google Books), en
  [RESULTADOS_FOTOS_REALES.md](RESULTADOS_FOTOS_REALES.md).

### 2. Heurística de extracción título/autor basada en layout
`src/ocr/field_extraction.py` asume que el título de una portada se
imprime en el cuerpo de letra más grande. Es una heurística razonable y
verificada empíricamente en el dataset de prueba, pero puede fallar en:
- Portadas donde el autor tiene más protagonismo tipográfico que el título
  (frecuente en autores best-seller).
- Portadas sin jerarquía clara de tamaños (todo el texto del mismo tamaño).
- Portadas con texto decorativo/artístico que el OCR confunde con título.

Mitigación implementada: el sistema no exige una única interpretación
correcta — genera varios candidatos y dejas que el scoring contra las
APIs externas (que sí tienen el título/autor reales) decida; además la
interfaz permite corrección manual cuando la confianza es baja.

**Confirmado con fotos reales**: en la evaluación de 253 fotografías
reales ([RESULTADOS_FOTOS_REALES.md](RESULTADOS_FOTOS_REALES.md)) se
observó justo el primer caso (autor con más protagonismo tipográfico
que el título) en varias portadas distintas de forma recurrente
("JAMES DASHNER", "NORA ROBERTS", "GUSTAVO ADOLFO BÉCQUER" impresos en
letra igual o mayor que el título del libro), confirmando que no es un
caso límite hipotético sino un patrón de fallo real y frecuente en
portadas comerciales de autores de gran tirada.

### 2b. Riesgo de confusión entre libros de una misma saga/autor
Cuando el título extraído queda incompleto (p. ej. solo una palabra de
un título de dos líneas), el sistema puede confundir dos libros
distintos del mismo autor/saga si comparten suficientes palabras y el
autor coincide exactamente. **Confirmado con datos reales**: la foto de
la portada de *"Mentes poderosas"* (Alexandra Bracken) se identificó
automáticamente, con 90% de confianza, como *"Nunca olvidan"* (el
segundo libro de la misma trilogía) — ver el análisis completo en
[RESULTADOS_FOTOS_REALES.md](RESULTADOS_FOTOS_REALES.md). Es la
limitación de fiabilidad más seria detectada en todo el proyecto, ya
que ocurre con confianza alta (por encima del umbral automático), no
en la zona de revisión manual. Mitigación futura recomendada: exigir
una coincidencia de ISBN o penalizar más fuertemente los candidatos
cuya diferencia de título, aunque pequeña, corresponda a una obra
distinta y catalogada por separado (actualmente el peso del ISBN es
solo 0.1 y no es obligatorio).

### 3. Dependencia de APIs externas
Google Books API aplica un límite de peticiones anónimas por IP que, en
el entorno de desarrollo usado, se alcanzó rápidamente (respuestas 429).
Se mitigó con reintentos limitados y backoff, y usando Open Library como
fuente redundante (sin límite de tasa conocido para uso normal). En un
uso intensivo (cientos de libros seguidos) podría notarse una
degradación temporal de Google Books como fuente.

**Confirmado a escala real**: al ampliar la validación de 34 a 253
fotos evaluadas de forma consecutiva, el límite de Google Books se
saturó de manera sostenida (avisos `429` casi continuos en
`outputs/logs/bookvision.log`), y la proporción de fotos "no
identificadas" subió del 76.5% al 89.3% — ver
[RESULTADOS_FOTOS_REALES.md](RESULTADOS_FOTOS_REALES.md). Confirma con
datos, no solo por anticipación teórica, que esta es la limitación de
escalabilidad más relevante del sistema tal como está configurado hoy
(sin clave de API propia).

### 4. Sin GPU / modelos ligeros
Por requisito de reproducibilidad (RNF2), se usan modelos que corren en
CPU (MobileNetV3-Small, EasyOCR/PaddleOCR en modo CPU). Esto es más
lento que con GPU, pero evita depender de hardware específico o de
CUDA/drivers, que romperían la reproducibilidad para cualquier persona
que clone el repositorio.

### 5. Detección de duplicados visuales sensible al umbral
El umbral de similitud visual (`config.MATCHING.visual_similarity_threshold
= 0.92`) se fijó por inspección manual, no por una búsqueda experimental
exhaustiva (no se dispone de suficientes fotos repetidas del mismo libro
físico en el dataset sintético para ese estudio). Con fotografías reales
del mismo ejemplar en distintos ángulos, este umbral debería recalibrarse.

**Fallo real detectado y corregido durante el desarrollo:** al probar el
pipeline completo se observó un falso positivo — dos portadas sintéticas
de libros distintos ("El Principito" y "Cien años de soledad"), que por
azar compartían la misma paleta de color (`_PALETTES` en
`synthetic_dataset.py`), obtuvieron una similitud de embedding visual de
0.927 (por encima del umbral 0.92), marcándose incorrectamente como
duplicados. Causa: un embedding de una CNN preentrenada en ImageNet
captura sobre todo color/textura/composición global, no el contenido
semántico exacto de la portada. **Corrección aplicada:**
`src/matching/book_identifier.py::is_visual_duplicate_corroborated`
exige ahora que el título identificado por texto no contradiga
claramente el título del supuesto duplicado antes de aceptar la
coincidencia puramente visual (test de regresión en
`tests/test_book_identifier.py`). Esto reduce el riesgo pero no lo
elimina del todo si el OCR no obtiene texto útil de ninguna de las dos
portadas — un caso extremo que solo la comprobación por ISBN podría
resolver con certeza.

## Trabajo futuro

- Sustituir el dataset sintético por un corpus de fotografías reales
  (100-200 imágenes) y recalcular todas las métricas.
- Añadir un modelo de detección de objetos (p. ej. YOLO ligero) para
  localizar automáticamente el libro dentro de fotos con fondo complejo,
  en lugar del contorno por umbralización actual (`find_cover_contour`).
- Explorar embeddings de imagen-texto (CLIP) para una identificación
  visual directa contra un catálogo de portadas oficiales, si en el
  futuro se dispone de licencia/acceso a ese catálogo.
- Ampliar la extracción de campos con un modelo de NLP (NER) entrenado
  o ajustado específicamente para separar título/autor/editorial en
  texto ruidoso de portada, en lugar de heurísticas de layout.
- Exportar el inventario a formatos estándar (BibTeX, MARC) para
  interoperar con sistemas bibliotecarios reales.
- Desplegar la interfaz Streamlit en un servicio accesible remotamente
  (requeriría credenciales de hosting — intervención del usuario).
