# Data-Mining-Deber3
---


## Arquitectura

**Flujo general de datos:**  Spark/Jupyter → Snowflake (`raw` → `analytics.obt_trips`).

<img width="1249" height="268" alt="Diagrama" src="https://github.com/user-attachments/assets/1c96307f-a789-415b-9377-e26dd4c2fb13" />

---

## Matriz de cobertura 2015–2025 por servicio/mes

- Todos los años y meses fueron **procesados correctamente** para ambos servicios (**Yellow y Green**).    

![Matriz](https://github.com/user-attachments/assets/637024dc-8dc6-4b34-8e72-bcdb88249b9b)

>  Estos resultados se documentaron mediante la matriz de cobertura generada en el Notebook 1.

---

## Pasos para Docker Compose y ejecución de notebooks

1. **Configuración inicial:**
   - Duplicar el archivo `.env.example` y renombrarlo a `.env`.  
   - Completar las variables necesarias con las credenciales de Snowflake.  

2. **Ejecución del contenedor:**
   ```bash
   docker compose up -d
   ```
   Esto levanta el contenedor con **Jupyter Notebook y Spark**.  

3. **Acceso al entorno:**
   - Entrar en el navegador a `http://localhost:8888`  
   - Ir a la carpeta `work/` dentro de Jupyter.  

4. **Ejecución de los notebooks (en orden):**

   | Orden | Notebook | Descripción |
   |:------:|:----------|:-------------|
   | 1 | `01_ingesta_parquet_raw.ipynb` | Ingesta de Parquet (2015–2025) hacia Snowflake (RAW) con prints de auditoría. |
   | 2 | `02_enriquecimiento_y_unificacion.ipynb` | Limpieza, enriquecimiento y unión de Yellow/Green. |
   | 3 | `03_construccion_obt.ipynb` | Construcción de la tabla `analytics.obt_trips` (OBT). |
   | 4 | `04_validaciones_y_exploracion.ipynb` | Validación de datos (rangos, nulos, coherencia). |
   | 5 | `05_data_analysis.ipynb` | Respuestas a las 20 preguntas de análisis. |

> La secuencia correcta de ejecución va del **notebook 1 al notebook 5**.

---

## Variables de Ambiente

El proyecto usa variables de entorno para parametrizar credenciales, rutas y parámetros.  
- Estas se definen en el archivo `.env`, tomando como base el archivo `.env.example` (sin credenciales reales).

### Listado de variables y propósito

#### **Credenciales de Snowflake**

| Variable | Descripción |
|----------|-------------|
| `SNOWFLAKE_ACCOUNT` | Identificador de la cuenta de Snowflake (formato: xxxxx-xxxxx) |
| `SNOWFLAKE_USER` | Usuario de acceso a Snowflake |
| `SNOWFLAKE_PASSWORD` | Contraseña de Snowflake |
| `SNOWFLAKE_ROLE` | Rol con permisos CREATE/INSERT/SELECT en los esquemas |

#### **Configuración de Snowflake**

| Variable | Descripción |
|----------|-------------|
| `SNOWFLAKE_DATABASE` | Base de datos donde se almacenan las tablas |
| `SNOWFLAKE_SCHEMA_RAW` | Esquema RAW de aterrizaje de datos sin transformar |
| `SNOWFLAKE_SCHEMA_ANALYTICS` | Esquema ANALYTICS con la tabla final OBT |
| `SNOWFLAKE_WAREHOUSE` | Warehouse de cómputo para ejecutar queries |

#### **Conector Spark-Snowflake**

| Variable | Descripción |
|----------|-------------|
| `SF_CONN_URL` | URL de conexión a Snowflake |
| `SF_JDBC_GROUP` | Grupo Maven del driver JDBC |
| `SF_JDBC_ARTIFACT` | Artefacto del driver JDBC |
| `SF_JDBC_VERSION` | Versión del driver JDBC de Snowflake |
| `SF_SPARK_GROUP` | Grupo Maven del conector Spark |
| `SF_SPARK_ARTIFACT` | Artefacto del conector Spark-Snowflake |
| `SF_SPARK_VERSION` | Versión del conector Spark-Snowflake |

#### **Origen de Datos NYC TLC**

| Variable | Descripción |
|----------|-------------|
| `SOURCE_BASE` | URL base de los archivos Parquet del NYC TLC |
| `TAXI_ZONE_URL` | URL del archivo CSV con el catálogo de zonas de taxi |
| `YEARS` | Rango de años a procesar (2015-2025) |
| `SERVICES` | Servicios de taxi a procesar (yellow, green) |

#### **Parámetros de Procesamiento**

| Variable | Descripción |
|----------|-------------|
| `RUN_ID` | Identificador único para auditoría de cada ejecución |
| `BATCH_SIZE` | Tamaño del lote de procesamiento (número de registros) |
| `START_YEAR` | Año inicial del rango de datos a procesar |
| `END_YEAR` | Año final del rango de datos a procesar |
---

## Diseño de RAW y OBT (columnas, derivadas, metadatos, supuestos)

### RAW
- Esquema espejo del origen Parquet.  
- Campos adicionales de auditoría:  
  `run_id`, `service_type`, `source_year`, `source_month`, `ingested_at_utc`, `source_path`, `row_count`.  
- Se creó la tabla **`INGESTA_AUDIT`** en RAW para almacenar auditoría de ingestas (servicio, año, mes, batch, número de filas y URL).  

### OBT (analytics.obt_trips)
- Grano: **1 fila = 1 viaje**.  
- Contiene todas las columnas necesarias para análisis (hechos, dimensiones y derivadas).  
- Columnas derivadas:
  - `trip_duration_min` (minutos entre pickup y dropoff).  
  - `avg_speed_mph` (velocidad promedio en mph).  
  - `tip_pct` (porcentaje de propina).  
- Incluye metadatos de lineage: `run_id`, `source_service`, `source_year`, `source_month`.  

### Supuestos
- Se procesaron todos los meses de 2015–2025 para ambos servicios.  
- El procesamiento fue mensual (por chunk) para optimizar rendimiento.  
- La tabla OBT se diseñó siguiendo la definición del PDF (modelo desnormalizado para análisis rápido).  

---

## Calidad / Auditoría (qué se valida y dónde se ve)

La validación de calidad y auditoría se puede observar en los notebooks mediante prints y registros automáticos, al igual que en las tablas creadas en Snowflake.  

### Dentro de los notebooks
- En todos los notebooks se muestran **prints de auditoría** como conteo de filas, nulos, tiempo de ejecución, etc.

### En Snowflake
- **Tabla `RAW.INGESTA_AUDIT`:** registra por servicio/año/mes el número de filas, URL de origen, batch y estado de carga.  
- **Tabla `CLEAN.UNIFIED_TRIPS`:** tabla intermedia usada para validar pasos antes de construir la OBT.  

### Validaciones aplicadas
- Consistencia entre `pickup_datetime` y `dropoff_datetime`.  
- Rangos válidos de distancia, duración y montos (no negativos).  
- No nulos en claves esenciales.  
- Idempotencia: reingestar un mes no genera duplicados.
- Entre otros.

---

## Checklist de Aceptación

- [x] Docker Compose levanta Spark y Jupyter Notebook
- [x] Todas las credenciales/parámetros provienen de variables de ambiente (.env)
- [x] Cobertura 2015–2025 (Yellow/Green) cargada en raw con matriz y conteos por lote
- [x] analytics.obt_trips creada con columnas mínimas, derivadas y metadatos
- [x] Idempotencia verificada reingestando al menos un mes
- [x] Validaciones básicas documentadas (nulos, rangos, coherencia)
- [x] 20 preguntas respondidas (texto) usando la OBT
- [x] README claro: pasos, variables, esquema, decisiones, troubleshooting

---

**Nombre:** Anahí Andrade  
**Curso:** Data Mining – Universidad San Francisco de Quito
