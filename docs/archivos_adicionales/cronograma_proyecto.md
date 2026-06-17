# Cronograma del proyecto — Sistema de inteligencia operativa WPE

**Autor:** Alejandro Pujana Quintero · Consultor  
**Cliente:** Área de Workplace Experience · Tabacalera multinacional · Clúster CCA  
**Fecha de inicio:** 1 de julio de 2026  
**Fecha de cierre estimada:** 13 de octubre de 2026  

---

## Resumen de fases

| Fase | Componente | Fecha de inicio | Fecha de cierre | Entregable principal |
|------|------------|----------------|----------------|----------------------|
| Fase 1 | Adquisición y validación de datos | 1 de julio de 2026 | 15 de julio de 2026 | Diccionario de datos · Informe de calidad · Línea base proceso manual |
| Fase 2 | Preprocesamiento y almacenamiento | 16 de julio de 2026 | 30 de julio de 2026 | Base Silver · Pipeline documentado · Informe post-procesamiento |
| Fase 3 | EDA y modelado de anomalías | 31 de julio de 2026 | 24 de agosto de 2026 | Reporte EDA ejecutivo · Modelo de anomalías validado · Informe de resultados |
| Fase 4 | Modelo semántico y dashboard de BI | 25 de agosto de 2026 | 14 de septiembre de 2026 | Dashboard ejecutivo funcional · Esquema Gold · Manual de uso — **MVP** |
| Fase 5 | Modelo predictivo de clasificación | 15 de septiembre de 2026 | 28 de septiembre de 2026 | Documento de viabilidad · Modelo documentado *(condicional)* |
| Fase 6 | Capa de IA generativa (RAG) | 29 de septiembre de 2026 | 13 de octubre de 2026 | Arquitectura RAG documentada · Demo *(si el tiempo lo permite)* |

---

## Detalle por fase

### Fase 1 — Adquisición y validación de datos
**1 de julio – 15 de julio de 2026**

Coordinación de la extracción de datos históricos del sistema ERP (periodo 2024–2025), validación de la estructura de campos recibida, documentación del diccionario de datos y generación del informe de calidad inicial. Incluye la documentación de la línea base del proceso manual actual como referencia para medir el impacto del sistema.

**Entregables:**
- Diccionario de datos con descripción de negocio de cada variable
- Informe de calidad inicial (nulos, duplicados, rangos, tipos)
- Línea base del tiempo del proceso manual de cierre mensual

---

### Fase 2 — Preprocesamiento y almacenamiento
**16 de julio – 30 de julio de 2026**

Construcción del pipeline ETL en Python que transforma los datos crudos en una base de datos estructurada. Incluye decodificación de códigos del ERP, normalización de formatos, tratamiento de registros irregulares y anonimización según los acuerdos de confidencialidad.

**Entregables:**
- Base de datos Silver estructurada (SQLite)
- Pipeline de preprocesamiento documentado y reproducible
- Informe de calidad post-procesamiento con registro de decisiones

---

### Fase 3 — EDA y modelado de anomalías
**31 de julio – 24 de agosto de 2026**

Análisis exploratorio del comportamiento de gasto por cuenta contable, centro de costo y periodo. Implementación del modelo de detección de patrones atípicos (Isolation Forest + Z-score), evaluado con métricas técnicas y validado con criterio de negocio del área.

**Entregables:**
- Reporte de EDA en lenguaje ejecutivo
- Modelo de anomalías validado con métricas documentadas
- Informe de resultados con interpretación de negocio y recomendaciones

---

### Fase 4 — Modelo semántico y dashboard de BI ⭐ MVP
**25 de agosto – 14 de septiembre de 2026**

Construcción del esquema estrella en Gold con vistas SQL en nomenclatura de negocio. Dashboard ejecutivo interactivo con visualización de gasto por cuenta y centro de costo, alertas automáticas por desviación presupuestal y manual de uso para el equipo del área.

**Esta fase constituye el producto mínimo viable (MVP) del proyecto y el entregable principal del contrato de consultoría.**

**Entregables:**
- Esquema estrella documentado (fct_gastos + 4 dimensiones)
- Vistas semánticas SQL en nomenclatura de negocio
- Dashboard ejecutivo funcional con alertas de desviación
- Manual de uso para el equipo (lenguaje no técnico)

---

### Fase 5 — Modelo predictivo de clasificación *(condicional)*
**15 de septiembre – 28 de septiembre de 2026**

Evaluación de la viabilidad de un modelo supervisado de clasificación de imputaciones contables. Esta fase se ejecuta únicamente si los datos históricos contienen registros etiquetados de imputaciones incorrectas. En caso contrario, el periodo se destina a consolidar la Fase 4 e iniciar la Fase 6 con mayor margen.

**Entregables:**
- Documento de viabilidad técnica con justificación basada en los datos
- Modelo de clasificación documentado con métricas *(si viable)*

---

### Fase 6 — Capa de IA generativa (RAG)
**29 de septiembre – 13 de octubre de 2026**

Diseño e implementación de una capa de consulta en lenguaje natural sobre los datos del área, que permita al equipo obtener respuestas sobre el comportamiento de gasto sin acceder directamente al sistema ERP. La arquitectura se documenta para integración futura independientemente del nivel de implementación alcanzado.

**Entregables:**
- Arquitectura RAG documentada con hoja de ruta de integración
- Demo funcional de al menos un caso de uso *(si el tiempo lo permite)*
- Sistema integrado end-to-end

---

## Notas sobre el cronograma

**MVP:** La Fase 4 es el entregable mínimo comprometido. Las Fases 5 y 6 amplían el alcance dentro del mismo periodo contractual.

**Fase 5 condicional:** Su ejecución depende de la disponibilidad de datos históricos etiquetados en la fuente. La decisión se toma al inicio de la Fase 1 una vez inspeccionados los datos reales.

**Riesgo crítico:** La entrega de los datos históricos del sistema ERP está acordada para antes del 30 de junio de 2026. Un retraso en esta entrega comprime directamente el tiempo disponible para todas las fases de desarrollo.

**Vacaciones de agosto:** El periodo vacacional de agosto se contempla como ventana de trabajo intensivo que puede absorber retrasos de fases anteriores sin comprometer la entrega del MVP en septiembre.
