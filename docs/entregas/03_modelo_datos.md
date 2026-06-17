# Entrega 3 — Diseño del modelo de datos y capa Gold del proyecto

**Autor:** Alejandro Pujana Quintero · Economista · MSc Data Science & IA (en curso)  
**Programa:** MSc Data Science & Inteligencia Artificial · Evolve Academy · Promoción Enero 2026  
**Repositorio:** [github.com/apujanaq/TFM-WPE-Inteligencia-Operativa](https://github.com/apujanaq/TFM-WPE-Inteligencia-Operativa)  
**Fecha:** Junio 2026  

> **Nota sobre el estado de este documento:** Esta entrega es una versión preliminar. Los datos reales del sistema ERP corporativo no están disponibles en esta fase del proyecto. El modelo de datos, el diccionario de campos, la capa Gold y los problemas de calidad se definen con base en el conocimiento operativo adquirido durante la práctica profesional en el área (julio–diciembre 2025). Todas las decisiones aquí documentadas se revisarán y validarán en la Fase 1 del proyecto, una vez recibidos los archivos históricos. Se agradece retroalimentación del profesor sobre la estructura propuesta antes de ejecutar la implementación.

---

## 1. Resumen de la idea y datos del proyecto

El área de Workplace Experience (WPE) de una tabacalera multinacional gestiona el control presupuestal de facilidades y flota mediante un proceso completamente manual: descarga mensual de la transacción de gastos desde el sistema ERP corporativo, depuración en Excel, construcción de tablas dinámicas y revisión cuenta por cuenta. Este proceso consume entre cuatro y ocho horas por cierre, genera inconsistencias entre el registro del área y el equipo contable regional, y elimina cualquier capacidad de detección temprana de desviaciones.

La solución consiste en construir un sistema de inteligencia operativa sobre los datos históricos de esa transacción de gastos. El flujo es: extracción estructurada de datos → pipeline de limpieza y normalización → modelo de detección de patrones atípicos → dashboard ejecutivo con alertas automáticas → capa de consulta en lenguaje natural. La arquitectura está diseñada para ser modular y replicable al clúster regional sin rediseño.

La fuente de datos es única: la transacción GD20 del sistema ERP corporativo (SAP), exportada en formato CSV por año contable. Cada archivo contiene el registro transaccional de todos los movimientos de gasto del área para ese periodo, con campos codificados en nomenclatura interna de SAP que requieren mapeo a lenguaje de negocio antes de poder ser analizados.

---

## 2. Tecnología y formato de almacenamiento

La decisión de almacenamiento sigue una lógica de progresión por capas, coherente con el volumen de datos y con las herramientas disponibles durante el desarrollo.

**Bronze:** CSV sin modificar, tal como salen de SAP. Dos archivos: `gd20_raw_2024.csv` y `gd20_raw_2025.csv`. Esta capa no se toca. Su función es garantizar trazabilidad absoluta hacia el dato original. Si algo sale mal en cualquier fase posterior, siempre existe un punto de retorno limpio.

**Silver y Gold:** SQLite como base de datos relacional local. SQLite vive en un único archivo, no requiere servidor, acepta consultas SQL estándar y es perfectamente suficiente para un volumen estimado de entre 10.000 y 50.000 registros por año. Permite construir el esquema estrella, ejecutar joins entre tablas, aplicar vistas semánticas y alimentar tanto el modelo de anomalías como el dashboard desde una misma fuente estructurada.

La elección de SQLite sobre alternativas como PostgreSQL responde a tres criterios concretos: el volumen de datos no justifica infraestructura de servidor, el desarrollo es individual y no requiere acceso concurrente, y la portabilidad de un archivo único facilita la entrega y documentación del sistema. Si el área decide integrar el sistema formalmente en el ecosistema Microsoft corporativo, la migración a una solución en la nube (Azure SQL, Microsoft Fabric) es el paso natural posterior al TFM, no durante él.

No se utilizan formatos Parquet ni JSON en este proyecto. Parquet tiene sentido para volúmenes de datos en el orden de millones de registros con necesidad de compresión columnar. JSON aplica para estructuras anidadas o APIs. Ninguno de los dos casos aplica aquí.

---

## 3. Estructura de capas de datos

El proyecto sigue la arquitectura Medallion estándar, adaptada al contexto del proyecto:

```
data/
├── bronze/
│   ├── gd20_raw_2024.csv       # Exportación SAP sin modificar — año contable 2024
│   └── gd20_raw_2025.csv       # Exportación SAP sin modificar — año contable 2025
├── silver/
│   └── wpe_silver.db           # SQLite — datos limpios, normalizados, codigos mapeados
└── gold/
    └── wpe_gold.db             # SQLite — esquema estrella + vistas semánticas
```

**Bronze** recibe los archivos tal como los exporta SAP: columnas con nombres internos en alemán o inglés técnico, códigos numéricos sin descripción, posibles comas en campos de texto, encoding variable. No se aplica ninguna transformación. La integridad de esta capa es no negociable.

**Silver** es donde ocurre la transformación: renombrado de columnas a nombres descriptivos, decodificación de códigos SAP (centros de costo, cuentas contables, clases de documento), estandarización de formatos de fecha, tratamiento de nulos, eliminación de duplicados, separación de los componentes del código CeCo (país, área, sede) en campos independientes. El resultado es una tabla limpia y legible, pero aún plana.

**Gold** es donde se construye el modelo relacional: tabla de hechos central (`fct_gastos`) con claves foráneas hacia las cuatro tablas de dimensiones (`dim_cuenta`, `dim_ceco`, `dim_proveedor`, `dim_fecha`). Encima de este esquema se crean vistas SQL con nomenclatura de negocio que son las que consume directamente el dashboard y el modelo de anomalías. La capa RAG también se alimenta de Gold, pero como capa de lectura que no modifica la estructura.

---

## 4. Definición de la capa Gold

La capa Gold está compuesta por una tabla de hechos y cuatro tablas de dimensión. El esquema estrella garantiza que cada tabla tiene una responsabilidad única y que las relaciones se establecen exclusivamente por claves numéricas, nunca por texto libre.

### 4.1 Tabla de hechos — `fct_gastos`

| Campo | Tipo | Descripción | Obligatorio |
|-------|------|-------------|-------------|
| `id_transaccion` | INTEGER (PK) | Clave primaria generada — identificador único de cada movimiento | Sí |
| `fecha_contabilizacion` | DATE | Fecha en que el movimiento quedó registrado en SAP | Sí |
| `importe` | FLOAT | Valor del movimiento en moneda local | Sí |
| `moneda` | TEXT | Código de moneda (COP, USD, etc.) | Sí |
| `clase_documento` | TEXT | Tipo de transacción: SA (reclasificación), BWE (recepción), KP (irregular) | Sí |
| `numero_documento` | TEXT | Número de documento SAP asociado a la transacción | Sí |
| `id_cuenta` | INTEGER (FK) | Referencia a `dim_cuenta` | Sí |
| `id_ceco` | INTEGER (FK) | Referencia a `dim_ceco` | Sí |
| `id_proveedor` | INTEGER (FK) | Referencia a `dim_proveedor` | No |
| `id_fecha` | INTEGER (FK) | Referencia a `dim_fecha` | Sí |
| `anio_contable` | INTEGER | Año del archivo fuente (2024 o 2025) | Sí |

**Granularidad:** una fila por movimiento de gasto registrado en SAP.  
**Volumen estimado:** entre 10.000 y 50.000 registros por año. Total histórico: 20.000–100.000 registros.  
**Variable objetivo para el modelo de anomalías:** `importe` en combinación con `id_cuenta`, `id_ceco` y `clase_documento`.  
**Consumo posterior:** EDA (Fase 3), modelo de detección de anomalías (Fase 3), dashboard (Fase 4), RAG (Fase 6).

### 4.2 Dimensión — `dim_cuenta`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id_cuenta` | INTEGER (PK) | Clave primaria |
| `codigo_cuenta` | TEXT | Código numérico de la cuenta en SAP |
| `nombre_cuenta` | TEXT | Descripción en lenguaje de negocio |
| `categoria_cuenta` | TEXT | Agrupación: Utilities / Buildings & Maintenance / Cafetería / Flota / Otros |

**Cuentas identificadas preliminarmente:** Cafetería, Property Taxes, Electricity, Water, General Charges, Buildings & Maintenance, Clean Expenses, Operating & Short Leasing, Variable Lease Cost, Mail, Security, Other Outsources.

### 4.3 Dimensión — `dim_ceco`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id_ceco` | INTEGER (PK) | Clave primaria |
| `codigo_ceco` | TEXT | Código completo de 10 dígitos de SAP |
| `codigo_pais` | TEXT | Primeros 4 dígitos (ej. 1764 = Colombia, 1724 = Ecuador) |
| `codigo_area` | TEXT | Dígitos 5–7 (ej. 7965 = Facilidades, 409 = otra área) |
| `codigo_sede` | TEXT | Últimos 3 dígitos (ej. 510 = Regional 1) |
| `nombre_pais` | TEXT | Nombre del país |
| `nombre_area` | TEXT | Nombre del área funcional |
| `nombre_sede` | TEXT | Nombre de la sede o regional |
| `periodo_clusterizacion` | TEXT | Indica si el CeCo pertenece al esquema pre o post clusterización CCA |

> **Nota crítica:** Durante el periodo 2024–2025 ocurrió la integración del clúster Andino al clúster CCA. Esto implicó cambios de códigos de CeCo para algunas áreas (ej. código 415 → 450 para ventas). El campo `periodo_clusterizacion` permite identificar y aislar este efecto estructural en el análisis de series temporales, evitando que el modelo de anomalías interprete un cambio de código como una anomalía de gasto.

### 4.4 Dimensión — `dim_proveedor`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id_proveedor` | INTEGER (PK) | Clave primaria |
| `codigo_proveedor` | TEXT | Código de proveedor en SAP |
| `nombre_proveedor` | TEXT | Nombre del proveedor (a confirmar disponibilidad en GD20) |

### 4.5 Dimensión — `dim_fecha`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id_fecha` | INTEGER (PK) | Clave primaria |
| `fecha` | DATE | Fecha completa |
| `anio` | INTEGER | Año |
| `mes` | INTEGER | Mes (1–12) |
| `trimestre` | INTEGER | Trimestre (1–4) |
| `nombre_mes` | TEXT | Nombre del mes en español |
| `periodo_cierre` | TEXT | Etiqueta del cierre mensual (ej. "2024-M03") |
| `es_cierre_anual` | INTEGER | 1 si es diciembre, 0 en otro caso |

---

## 5. Relaciones entre datos

El modelo estrella concentra todas las relaciones en la tabla `fct_gastos`. Las dimensiones no se relacionan entre sí directamente.

```
dim_cuenta.id_cuenta     1 ──── N  fct_gastos.id_cuenta
dim_ceco.id_ceco         1 ──── N  fct_gastos.id_ceco
dim_proveedor.id_proveedor 1 ── N  fct_gastos.id_proveedor
dim_fecha.id_fecha       1 ──── N  fct_gastos.id_fecha
```

Todas las relaciones son 1:N. Una cuenta puede aparecer en muchos movimientos, pero cada movimiento tiene exactamente una cuenta. Lo mismo aplica para CeCo, proveedor y fecha.

Los joins en las consultas analíticas siempre parten de `fct_gastos` hacia las dimensiones, nunca al revés. Esto es lo que permite que el dashboard y el modelo accedan a información descriptiva (nombre del área, categoría de cuenta) sin duplicar esa información en la tabla de hechos.

El problema más probable al cruzar fuentes es la inconsistencia de códigos durante el periodo de clusterización. Un CeCo que existía con un código en 2024 puede aparecer con un código diferente en 2025 representando la misma entidad. Si no se controla en Silver, el join en Gold producirá filas sin match o duplicará entidades. Este riesgo está contemplado en la dimensión `dim_ceco` con el campo `periodo_clusterizacion`.

El proyecto trabaja con una única fuente de datos (GD20 de SAP). No hay joins entre fuentes externas. Esto simplifica el modelo pero también lo limita: toda la información disponible para el análisis viene de una sola transacción, lo que hace que la calidad del mapeo de códigos en Silver sea determinante para la calidad de cualquier resultado en Gold.

---

## 6. Diccionario de datos inicial

Este diccionario es preliminar. Los nombres de columna exactos en la exportación SAP se confirmarán en la Fase 1 cuando se reciban los archivos. Los campos aquí documentados se basan en el conocimiento operativo del proceso de descarga GD20.

| Campo (Bronze) | Campo (Silver/Gold) | Tipo | Descripción | Obligatorio | Observaciones |
|----------------|---------------------|------|-------------|-------------|---------------|
| *(nombre SAP)* | `fecha_contabilizacion` | DATE | Fecha del registro contable | Sí | Puede venir en formato DD.MM.YYYY — requiere conversión |
| *(nombre SAP)* | `importe` | FLOAT | Valor del movimiento | Sí | Puede incluir separador de miles con punto — requiere limpieza |
| *(nombre SAP)* | `moneda` | TEXT | Código de moneda | Sí | Esperado: COP, USD |
| *(nombre SAP)* | `codigo_cuenta` | TEXT | Cuenta contable SAP | Sí | Código numérico — requiere mapeo a nombre descriptivo |
| *(nombre SAP)* | `codigo_ceco` | TEXT | Centro de costo SAP — 10 dígitos | Sí | Requiere descomposición en país, área y sede |
| *(nombre SAP)* | `clase_documento` | TEXT | Tipo de transacción | Sí | SA / BWE / KP — requiere mapeo a descripción |
| *(nombre SAP)* | `numero_documento` | TEXT | Número de documento SAP | Sí | Puede usarse para detectar duplicados |
| *(nombre SAP)* | `codigo_proveedor` | TEXT | Proveedor asociado | No | Disponibilidad a confirmar en Fase 1 |
| *(nombre SAP)* | `descripcion_posicion` | TEXT | Texto libre de la posición | No | Campo de baja estandarización — uso limitado en modelos |

---

## 7. Problemas de calidad esperados

Los problemas aquí descritos se anticipan con base en el conocimiento del proceso de descarga GD20 y del comportamiento observado durante la práctica profesional. No son genéricos: están anclados al caso específico.

**Codificación SAP sin descripción.** Todos los campos clave (cuenta, CeCo, proveedor) vienen como códigos numéricos. Sin el mapeo correcto, los datos son ininterpretables. El diccionario de mapeo no existe como archivo formal — se construirá en Fase 1 a partir del conocimiento del área.

**Clase de documento KP.** Esta clase representa movimientos cuyo origen no está claro: pueden ser facturas que no corresponden al área, ajustes contables externos o errores de imputación. Históricamente son la principal fuente de desbalance con el equipo contable de Argentina. Su tratamiento requiere criterio de negocio, no solo una regla técnica.

**Cambios de códigos por clusterización.** Durante 2024–2025 ocurrió la integración del clúster Andino al CCA. Algunos CeCos cambiaron de código sin que el gasto subyacente cambiara. Si no se controla, el análisis de series temporales interpretará estos cambios como variaciones de gasto cuando en realidad son cambios de nomenclatura.

**Gastos de otras áreas en cuentas WPE.** Existen movimientos que caen en las cuentas del área pero que no fueron generados por WPE. Identificarlos requiere conocimiento operativo del área y no puede automatizarse completamente con reglas técnicas.

**Importes con formato europeo.** SAP puede exportar valores con punto como separador de miles y coma como decimal (ej. `1.234,56`). Python lee esto como texto, no como número. Requiere transformación explícita en Silver.

**Fechas en formato no estándar.** El formato de fecha esperado en la exportación SAP es DD.MM.YYYY, que no es reconocido automáticamente por Pandas como fecha. Requiere conversión explícita con `pd.to_datetime(..., format='%d.%m.%Y')`.

**Nulos en campos no obligatorios.** El campo de proveedor puede estar vacío en movimientos de reclasificación (SA). Esto es esperable y no constituye un error, pero debe documentarse como decisión de tratamiento en Silver.

**Duplicados por correcciones retroactivas.** SAP permite modificar registros de periodos anteriores dentro del mismo año contable. Esto puede generar registros duplicados con el mismo número de documento pero fechas distintas. La estrategia de deduplicación se definirá en Fase 1 cuando se inspeccione el volumen real.

---

## 8. Decisiones de limpieza y transformación previstas

Estas decisiones son hipótesis iniciales. Se revisarán y ajustarán en la Fase 2 una vez inspeccionados los datos reales.

**Renombrado de columnas.** Los nombres de columna de SAP se reemplazarán en Silver por nombres en español, descriptivos y en snake_case. La tabla de mapeo quedará documentada en el repositorio como referencia permanente.

**Decodificación de CeCo.** El código de 10 dígitos se separará en tres campos (`codigo_pais`, `codigo_area`, `codigo_sede`) mediante extracción de subcadenas. Se construirá una tabla de mapeo que traduzca cada combinación a nombres descriptivos.

**Decodificación de cuentas.** Se construirá una tabla de mapeo entre el código numérico de cuenta SAP y su descripción en lenguaje de negocio y su categoría (Utilities, Facilidades, Flota, etc.).

**Conversión de importes.** Se eliminará el separador de miles y se convertirá la coma decimal a punto. El campo pasará de TEXT a FLOAT.

**Conversión de fechas.** Se aplicará `pd.to_datetime` con el formato explícito correspondiente al export SAP. Se generarán los campos derivados de año, mes, trimestre y periodo de cierre.

**Tratamiento de KP.** Los registros con clase de documento KP se mantendrán en el dataset pero se marcarán con un flag `es_irregular = 1`. No se eliminan porque son informativos para el modelo de anomalías y para el análisis del desbalance con el equipo contable.

**Gastos de otras áreas.** Se identificarán mediante los códigos de CeCo que no pertenecen al área WPE. Se incluirán en el dataset con un flag `es_externo = 1` para permitir análisis de impacto sin contaminar los agregados del área.

**Duplicados.** Se identificarán registros con el mismo número de documento y se conservará únicamente el registro más reciente por fecha de contabilización. Los duplicados eliminados quedarán logueados en el informe de calidad.

**Criterio de registro válido.** Un registro es válido si tiene fecha de contabilización, importe distinto de cero, código de cuenta y código de CeCo completos. Los registros que no cumplan este criterio se excluirán del dataset Silver y se documentarán en el informe de calidad.

---

## 9. Riesgos del modelo de datos

**Lo que está más claro.** La estructura de la tabla de hechos y las dimensiones de cuenta y fecha. El proceso de descarga GD20 es conocido, el esquema estrella está bien definido y la lógica de transformación en Silver es técnicamente directa. El mayor activo aquí es el conocimiento operativo del área adquirido durante la práctica, que permite anticipar problemas que no serían visibles para alguien externo.

**Lo que genera más incertidumbre.** La estructura exacta de los campos en la exportación GD20: nombres de columna, formatos de fecha, codificación de caracteres, presencia o ausencia del campo de proveedor, y volumen real de registros por año. Todo el modelo de datos aquí descrito es una hipótesis bien fundamentada, no una certeza. La Fase 1 del proyecto tiene como objetivo primario resolver esta incertidumbre.

**La fuente que puede dar más problemas.** Los registros con clase de documento KP son el mayor riesgo de calidad. No tienen una interpretación unívoca, su volumen es variable entre cuentas y su tratamiento incorrecto puede distorsionar tanto el EDA como el modelo de anomalías. Requieren una sesión de validación con el área antes de tomar cualquier decisión de tratamiento.

**Qué ocurriría si no se puede construir la capa Gold como está definida.** El escenario de contingencia es construir una versión simplificada sin esquema estrella: una única tabla plana en Silver con todos los campos decodificados, sobre la que se aplican directamente las vistas semánticas para el dashboard. Esto sacrifica la eficiencia de las consultas y la escalabilidad al clúster, pero mantiene el MVP funcional dentro del plazo del proyecto.

**Alternativa de simplificación.** Si el volumen de datos o la complejidad de los códigos SAP supera lo anticipado, el modelo se puede simplificar eliminando la dimensión de proveedor y trabajando con `dim_ceco` a nivel de país solamente, en lugar de descomponerlo en sus tres componentes. Esto reduciría la granularidad del análisis pero mantendría la funcionalidad esencial del dashboard y el modelo de anomalías.
