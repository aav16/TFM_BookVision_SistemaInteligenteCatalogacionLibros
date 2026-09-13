# Documentación técnica

Estos documentos describen la documentación técnica completa del sistema
BookVision (arquitectura, decisiones de diseño, experimentos y resultados,
limitaciones). Se han copiado aquí, sin modificar, desde el proyecto
completo del TFM para acompañar al núcleo algorítmico de este repositorio.

**Nota importante:** algunos de estos documentos referencian rutas de
código (`src/`, `main.py`, `app/`, `tests/`, `experiments/`) que pertenecen
a una extensión de aplicación completa (interfaz Streamlit, persistencia en
base de datos, CLI, suite de pruebas) construida sobre este mismo núcleo
algorítmico. Esa extensión **no forma parte de este repositorio**, que
contiene únicamente el notebook autocontenido
(`TFM_BookVision_Nucleo_Algoritmico.ipynb`) evaluado como TFM — ver el
[README](../README.md) de la raíz para más detalle sobre este alcance.

- [DOCUMENTACION_TECNICA.md](DOCUMENTACION_TECNICA.md) — arquitectura,
  requisitos, modelo de datos y justificación de decisiones tecnológicas.
- [EXPERIMENTOS.md](EXPERIMENTOS.md) — metodología de los 5 experimentos
  de evaluación sobre el dataset sintético.
- [RESULTADOS.md](RESULTADOS.md) — resultados y gráficas de esos 5
  experimentos.
- [RESULTADOS_FOTOS_REALES.md](RESULTADOS_FOTOS_REALES.md) — validación
  con 253 fotografías reales, incluida la calibración del umbral de
  decisión.
- [LIMITACIONES.md](LIMITACIONES.md) — limitaciones actuales y trabajo
  futuro.
