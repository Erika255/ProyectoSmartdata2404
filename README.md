# 🏥 Trabajo Final — Ingeniería de Datos con Databricks

Pipeline ETL end-to-end con arquitectura Medallion (Bronze → Silver → Gold) sobre Azure Databricks, aplicado al análisis de deserción de pacientes en una clínica de salud.

---

## 📊 Datasets

Datos ficticios generados con estructura real del sector salud peruano:

- **citas.csv** — 3,000 registros de citas médicas programadas con estado (Atendido, Inasistencia, Cancelado)
- **pacientes.csv** — 2,500 pacientes con datos demográficos y tipo de seguro
- **medicos.csv** — 60 médicos con especialidad y turno
- **especialidades.csv** — 15 especialidades médicas con costo y duración
- **atenciones.csv** — 1,806 atenciones efectivas con diagnóstico y tratamiento
- **pagos.csv** — 1,806 registros de pagos con monto, tipo y estado

---

## 🏗️ Arquitectura Medallion

```
[Azure Data Lake Gen2 - Raw]
        ↓  (Managed Identity)
🥉 Bronze  →  Ingesta CSV → Delta sin transformación
        ↓
🥈 Silver  →  Limpieza, joins, UDF de riesgo de deserción
        ↓
🥇 Golden  →  Métricas de negocio listas para dashboard
        ↓
📊 Dashboard  →  Databricks Lakeview / Power BI
```

**Principios de diseño:**
- Sin secretos en código: conexión a ADLS 100% via Managed Identity
- Idempotente: todos los notebooks usan `mode("overwrite")` y `CREATE IF NOT EXISTS`
- Single source of truth: Unity Catalog governa catalog/schema/tabla + permisos
- Capas independientes: falla aislada por capa

---

## ☁️ Servicios Azure aprovisionados

| Servicio | Nombre | Propósito |
|---|---|---|
| Resource Group | `rg-project` | Contenedor de todos los recursos |
| Storage Account ADLS Gen2 | `adlsproyecto2404` | Contenedores raw, bronze, silver, golden, metastore |
| Access Connector for Azure Databricks | `ac-project2404` | Managed Identity para acceso seguro al storage |
| Key Vault | `akvsmartdata2404` | Almacén seguro de secretos |
| Azure Databricks Workspace (dev) | `adbsmartdata2404` | Desarrollo y pruebas del ETL |
| Azure Databricks Workspace (prod) | `adbsmartdata2404prod` | Producción — despliegue via CI/CD |

**Permiso clave:** el Access Connector tiene rol `Storage Blob Data Contributor` sobre el Storage Account. Esa es la única credencial del proyecto.

---

## 📁 Estructura del repositorio

```
ProyectoSmartdata2404/
├── PrepAmb/
│   └── 0_preparacion_ambiente       # Crea catalog, schemas, external locations y tablas
├── proceso/
│   ├── 1_ingest_citas               # Ingesta citas.csv → Bronze
│   ├── 1_ingest_pacientes           # Ingesta pacientes.csv → Bronze
│   ├── 1_ingest_medicos             # Ingesta medicos.csv → Bronze
│   ├── 1_ingest_especialidades      # Ingesta especialidades.csv → Bronze
│   ├── 1_ingest_atenciones          # Ingesta atenciones.csv → Bronze
│   ├── 1_ingest_pagos               # Ingesta pagos.csv → Bronze
│   ├── 2_transform                  # Limpieza, joins y UDF → Silver
│   ├── 3_load                       # Métricas de deserción → Golden
│   └── 4_grants_medallion           # GRANTS a usuarios y grupos
├── seguridad/
│   └── grants                       # Roles, grupos y GRANTS de Unity Catalog
├── reversion/
│   └── drop_medallion               # DROP de tablas y rutas físicas
├── dashboard/
│   └── *.png                        # Capturas del dashboard
├── datasets/
│   └── *.csv                        # Datasets de clínica (ficticios)
├── certificaciones/
│   └── *.png                        # Certificaciones Databricks
├── evidencias/
│   └── *.png                        # Capturas de ejecuciones exitosas
├── .github/workflows/
│   └── deploy.yml                   # CI/CD GitHub Actions
├── databricks.yml                   # Asset Bundle (define el job WF_ADB)
└── README.md                        # Este archivo
```

---

## 🔄 El ETL en detalle

### Bronze — Ingesta
Lee los 6 CSVs desde el contenedor `raw` via Managed Identity y los persiste como tablas Delta en `catalog_clinica.bronze`. Características:
- Schema explícito definido con `StructType` (no inferido)
- Columna de auditoría `INGESTION_DATE` con `current_timestamp()`
- Modo `overwrite` — idempotente y re-ejecutable

### Silver — Transformación
Aplica limpieza y enriquecimiento:
- Join de `citas` + `pacientes` + `médicos` + `especialidades` → `citas_detalle`
- Join de `atenciones` + `pagos` → `atenciones_detalle`
- UDF para clasificar riesgo de deserción: Alto (Inasistencia), Medio (Cancelado), Bajo (Atendido)
- Columnas derivadas: `anio`, `mes`, `dia_semana`, `hora`, `turno`

### Golden — Métricas de negocio

| Tabla | Pregunta de negocio |
|---|---|
| `desercion_por_especialidad` | ¿Qué especialidades tienen mayor tasa de inasistencia? |
| `desercion_por_seguro` | ¿Qué tipo de seguro tiene más deserción? |
| `ingresos_por_especialidad` | ¿Qué especialidades generan más ingresos? |
| `pacientes_recurrentes` | ¿Qué pacientes tienen historial de inasistencias? |

---

## 🔐 Seguridad

Modelo de roles implementado en Unity Catalog:

| Grupo | Permisos |
|---|---|
| `data_engineers_clinica` | USE CATALOG + USE SCHEMA + CREATE TABLE en bronze, silver, golden |
| Usuario individual | SELECT en todas las tablas + READ/WRITE FILES en external locations |

---

## 🚀 CI/CD con GitHub Actions

El workflow `.github/workflows/deploy.yml` ejecuta automáticamente al hacer push a `main`:

```
push to main
      ↓
  Export notebooks from dev workspace
      ↓
  Deploy notebooks to prod workspace
      ↓
  Create/Update Databricks Workflow WF_ADB
      ↓
  Execute WF_ADB in prod
      ↓
  Monitor execution
```

**Secrets configurados en GitHub:**
- `DATABRICKS_ORIGIN_HOST` — URL del workspace dev
- `DATABRICKS_ORIGIN_TOKEN` — PAT del workspace dev
- `DATABRICKS_DEST_HOST` — URL del workspace prod
- `DATABRICKS_DEST_TOKEN` — PAT del workspace prod

---

## 📸 Evidencias

Ver carpeta `evidencias/` para capturas de:
- Ejecución exitosa del workflow WF_ADB
- GitHub Actions corriendo correctamente
- Servicios aprovisionados en Azure
- Tablas creadas en Unity Catalog

---

## 👩‍💻 Autora

**Erika Vasquez Ruiz** — Trabajo Final del curso SmartData: Ingeniería de Datos e IA con Databricks (Grupo G13)

---

## 📚 Referencias

- [Databricks Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
- [Unity Catalog](https://docs.databricks.com/data-governance/unity-catalog/index.html)
- [Access Connector for Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/administration-guide/access-control/azure-managed-identity)
- [GitHub Actions para Databricks](https://docs.databricks.com/dev-tools/ci-cd/ci-cd-github.html)
