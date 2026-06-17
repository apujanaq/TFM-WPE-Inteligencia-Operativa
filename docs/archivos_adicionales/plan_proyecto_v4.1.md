# Plan de proyecto v4 — Sistema de inteligencia operativa WPE

**Autor:** Alejandro Pujana Quintero · Economista · MSc Data Science & IA (en curso)  
**Rol:** Consultor  
**Contexto:** Trabajo de Fin de Máster · MSc Data Science & IA · Evolve Academy · Promoción Enero 2026  
**Cliente:** Workplace Experience · Tabacalera multinacional · Clúster CCA  
**Versión:** 4.1 · Junio 2026  
**Repositorio:** [github.com/apujanaq/TFM-WPE-Inteligencia-Operativa](https://github.com/apujanaq/TFM-WPE-Inteligencia-Operativa)  
**Fecha de inicio:** 1 de julio de 2026  
**Fecha de cierre estimada:** 13 de octubre de 2026  

---

## Objetivo general

Diseñar e implementar un sistema de inteligencia operativa para el área de Workplace Experience de la tabacalera, que automatice el control presupuestal mediante un pipeline de datos, un modelo de detección de anomalías y un dashboard ejecutivo, con capacidad de escalar al clúster regional CCA y de incorporar una capa de inteligencia artificial generativa para consulta en lenguaje natural.

---

## Objetivos específicos

| # | Objetivo | Fase | Indicadores asociados |
|---|----------|------|----------------------|
| OE-1 | Adquirir, validar y documentar los datos históricos de la transacción de gastos del área (2024–2025), estableciendo la línea base del proceso manual actual y el diccionario de datos del sistema | F1 | IG-2, F1-1, F1-2 |
| OE-2 | Construir un pipeline de preprocesamiento reproducible que limpie, normalice y estructure los datos históricos en esquema estrella SQLite, con todas las decisiones de tratamiento documentadas | F2 | IG-2, F2-1, F2-2 |
| OE-3 | Desarrollar un análisis exploratorio de los datos de gasto e implementar un modelo de detección de anomalías evaluado con métricas técnicas y validado con criterio de negocio del área | F3 | F3-1, F3-2 |
| OE-4 | Construir un modelo semántico sobre los datos estructurados y un dashboard ejecutivo funcional que permita visualizar el control presupuestal sin intervención técnica, con alertas automáticas por desviación | F4 | IG-1, IG-3, F4-1, F4-2 |
| OE-5 | Evaluar la viabilidad de un modelo predictivo de clasificación de imputaciones contables y, de ser viable, entrenarlo, evaluarlo y documentarlo como capacidad adicional del sistema | F5 | F5-1, F5-2 |
| OE-6 | Explorar el diseño de una capa de consulta en lenguaje natural sobre los datos del área y documentar su arquitectura para integración futura | F6 | F6-1, F6-2 |
| OE-7 | Mantener documentación estructurada y trazable de cada fase en el repositorio, como base de conocimiento del sistema y soporte para la capa generativa | Transversal | IG-4 |

---

## Indicadores de éxito

### Indicadores generales

| Código | Indicador | Definición | Meta verificable |
|--------|-----------|------------|-----------------|
| IG-1 | Reducción de tiempo en cierre mensual | Tiempo del proceso manual actual vs. tiempo con el sistema automatizado | Reducción ≥ 50% sobre la línea base documentada en Fase 1 |
| IG-2 | Cobertura del histórico procesado | % de registros 2024–2025 correctamente cargados, limpios y disponibles | ≥ 98% sin pérdida de información |
| IG-3 | Disponibilidad del dashboard | El sistema ejecuta una demo en vivo sin intervención técnica | Demo funcional validada con el área antes de la defensa |
| IG-4 | Documentación acumulada | Cada fase cierra con documentación estructurada en el repositorio | 100% de fases documentadas antes del cierre del proyecto |

### Indicadores específicos por fase

| Código | Fase | Indicador | Meta verificable |
|--------|------|-----------|-----------------|
| F1-1 | F1 | % de campos del diccionario documentados con descripción de negocio | 100% de campos clave documentados y validados |
| F1-2 | F1 | Línea base del tiempo del proceso manual documentada | Sí — entregada como parte del informe de recepción |
| F2-1 | F2 | % de registros reproducidos sin pérdida tras el pipeline | ≥ 98% |
| F2-2 | F2 | % de campos con decisión de tratamiento documentada | 100% — decisiones en el repositorio |
| F3-1 | F3 | Reporte de EDA entregado en lenguaje ejecutivo | Sí — revisado por el consultor antes de compartir |
| F3-2 | F3 | Modelo de anomalías evaluado con métricas documentadas | Sí — métricas + sesión de validación con el área |
| F4-1 | F4 | Modelo semántico documentado con vistas SQL en nomenclatura de negocio | Sí — disponible en repositorio |
| F4-2 | F4 | Dashboard ejecutable en demo sin intervención técnica | Sí — validado con el área antes de la defensa |
| F5-1 | F5 | Viabilidad del modelo de clasificación confirmada con evidencia | Sí / No — decisión documentada con justificación |
| F5-2 | F5 | Si viable: modelo entrenado, evaluado y documentado | Sí — métricas + decisión de umbral registradas |
| F6-1 | F6 | Arquitectura RAG documentada en repositorio | Sí — disponible como guía para integración futura |
| F6-2 | F6 | Si el tiempo lo permite: demo de al menos un caso de uso en lenguaje natural | Sí — caso documentado y validado con el área |

---

## Arquitectura del sistema

### Arquitectura Medallion

| Capa | Contenido | Herramienta | Fase |
|------|-----------|-------------|------|
| Bronze | CSV crudos exportados del ERP, sin modificar | Sistema de archivos local / SharePoint | F1 |
| Silver | Datos limpios, normalizados, códigos mapeados | SQLite (`wpe_silver.db`) · Python · VS Code | F2 |
| Gold | Esquema estrella + vistas semánticas | SQLite (`wpe_gold.db`) · SQL · VS Code | F4 |
| Consumo | Dashboard, modelo, RAG | Streamlit · Scikit-learn · LangChain | F3–F6 |

**Estrategia de almacenamiento:** SQLite local durante todo el desarrollo (Opciones B). Al finalizar el proyecto, el sistema se documenta para migración al ecosistema Microsoft corporativo (Microsoft Fabric / Azure SQL) si el área decide integrarlo formalmente. El código del pipeline es agnóstico al destino de almacenamiento: cambiar de SQLite a Fabric requiere modificar únicamente la capa de carga, no la lógica de transformación.

### Modelo de datos — Esquema estrella (Gold)

```
dim_cuenta ──────┐
dim_ceco ────────┤
                 ├──── fct_gastos (tabla de hechos central)
dim_proveedor ───┤
dim_fecha ───────┘
```

| Tabla | Tipo | Campos clave | Granularidad |
|-------|------|-------------|--------------|
| `fct_gastos` | Hechos | id_transaccion, importe, moneda, clase_documento, id_cuenta, id_ceco, id_proveedor, id_fecha, anio_contable | Una fila por movimiento de gasto |
| `dim_cuenta` | Dimensión | id_cuenta, codigo_cuenta, nombre_cuenta, categoria_cuenta | Una fila por cuenta contable |
| `dim_ceco` | Dimensión | id_ceco, codigo_ceco, codigo_pais, codigo_area, codigo_sede, nombre_pais, nombre_area, nombre_sede, periodo_clusterizacion | Una fila por centro de costo |
| `dim_proveedor` | Dimensión | id_proveedor, codigo_proveedor, nombre_proveedor | Una fila por proveedor |
| `dim_fecha` | Dimensión | id_fecha, fecha, anio, mes, trimestre, periodo_cierre, es_cierre_anual | Una fila por fecha única |

**Nota sobre clusterización:** Durante 2024–2025 ocurrió la integración del clúster Andino al CCA. Algunos CeCos cambiaron de código sin que el gasto subyacente cambiara. El campo `periodo_clusterizacion` en `dim_ceco` permite aislar este efecto en el análisis de series temporales.

---

## Fases del proyecto

### Fase 1 — Adquisición y validación de datos

**Fechas:** 1 julio – 15 julio 2026  
**Objetivo:** OE-1  

#### Herramientas

| Herramienta | Rol en esta fase |
|-------------|-----------------|
| SAP GD20 | Fuente de extracción de datos históricos 2024–2025 |
| Excel | Inspección visual inicial de los archivos exportados |
| SharePoint / Teams | Canal de recepción y almacenamiento de archivos desde el área |
| Python · VS Code | Script de auditoría inicial sobre los CSV recibidos |
| GitHub | Control de versiones — primer commit del repositorio |

#### Flujo de trabajo

```
SAP GD20
  │  Exportación manual por año contable
  │  Formato: CSV · Encoding: a confirmar · Separador: a confirmar
  ▼
/data/bronze/
  ├── gd20_raw_2024.csv   ← archivo sin modificar
  └── gd20_raw_2025.csv   ← archivo sin modificar
  │
  │  audit.py (VS Code)
  │  - Carga con pd.read_csv()
  │  - Inspección: shape, dtypes, nulos, duplicados, rangos
  │  - Output: informe de calidad en /reports/audit_report.md
  ▼
Entregable: diccionario de datos + informe de calidad + línea base proceso manual
```

#### Actividades

- `[HUMANO]` Coordinar extracción GD20 con el área — histórico 2024–2025, alcance de cuentas y CeCos
- `[HUMANO]` Documentar línea base del tiempo del proceso manual (entrevista con responsable del área)
- `[HUMANO]` Validar que los campos recibidos corresponden a la estructura conocida de la práctica
- `[HUMANO]` Firmar acuerdo de confidencialidad y establecer protocolo de manejo de datos
- `[HUMANO]` Acordar protocolo de actualización incremental post-proyecto
- `[COLABORACIÓN]` Diccionario de datos: IA genera estructura base, consultor completa descripción de negocio
- `[IA]` `audit.py` — script de auditoría: nulos, duplicados, rangos atípicos, tipos de datos, encoding

#### Problemas de calidad anticipados en esta fase

- Nombres de columna en nomenclatura SAP (alemán/inglés técnico) — no descriptivos
- Formato de fecha no estándar (DD.MM.YYYY)
- Importes con formato europeo (punto como separador de miles, coma como decimal)
- Volumen real de registros desconocido hasta recibir los archivos
- Presencia y formato del campo de proveedor a confirmar

#### Entregables

- `/data/bronze/gd20_raw_2024.csv` y `gd20_raw_2025.csv`
- `/reports/audit_report.md` — informe de calidad inicial
- `/docs/data_dictionary_v1.md` — diccionario de datos con descripción de negocio
- `/docs/baseline_proceso_manual.md` — línea base del tiempo del cierre mensual

> **Riesgo crítico:** El retraso en la entrega de datos bloquea todas las fases siguientes. Fecha límite acordada: 30 de junio de 2026.

---

### Fase 2 — Preprocesamiento y almacenamiento

**Fechas:** 16 julio – 30 julio 2026  
**Objetivo:** OE-2  

#### Herramientas

| Herramienta | Rol en esta fase |
|-------------|-----------------|
| Python · Pandas · NumPy | Pipeline ETL: extracción, transformación y carga |
| SQLite | Base de datos Silver — `wpe_silver.db` |
| VS Code | Entorno de desarrollo del pipeline |
| GitHub | Control de versiones del pipeline |
| Jupyter Notebook | Exploración y validación de transformaciones antes de modularizar |

#### Flujo de trabajo

```
/data/bronze/gd20_raw_2024.csv
/data/bronze/gd20_raw_2025.csv
  │
  │  src/pipeline/extract.py
  │  - pd.read_csv() con encoding y separador correctos
  │  - Validación de estructura: columnas esperadas presentes
  ▼
  │  src/pipeline/transform.py
  │  - Renombrado de columnas a snake_case descriptivo
  │  - pd.to_datetime() con format='%d.%m.%Y'
  │  - Conversión de importes: str → float (limpieza formato europeo)
  │  - Decodificación de CeCo: split en codigo_pais, codigo_area, codigo_sede
  │  - Mapeo de códigos → nombres descriptivos (tablas de mapeo en /config/)
  │  - Flag is_kp: 1 si clase_documento == 'KP', 0 en otro caso
  │  - Flag is_externo: 1 si CeCo no pertenece al área WPE
  │  - Deduplicación: conservar registro más reciente por numero_documento
  │  - Criterio de validez: fecha + importe != 0 + codigo_cuenta + codigo_ceco completos
  ▼
  │  src/pipeline/load.py
  │  - Carga a SQLite: wpe_silver.db
  │  - Tabla única: gastos_silver (tabla plana normalizada)
  │  - Log de registros excluidos → /reports/quality_report.md
  ▼
/data/silver/wpe_silver.db
  └── gastos_silver (tabla plana limpia y decodificada)

Entregable: pipeline documentado + base Silver + informe de calidad post-procesamiento
```

#### Actividades

- `[HUMANO]` Definir reglas de negocio para la limpieza: qué cuentas incluir, cómo tratar KP y gastos externos
- `[HUMANO]` Validar que la base Silver refleja correctamente la lógica operativa del área
- `[HUMANO]` Acordar con el área la arquitectura de almacenamiento para la entrega final
- `[IA]` `extract.py` — carga de CSV con validación de estructura
- `[IA]` `transform.py` — todas las transformaciones documentadas arriba
- `[IA]` `load.py` — carga a SQLite con logging de registros excluidos
- `[COLABORACIÓN]` Tablas de mapeo en `/config/`: IA genera estructura, consultor completa con conocimiento del área
- `[COLABORACIÓN]` Anonimización: IA aplica transformaciones, consultor valida cumplimiento con Legal

#### Decisiones de tratamiento documentadas

| Problema | Decisión | Justificación |
|----------|----------|---------------|
| Clase documento KP | Mantener con flag `is_kp=1` | Son informativos para el modelo de anomalías y para el desbalance con contabilidad |
| Gastos de otras áreas | Mantener con flag `is_externo=1` | Permiten análisis de impacto sin contaminar agregados del área |
| Duplicados | Conservar registro más reciente por número de documento | Las correcciones retroactivas en SAP son intencionales |
| Cambios de CeCo por clusterización | Campo `periodo_clusterizacion` en dim_ceco | Aislar efecto estructural del análisis de series temporales |
| Registros inválidos | Excluir y loguear en quality_report | Trazabilidad completa de registros no procesados |

#### Entregables

- `/src/pipeline/extract.py`, `transform.py`, `load.py` — pipeline modular documentado
- `/data/silver/wpe_silver.db` — base de datos Silver
- `/config/mapeo_cuentas.csv`, `mapeo_cecos.csv` — tablas de decodificación
- `/reports/quality_report.md` — informe post-procesamiento con log de excluidos

---

### Fase 3 — Análisis exploratorio y modelado de anomalías

**Fechas:** 31 julio – 24 agosto 2026  
**Objetivo:** OE-3  

#### Herramientas

| Herramienta | Rol en esta fase |
|-------------|-----------------|
| Python · Pandas · Matplotlib · Seaborn | EDA y visualización exploratoria |
| Scikit-learn | Isolation Forest + Z-score multivariado |
| Jupyter Notebook | Análisis exploratorio interactivo |
| SQLite | Fuente de datos Silver para consultas |
| VS Code | Modularización del modelo en `src/models/anomaly_detection.py` |
| GitHub | Control de versiones de notebooks y módulos |

#### Flujo de trabajo

```
/data/silver/wpe_silver.db
  │
  │  notebooks/03_eda_gastos.ipynb
  │  - Consultas SQL sobre gastos_silver
  │  - Distribución de gasto por cuenta, CeCo, mes, clase_documento
  │  - Series temporales: evolución mensual 2024–2025
  │  - Análisis de KP: frecuencia y concentración por cuenta
  │  - Análisis del efecto clusterización en series temporales
  │  - Visualizaciones: Matplotlib / Seaborn
  ▼
  │  [HUMANO] Análisis preliminar de hallazgos con criterio de negocio
  │  [COLABORACIÓN] IA revisa lógica técnica y estructura lo que falta
  ▼
  │  notebooks/04_modelo_anomalias.ipynb
  │  - Feature engineering: importe_zscore por cuenta+CeCo+mes
  │  - Isolation Forest sobre fct_gastos (excluyendo KP y externos)
  │  - Evaluación: precision, recall sobre muestra validada por el área
  │  - Selección del modelo final con criterio de negocio
  ▼
  │  src/models/anomaly_detection.py
  │  - Modularización del modelo seleccionado
  │  - Docstrings NumPy completos
  │  - Output: tabla gold_anomalias_detectadas → wpe_gold.db
  ▼
/data/gold/wpe_gold.db
  └── gold_anomalias_detectadas

Entregable: reporte EDA ejecutivo + modelo validado + informe de resultados
```

#### Actividades

- `[HUMANO]` Definir qué constituye una anomalía real — umbral de negocio, no solo estadístico
- `[HUMANO]` Producir análisis preliminar del EDA con criterio de negocio
- `[HUMANO]` Traducir hallazgos técnicos a lenguaje ejecutivo para el área
- `[HUMANO]` Validar resultados del modelo contra conocimiento operativo del área
- `[COLABORACIÓN]` EDA: consultor produce análisis preliminar, IA revisa lógica técnica
- `[IA]` Implementación de Isolation Forest + Z-score en Scikit-learn
- `[COLABORACIÓN]` Selección del modelo: IA presenta métricas, consultor decide con criterio operativo

#### Punto ciego crítico

El modelo de anomalías se entrena sobre datos sin etiquetas (no supervisado). No existe una definición objetiva de "anomalía correcta" hasta que el área valide los resultados. Si el área no puede dedicar tiempo a esa sesión de validación, el modelo produce resultados técnicamente válidos pero sin interpretación de negocio confirmada. **Esta sesión de validación con el área es el entregable más crítico de esta fase y debe agendarse antes de arrancar el modelado.**

#### Entregables

- `notebooks/03_eda_gastos.ipynb` — análisis exploratorio documentado
- `notebooks/04_modelo_anomalias.ipynb` — desarrollo del modelo
- `src/models/anomaly_detection.py` — módulo del modelo
- `/reports/eda_report.md` — hallazgos en lenguaje ejecutivo
- `/reports/anomalias_report.md` — anomalías detectadas con interpretación de negocio

---

### Fase 4 — Modelo semántico y dashboard de BI

**Fechas:** 25 agosto – 14 septiembre 2026  
**Objetivo:** OE-4  

#### Herramientas

| Herramienta | Rol en esta fase |
|-------------|-----------------|
| SQLite | Construcción del esquema estrella y vistas semánticas (Gold) |
| SQL | Definición de tablas dimensión, tabla de hechos y vistas |
| Python · Streamlit | Dashboard interactivo sobre Gold |
| VS Code | Desarrollo del dashboard y las vistas SQL |
| GitHub | Control de versiones |
| Microsoft Fabric / Power BI | Opción de entrega al área post-TFM (no durante el desarrollo) |

#### Flujo de trabajo

```
/data/silver/wpe_silver.db (gastos_silver)
  │
  │  sql/01_schema_estrella.sql
  │  - CREATE TABLE fct_gastos
  │  - CREATE TABLE dim_cuenta, dim_ceco, dim_proveedor, dim_fecha
  │  - INSERT desde gastos_silver con transformaciones finales
  ▼
/data/gold/wpe_gold.db
  ├── fct_gastos
  ├── dim_cuenta
  ├── dim_ceco
  ├── dim_proveedor
  └── dim_fecha
  │
  │  sql/02_vistas_semanticas.sql
  │  - vw_gasto_por_cuenta: gasto mensual agrupado por cuenta y periodo
  │  - vw_gasto_por_ceco: gasto mensual agrupado por área y país
  │  - vw_desviaciones: comparativo gasto actual vs. promedio histórico
  │  - vw_anomalias: join fct_gastos + gold_anomalias_detectadas
  ▼
  │  bi/dashboard.py (Streamlit)
  │  - Página 1: resumen ejecutivo — gasto total, desviaciones, alertas
  │  - Página 2: detalle por cuenta contable
  │  - Página 3: detalle por CeCo / área / país
  │  - Página 4: anomalías detectadas con interpretación
  │  - Filtros: año, mes, cuenta, área
  ▼
Demo funcional validada con el área

Entregable: esquema Gold + vistas semánticas + dashboard + manual de uso
```

#### Actividades

- `[HUMANO]` Definir qué métricas necesita ver el equipo y qué decisión habilita cada vista
- `[HUMANO]` Sesión de validación del dashboard con responsable del área antes de la defensa
- `[COLABORACIÓN]` Modelo semántico: IA genera esquema y vistas, consultor valida nomenclatura de negocio
- `[IA]` Dashboard Streamlit sobre el modelo semántico
- `[IA]` Sistema de alertas por desviación de umbral presupuestal
- `[COLABORACIÓN]` Diseño de vistas: IA genera opciones, consultor selecciona por pertinencia operativa

#### Entregables

- `/sql/01_schema_estrella.sql` — definición del esquema Gold
- `/sql/02_vistas_semanticas.sql` — vistas en nomenclatura de negocio
- `/data/gold/wpe_gold.db` — base de datos Gold
- `bi/dashboard.py` — dashboard Streamlit funcional
- `/docs/manual_uso_wpe.md` — manual para usuario no técnico

> **MVP del proyecto:** Esta fase es el entregable mínimo comprometido en el contrato de consultoría.

---

### Fase 5 — Modelo predictivo de clasificación de imputaciones

**Fechas:** 15 septiembre – 28 septiembre 2026  
**Objetivo:** OE-5  
**Condición:** Solo se ejecuta si los datos históricos contienen registros etiquetados de imputaciones incorrectas.

#### Herramientas

| Herramienta | Rol en esta fase |
|-------------|-----------------|
| Python · Scikit-learn · XGBoost · Pandas | Entrenamiento y evaluación del modelo |
| Jupyter Notebook | Desarrollo experimental del modelo |
| SQLite | Fuente de datos Gold para features |
| VS Code | Modularización en `src/models/classification.py` |
| GitHub | Control de versiones |

#### Flujo de trabajo

```
/data/gold/wpe_gold.db (fct_gastos)
  │
  │  [HUMANO] Verificar existencia de histórico etiquetado
  │  Si no existe → documentar como trabajo futuro y liberar tiempo para F6
  ▼ (si viable)
  │  notebooks/05_modelo_clasificacion.ipynb
  │  - Feature engineering: importe, cuenta, CeCo, mes, clase_documento, is_kp
  │  - Train/test split estratificado por cuenta
  │  - Logistic Regression / Random Forest / XGBoost
  │  - Métricas: precision, recall, F1 — decisión de umbral con el área
  ▼
  │  src/models/classification.py
  │  - Módulo del modelo seleccionado
  │  - Docstrings NumPy completos
  ▼
Entregable: documento de viabilidad + modelo documentado (si viable)
```

#### Entregables

- `/reports/viabilidad_clasificacion.md` — decisión documentada con evidencia
- `notebooks/05_modelo_clasificacion.ipynb` — si viable
- `src/models/classification.py` — si viable

---

### Fase 6 — Capa de IA generativa (RAG)

**Fechas:** 29 septiembre – 13 octubre 2026  
**Objetivo:** OE-6  

#### Herramientas

| Herramienta | Rol en esta fase |
|-------------|-----------------|
| Python · LangChain | Implementación de la capa RAG |
| API LLM (Anthropic Claude o equivalente) | Modelo de lenguaje para generación de respuestas |
| SQLite | Fuente de datos Gold para el agente SQL |
| Streamlit | Integración de la interfaz de consulta al dashboard |
| VS Code | Desarrollo de `src/rag/query_engine.py` |
| GitHub | Control de versiones |

#### Flujo de trabajo

```
/data/gold/wpe_gold.db + vistas semánticas
  │
  │  src/rag/query_engine.py
  │  - SQL agent sobre wpe_gold.db
  │  - Contexto del sistema: descripción de las vistas semánticas
  │  - LangChain: herramienta de consulta SQL + LLM para generación
  │  - Input: pregunta en lenguaje natural
  │  - Output: respuesta en lenguaje natural con datos reales del área
  ▼
  │  bi/dashboard.py (actualización)
  │  - Página adicional: consulta en lenguaje natural
  │  - Input box + botón de envío
  │  - Respuesta generada + query SQL ejecutada (trazabilidad)
  ▼
Entregable: arquitectura RAG documentada + demo si el tiempo lo permite
```

#### Actividades

- `[HUMANO]` Definir casos de uso reales: qué preguntas haría el equipo en el día a día
- `[HUMANO]` Validar que las respuestas son correctas en términos de negocio
- `[HUMANO]` Decidir alcance final según tiempo disponible al cerrar Fase 4
- `[IA]` `query_engine.py` — SQL agent + LLM
- `[IA]` Integración de interfaz de consulta en el dashboard
- `[COLABORACIÓN]` Pruebas de calidad: IA genera casos de prueba, consultor evalúa con conocimiento del área

#### Punto ciego crítico

La capa RAG responde en lenguaje natural pero las respuestas dependen de la calidad del modelo semántico de Fase 4. Si las vistas semánticas no están bien nombradas y documentadas, el agente SQL genera consultas incorrectas aunque el LLM formule frases coherentes. **La documentación de las vistas en `/docs/` no es opcional: es el contexto del sistema que alimenta al agente.**

#### Entregables

- `src/rag/query_engine.py` — motor de consulta RAG
- `/docs/arquitectura_rag.md` — arquitectura documentada para integración futura
- Demo funcional de al menos un caso de uso (si el tiempo lo permite)

---

## Cronograma

| Fase | Componente | Fecha de inicio | Fecha de cierre | Entregable principal |
|------|------------|----------------|----------------|----------------------|
| Fase 1 | Adquisición y validación de datos | 1 de julio de 2026 | 15 de julio de 2026 | Diccionario de datos · Informe de calidad · Línea base proceso manual |
| Fase 2 | Preprocesamiento y almacenamiento | 16 de julio de 2026 | 30 de julio de 2026 | Base Silver · Pipeline documentado · Informe post-procesamiento |
| Fase 3 | EDA y modelado de anomalías | 31 de julio de 2026 | 24 de agosto de 2026 | Reporte EDA ejecutivo · Modelo validado · Informe de resultados |
| Fase 4 | Modelo semántico y dashboard de BI | 25 de agosto de 2026 | 14 de septiembre de 2026 | Dashboard funcional · Esquema Gold · Manual de uso — **MVP** |
| Fase 5 | Modelo predictivo de clasificación | 15 de septiembre de 2026 | 28 de septiembre de 2026 | Documento de viabilidad · Modelo documentado *(condicional)* |
| Fase 6 | Capa de IA generativa (RAG) | 29 de septiembre de 2026 | 13 de octubre de 2026 | Arquitectura RAG documentada · Demo *(si el tiempo lo permite)* |

**Fecha de inicio:** 1 de julio de 2026  
**Fecha de cierre estimada:** 13 de octubre de 2026  

---

## Protocolo de colaboración

> Pendiente de actualización al finalizar el curso AI Fluency. La configuración de VS Code + Claude Code + archivos de contexto + canary system se define en sesión dedicada antes de las fases de código.

### Descripción del producto
Cada tarea genera el output que corresponde a su naturaleza: `.md` para documentación y trazabilidad, `.py` para lógica ejecutable, `.ipynb` para exploración y análisis, lenguaje ejecutivo cuando el destinatario es el área. Docstrings NumPy y comentarios en líneas no triviales son obligatorios en todo el código sin excepción.

### Descripción del proceso
Siempre: explicación lógica conceptual antes del código, con analogías cuando aporten claridad estructural. El código sin comprensión de su lógica no permite hacer cambios estructurales durante el proyecto ni defenderlo ante un tribunal.

### Descripción del performance
Rol: colega senior de datos con criterio de producto a largo plazo. Estricto con el "por qué" de cada decisión técnica. Estándar permanente: ¿este sistema sirve cuando el consultor no esté presente? Cada decisión técnica debe quedar documentada con su justificación de negocio, no solo técnica.

---

## Notas sobre delegación

El valor diferencial del consultor no está en la escritura de código sino en las decisiones que ninguna IA puede tomar:

- Qué constituye una anomalía operativa real versus ruido estadístico
- Qué métricas habilitan decisiones reales para el equipo WPE
- Si los resultados del sistema son correctos en términos del negocio del área
- Cómo traducir hallazgos técnicos a lenguaje ejecutivo para el área
- Qué alcance tiene sentido en función del tiempo real disponible

Estas decisiones requieren el conocimiento operativo adquirido durante la práctica profesional (julio–diciembre 2025) y no son replicables por un agente externo sin ese contexto.

---

## Historial de versiones

| Versión | Fecha | Cambios principales |
|---------|-------|---------------------|
| v1.0 | Jun 2026 | Plan inicial — 5 fases, actividades y entregables base |
| v2.0 | Jun 2026 | Arquitectura batch+modular, modelo semántico, Fase 5 piloto, RAG como Fase 6 |
| v2.1 | Jun 2026 | Protocolo de colaboración general, Fase 5 a 2 semanas |
| v2.2 | Jun 2026 | Protocolo consolidado, ecosistema Microsoft documentado |
| v3.0 | Jun 2026 | Objetivos, indicadores, herramientas por fase, cronograma con fechas reales |
| v4.0 | Jun 2026 | Flujo de trabajo técnico por fase, esquema estrella, decisiones de tratamiento de datos, puntos ciegos críticos identificados, arquitectura SQLite→Fabric documentada |

---

## Uso de inteligencia artificial — AI Diligence

Este proyecto fue desarrollado en colaboración con Claude (Anthropic) como herramienta de IA generativa. El detalle completo del uso de IA, las implicaciones de privacidad, el proceso de verificación de outputs y la apropiación de responsabilidades se documenta en:

**`AI_DILIGENCE_STATEMENT.md`** — disponible en la raíz del repositorio.

Este proyecto aplica el marco de AI Fluency de Anthropic (Delegación, Descripción, Discernimiento, Diligencia) como metodología de colaboración con IA. El historial de versiones del plan de proyecto es evidencia directa del proceso iterativo de revisión y refinamiento aplicado a cada output generado con colaboración de IA.

| Versión | Fecha | Cambios principales |
|---------|-------|---------------------|
| v1.0 | Jun 2026 | Plan inicial — 5 fases, actividades y entregables base |
| v2.0 | Jun 2026 | Arquitectura batch+modular, modelo semántico, Fase 5 piloto, RAG como Fase 6 |
| v2.1 | Jun 2026 | Protocolo de colaboración general, Fase 5 a 2 semanas |
| v2.2 | Jun 2026 | Protocolo consolidado, ecosistema Microsoft documentado |
| v3.0 | Jun 2026 | Objetivos, indicadores, herramientas por fase, cronograma con fechas reales |
| v4.0 | Jun 2026 | Flujo de trabajo técnico por fase, esquema estrella, decisiones de tratamiento, puntos ciegos |
| v4.1 | Jun 2026 | AI Diligence Statement incorporado, referencia al repositorio |
