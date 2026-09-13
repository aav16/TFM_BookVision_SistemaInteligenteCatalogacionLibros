# BookVision — Núcleo algorítmico (TFM)

[![Ejecutar notebook de extremo a extremo](https://github.com/aav16/TFM_BookVision_SistemaInteligenteCatalogacionLibros/actions/workflows/execute-notebook.yml/badge.svg)](https://github.com/aav16/TFM_BookVision_SistemaInteligenteCatalogacionLibros/actions/workflows/execute-notebook.yml)

Trabajo de Fin de Máster — Máster Universitario en Ciencia de Datos (UCJC)
**Autor:** Antonio Álvarez Velasco · **Director:** Anas Ahachad

Este repositorio contiene el sistema BookVision —el sistema de identificación
de libros a partir de una fotografía de su portada que constituye el TFM—,
condensado en un único notebook autocontenido y documentado en
[`docs/`](docs/). El objeto de estudio de este TFM es el algoritmo de
identificación y su evaluación experimental, no una aplicación de usuario
final; por eso el repositorio no incluye interfaz web, base de datos ni CLI
(ver "Qué no incluye este repositorio (y por qué)" más abajo). Todo el
contenido algorítmico relevante está en un único notebook.

## Qué revisar

Abre **`TFM_BookVision_Nucleo_Algoritmico.ipynb`** con Jupyter. Contiene, en
orden, y ejecutado sobre imágenes y datos reales del proyecto:

1. Preprocesamiento de imagen (OpenCV): redimensionado, **corrección de
   perspectiva y recorte automático** de la portada (detección de contorno +
   `warpPerspective`), *deskew*, reducción de ruido y contraste — con demo
   visual antes/después sobre una imagen rotada y otra con perspectiva.
2. OCR (EasyOCR/PaddleOCR) — usando resultados reales ya calculados, con una
   celda opcional para ejecutar OCR en vivo si tienes `paddleocr` instalado.
3. Extracción de campos (título/autor/ISBN) — comparando la heurística
   ingenua (línea más larga) frente a la real, basada en tamaño de
   letra/posición.
4. Huella visual con una CNN preentrenada (MobileNetV3) para detectar
   duplicados — similitud coseno entre portadas.
5. Matching de texto con RapidFuzz.
6. Score combinado (título + autor) y umbral de decisión (identificación
   automática / revisión / no identificado), aplicado a las 15 imágenes de
   ejemplo.
7. Consulta en vivo (opcional) a **Google Books, con Open Library como
   respaldo** si falla o no hay resultados — y cálculo del **score completo
   con sus tres términos** (título, autor e ISBN) sobre el candidato real
   que devuelva.
8. Los 5 experimentos de evaluación del proyecto (Fase 10), con sus gráficas
   e interpretación.
9. Calibración empírica del umbral de decisión (0.90/0.70), ejecutada sobre
   253 fotografías reales: precisión, cobertura y recall para cada umbral
   candidato, con su gráfica.

(La sección "0", justo antes de la 1, presenta las imágenes de ejemplo.)

## Cómo ejecutarlo

```bash
pip install -r requirements.txt
jupyter notebook TFM_BookVision_Nucleo_Algoritmico.ipynb
```

Ejecuta Jupyter desde la raíz de este repositorio —
el notebook carga las imágenes de `data/synthetic_samples/` y los resultados
de `results/` con rutas relativas.

**Nota sobre la sección 4 (embeddings):** en algunos entornos Windows,
PyTorch puede fallar al cargar dentro del proceso del kernel de Jupyter
(`OSError` al cargar `shm.dll`) aunque funcione perfectamente en un script
normal — es un problema de entorno conocido, no del código. La celda lo
detecta y se salta con un aviso en vez de romper el notebook; si ocurre,
basta con reiniciar el kernel o ejecutar esa celda de nuevo.

## Contenido de la carpeta

- `TFM_BookVision_Nucleo_Algoritmico.ipynb` — el notebook (documento único).
- `TFM_BookVision_Nucleo_Algoritmico_ejecutado.html` — el mismo notebook ya
  ejecutado, exportado a HTML, para ver los resultados sin instalar nada.
- `data/synthetic_samples/` — 15 imágenes (3 títulos × 5 condiciones de
  captura) tomadas del dataset sintético completo del proyecto, suficientes
  para las demos en vivo.
- `results/*.csv` — resultados ya calculados de los 5 experimentos, del
  benchmark de OCR (300 ejecuciones sobre las 75 imágenes del dataset
  completo) y de la calibración del umbral sobre 253 fotografías reales. Se
  reutilizan aquí para no depender de instalar ambos motores OCR, de volver
  a ejecutar ~16 minutos de benchmark, ni de disponer de las 253 fotografías
  (no distribuidas por derechos de autor de las portadas).
- `requirements.txt` — dependencias mínimas para este notebook.
- `docs/` — documentación técnica y de resultados, con sus gráficas en
  `docs/figures/` (ver [docs/README.md](docs/README.md) para el índice
  completo).
- `.github/workflows/execute-notebook.yml` — comprueba automáticamente, en
  cada cambio, que el notebook se ejecuta de extremo a extremo sin errores.
- `LICENSE` — licencia MIT.

## Qué no incluye este repositorio (y por qué)

El objeto de estudio de este TFM es el algoritmo de identificación de libros
y su evaluación experimental, no su empaquetado como producto: por eso este
repositorio no incluye interfaz de usuario, persistencia en base de datos ni
línea de comandos. Como posible continuación futura del proyecto, cabría
plantearse envolver este núcleo algorítmico en una aplicación completa, pero
esa ingeniería de aplicación no formaba parte del objeto de estudio de este
TFM y, por tanto, no se ha abordado aquí.
