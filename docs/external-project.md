# Usar el Lab desde otro Proyecto

Esta guia describe como usar la infraestructura de `spark-cluster-lab` desde un
proyecto externo, sin crear otro cluster Spark ni levantar otro stack.

Proyecto cliente de ejemplo:

```text
/Users/jkn/Documents/Projects/spark-cluster-lab-test
```

Servidor donde corre la infraestructura:

```text
100.85.61.29
```

## 1. Confirmar que el lab esta levantado

En el servidor donde corre `spark-cluster-lab`:

```bash
cd /home/jkn/Projects/spark-cluster-lab
docker compose ps
```

Servicios clave que deben estar arriba:

| Servicio | Endpoint desde el proyecto cliente |
| --- | --- |
| Spark Master | `spark://100.85.61.29:7077` |
| Spark Master UI | `http://100.85.61.29:8080` |
| MinIO S3 API | `http://100.85.61.29:9000` |
| MinIO Console | `http://100.85.61.29:9001` |
| Polaris REST Catalog | `http://100.85.61.29:8181/api/catalog` |
| Hive Metastore | `thrift://100.85.61.29:9083` |
| HiveServer2 | `jdbc:hive2://100.85.61.29:10000` |
| ClickHouse HTTP | `http://100.85.61.29:8123` |
| ClickHouse Native | `100.85.61.29:9002` |

## 2. Entrar al proyecto cliente

En tu maquina local:

```bash
cd /Users/jkn/Documents/Projects/spark-cluster-lab-test
```

Ese proyecto debe contener tus DDL, scripts de datos sinteticos y jobs de
prueba. No necesita definir otro `docker compose` para Spark.

## 3. Variables recomendadas

Usa estas variables en scripts o `.env` del proyecto cliente:

```bash
export SPARK_MASTER_URL=spark://100.85.61.29:7077
export MINIO_ENDPOINT=http://100.85.61.29:9000
export MINIO_ACCESS_KEY=minioadmin
export MINIO_SECRET_KEY=minioadmin
export HIVE_METASTORE_URI=thrift://100.85.61.29:9083
export POLARIS_CATALOG_URI=http://100.85.61.29:8181/api/catalog
export POLARIS_WAREHOUSE=lakehouse
export POLARIS_CREDENTIAL=root:s3cr3t
export CLICKHOUSE_HOST=100.85.61.29
export CLICKHOUSE_HTTP_PORT=8123
export CLICKHOUSE_NATIVE_PORT=9002
```

## 4. Dependencias del proyecto cliente

El cliente Python debe usar una version compatible con Spark `3.5.x`:

```bash
pip install 'pyspark>=3.5.6,<3.6'
```

Si el proyecto crea datos sinteticos con Python, instala tambien las librerias
que use tu script, por ejemplo `faker`, `pandas` o `pyarrow`.

## 5. Crear una SparkSession contra el cluster remoto

Ejemplo base para scripts Python:

```python
import os

from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("spark-cluster-lab-test")
    .master(os.getenv("SPARK_MASTER_URL", "spark://100.85.61.29:7077"))
    .config(
        "spark.sql.extensions",
        "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions",
    )
    .config("spark.sql.catalog.iceberg", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.iceberg.type", "hive")
    .config("spark.sql.catalog.iceberg.uri", "thrift://100.85.61.29:9083")
    .config("spark.sql.catalog.iceberg.warehouse", "s3a://warehouse/iceberg")
    .config("spark.sql.catalog.polaris", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.polaris.catalog-impl", "org.apache.iceberg.rest.RESTCatalog")
    .config("spark.sql.catalog.polaris.uri", "http://100.85.61.29:8181/api/catalog")
    .config("spark.sql.catalog.polaris.warehouse", "lakehouse")
    .config("spark.sql.catalog.polaris.credential", "root:s3cr3t")
    .config("spark.sql.catalog.polaris.scope", "PRINCIPAL_ROLE:ALL")
    .config("spark.hadoop.fs.s3a.endpoint", "http://100.85.61.29:9000")
    .config("spark.hadoop.fs.s3a.access.key", "minioadmin")
    .config("spark.hadoop.fs.s3a.secret.key", "minioadmin")
    .config("spark.hadoop.fs.s3a.path.style.access", "true")
    .config("spark.hadoop.fs.s3a.connection.ssl.enabled", "false")
    .getOrCreate()
)
```

## 6. Ejecutar DDL contra Iceberg

Si tienes archivos SQL en el proyecto cliente, puedes ejecutarlos desde Python
leyendo el archivo y enviando cada sentencia a Spark:

```python
from pathlib import Path

sql_text = Path("ddl/create_tables.sql").read_text()

for statement in sql_text.split(";"):
    statement = statement.strip()
    if statement:
        spark.sql(statement)
```

Ejemplo de DDL para Hive Metastore:

```sql
CREATE DATABASE IF NOT EXISTS iceberg.lab;

CREATE TABLE IF NOT EXISTS iceberg.lab.synthetic_events (
  id BIGINT,
  event_name STRING,
  amount DOUBLE,
  created_at TIMESTAMP
) USING iceberg;
```

Ejemplo de DDL para Polaris:

```sql
CREATE NAMESPACE IF NOT EXISTS polaris.lab;

CREATE TABLE IF NOT EXISTS polaris.lab.synthetic_events (
  id BIGINT,
  event_name STRING,
  amount DOUBLE,
  created_at TIMESTAMP
) USING iceberg;
```

## 7. Cargar datos sinteticos

Ejemplo minimo:

```python
from datetime import datetime

rows = [
    (1, "created", 10.5, datetime.utcnow()),
    (2, "paid", 25.0, datetime.utcnow()),
]

df = spark.createDataFrame(rows, "id long, event_name string, amount double, created_at timestamp")

df.writeTo("iceberg.lab.synthetic_events").append()
df.writeTo("polaris.lab.synthetic_events").append()
```

Luego valida:

```python
spark.sql("SELECT count(*) FROM iceberg.lab.synthetic_events").show()
spark.sql("SELECT count(*) FROM polaris.lab.synthetic_events").show()
```

## 8. Ver resultados

MinIO:

```text
http://100.85.61.29:9001
```

Credenciales:

```text
usuario: minioadmin
password: minioadmin
```

Busca el bucket `warehouse` y los prefijos `iceberg/` y `polaris/`.

Spark UI:

```text
http://100.85.61.29:8080
```

## 9. Probar ClickHouse desde el proyecto cliente

Consulta rapida por HTTP:

```bash
curl 'http://100.85.61.29:8123/?query=SELECT%201'
```

Si tu proyecto necesita cargar datos en ClickHouse, usa el endpoint HTTP
`http://100.85.61.29:8123` o el puerto nativo `100.85.61.29:9002`.

## 10. Regla importante

El proyecto cliente no debe crear otro cluster Spark. Debe conectarse al cluster
existente:

```text
spark://100.85.61.29:7077
```

El tamano real del cluster se controla solo en `spark-cluster-lab/compose.yaml`:

```yaml
x-cluster-config:
  worker-count: &worker-count 4
  worker-memory: &worker-memory "8g"
  worker-cores: &worker-cores "3"
```
