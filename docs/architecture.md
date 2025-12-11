# Arquitectura del Sistema

## Visión General

El sistema H1 es un pipeline de procesamiento de datos batch que opera sobre eventos de Google Analytics 4 almacenados en BigQuery. La arquitectura sigue un patrón ETL (Extract, Transform, Load) con énfasis en la detección de patrones complejos de negocio.

## Componentes Principales

### 1. Capa de Datos (BigQuery)

#### Tablas Fuente
- **GA4 Events** (`sorteostec-analytics360.analytics_277858205.events_intraday_*`)
  - Eventos raw de interacción web
  - Particionada por fecha (_TABLE_SUFFIX)
  - Filtrada por platform='WEB'

#### Tablas de Catálogo
- `ml_siteconversion.sorteo`: Catálogo de productos/sorteos
- `ml_siteconversion.promocion`: Definición de promociones
- `ml_siteconversion.condicion_promocion`: Reglas de promociones
- `ml_siteconversion.grupo_condiciones_promocion`: Agrupación de condiciones

#### Tablas Procesadas
- `h1.intentos_producto_canonico_web_*`: Base canónica de intentos
- `h1.sesiones_funnel_lineal_web_*`: Análisis lineal del funnel
- `h1.ga4_patrones_promociones_*`: Patrones detectados
- `h1.patrones_y_funnel_web_*`: Tabla final consolidada

### 2. Capa de Procesamiento (Python)

#### H1Script.py - Pipeline Principal
```
[BigQuery] → [DataFrame] → [Enriquecimiento] → [Detección] → [Agregación] → [Output]
```

**Fases del procesamiento:**

1. **Extracción** (5 min)
   - Ejecuta queries SQL parametrizados
   - Carga DataFrames en memoria

2. **Enriquecimiento** (2 min)
   - Merge con catálogos
   - Interpretación de condiciones
   - Normalización de fechas

3. **Detección de Patrones** (2 hrs)
   - Iteración por sesión
   - Evaluación de promociones simples
   - Evaluación de promociones combinadas
   - Clasificación completa/incompleta

4. **Agregación** (5 min)
   - Nivel sesión
   - Cálculo de KPIs
   - Estadísticas de conversión

5. **Output** (1-2min)
   - Generación de CSVs
   - Carga a BigQuery

#### H1ShortScript.py - Post-proceso
- Versión optimizada sin recreación de tablas base
- Solo ejecuta análisis sobre datos existentes
- Útil para iteraciones rápidas

### 3. Capa de Utilidades

#### BQLoadClass.py
Wrapper sobre google-cloud-bigquery que provee:
- Gestión de credenciales
- Estandarización de fechas
- Carga batch de DataFrames
- Manejo de esquemas

## Flujo de Datos Detallado

### Entrada
```sql
GA4 Events → Filtrado (WEB, fechas) → Deduplicación → Agregación por intento
```

### Transformación Principal
```python
for sesion in todas_sesiones:
    promociones_completas = evaluar_promociones_sesion(sesion)
    for producto in sesion:
        patrones = detectar_patrones_producto(
            producto,
            condiciones,
            promociones_completas
        )
```

### Salida
```
DataFrame → CSV (local) + BigQuery (persistente)
```

## Particionamiento y Clustering

### Estrategia de Particionamiento
- **Campo**: `attempt_date` o `session_date`
- **Tipo**: Daily
- **Beneficio**: Reduce costos de consulta hasta 90%

### Estrategia de Clustering
- **Campos**: `ITEM`, `STATUS`, `login_bucket_bc`
- **Beneficio**: Mejora performance en queries frecuentes

## Consideraciones de Performance

### Memoria
- Dataset completo: ~2-3GB en memoria
- Picos durante merge: hasta 4GB
- Recomendado: 8GB RAM mínimo

### Tiempo de Ejecución
- Pipeline completo: 15-30 minutos
- Post-proceso: 5-10 minutos
- Queries BigQuery: 10-30 segundos cada una

### Optimizaciones Implementadas
1. **Deduplicación temprana** en SQL
2. **Procesamiento vectorizado** con pandas
3. **Caché de promociones multi-producto**
4. **Context managers** para gestión de memoria
5. **Índices en DataFrames** para joins rápidos

## Escalabilidad

### Límites Actuales
- Máximo período recomendado: 12 meses
- Máximo sesiones por ejecución: ~1M
- Máximo productos únicos: ~1000

### Estrategias de Escalado
1. **Procesamiento incremental** por mes
2. **Paralelización** por user_pseudo_id
3. **Materialización** de vistas intermedias
4. **Scheduled queries** en BigQuery

## Monitoreo y Observabilidad

### Logging
- **Nivel**: INFO por defecto
- **Rotación**: 10MB, 5 archivos
- **Formato**: timestamp | level | module | message

### Métricas Clave
- Tiempo por etapa
- Registros procesados
- Memoria utilizada
- Tasa de detección de patrones

### Puntos de Control
```python
with _time_block("Nombre de etapa"):
    # procesamiento
    logger.info(_df_stats(df, "nombre_df"))
```

## Manejo de Errores

### Estrategia General
- Try-catch en operaciones críticas
- Valores por defecto para datos faltantes
- Logging detallado de excepciones
- Rollback parcial cuando es posible

### Casos Especiales
- **Fechas inválidas**: Se parsean múltiples formatos
- **Promociones sin vigencia**: Se ignoran
- **Productos no catalogados**: Se mantienen con precio NULL

## Seguridad

### Credenciales
- Service Account con permisos mínimos
- JSON key no versionado
- Path hardcodeado (considerar variables de entorno)

### Acceso a Datos
- Read-only en tablas fuente
- Write en dataset específico (h1)
- No PII en logs o CSVs