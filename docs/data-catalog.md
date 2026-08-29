# Catálogo de datos (OpenMetadata)

Capa opcional de descubrimiento/gobierno de datos (búsqueda, glosario,
linaje), separada de la infra principal. No es el catálogo técnico que usan
Spark/Iceberg para resolver tablas (ese es Hive Metastore o Polaris, ver
[README](../README.md#configuración-del-clúster)) — es un panel para
explorar y documentar lo que ya existe ahí.

Vendorizado desde el quickstart oficial de OpenMetadata 1.13.3 en
[`compose.openmetadata.yaml`](../compose.openmetadata.yaml), como stack
separado que se conecta a la red del lab.

## Levantar el stack

Necesita que el lab principal ya esté arriba (usa su red para llegar a
`hiveserver2`):

```bash
docker compose up -d
docker compose -f compose.openmetadata.yaml up -d
docker compose -f compose.openmetadata.yaml ps
```

La primera vez tarda unos minutos: Postgres + Elasticsearch + la migración
de esquema (`om-execute-migrate-all`) antes de que el servidor quede healthy.

## Acceder

| Recurso | URL | Uso |
| --- | --- | --- |
| OpenMetadata UI | `http://<host>:8585` | Buscar, documentar, ver linaje. |
| Login | `admin@open-metadata.org` / `admin` | Credencial default de la imagen, cámbiala si vas a dejarlo expuesto. |
| Airflow (ingestion) | `http://<host>:18085` | Ver el estado de los pipelines de ingestión (usuario `admin`/`admin`). |
| Elasticsearch | `http://<host>:19200` | Backend de búsqueda, normalmente no lo tocás directo. |

## Conectar tus tablas Iceberg/Hive

**Importante:** en la versión 1.13.3 de OpenMetadata no existe todavía un
conector nativo de Iceberg (ni para Hive Metastore ni para catálogos REST
como Polaris). Lo que sí hay es el conector **Hive**, que habla con
HiveServer2 y sirve para catalogar las tablas del catálogo `iceberg.*`
(las que están registradas en Hive Metastore) — no para `polaris.*`.

1. En la UI, andá a **Settings → Services → Databases → Add New Service**.
2. Elegí **Hive** como tipo de servicio.
3. Configuración de conexión:
   - **Host and Port**: `hiveserver2:10000`
   - **Scheme**: `hive`
   - **Auth**: `NONE` (este lab no tiene autenticación en HiveServer2)
   - **Database Name** (opcional): dejalo en blanco para que escanee todo.
4. Guardá el servicio y andá a la pestaña **Ingestion** del servicio →
   **Add Ingestion** → tipo **Metadata**. Dejá el schedule en manual o el
   que prefieras, y corré el pipeline.
5. Cuando termine, en **Explore** deberían aparecer tus bases/tablas, por
   ejemplo `iceberg.lab.events` (o `iceberg.lab.cdr_*` según lo que hayas
   creado), con sus columnas.

Nota: OpenMetadata lee el **esquema** vía HiveServer2 (que sí entiende la
tabla porque está en el metastore), pero no necesariamente puede *ejecutar*
`SELECT` sobre datos Iceberg reales con el motor Hive 3.1.0 de este lab —
para eso seguís usando Spark. Acá solo estás catalogando/documentando, no
consultando datos.

### Polaris (`polaris.*`)

No hay conector compatible en esta versión. Alternativas si lo necesitás:

- Actualizar a una versión de OpenMetadata más reciente que sí traiga
  conector Iceberg con soporte REST catalog (revisar el changelog de
  [openmetadata.org](https://open-metadata.org/product-updates) antes de
  subir la versión en `compose.openmetadata.yaml`).
- Documentar esas tablas a mano en OpenMetadata (crear el asset manualmente
  vía UI o API), sin ingestión automática.

## Apagar / limpiar

```bash
docker compose -f compose.openmetadata.yaml down
```

Agregá `-v` si además querés borrar el catálogo de OpenMetadata (Postgres +
índices de Elasticsearch) y empezar de cero.
