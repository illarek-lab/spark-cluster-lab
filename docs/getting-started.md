# Como Comenzar

Guia rapida para levantar el lab, validar que los servicios estan vivos y
ejecutar las primeras pruebas con Spark, Iceberg, Polaris, Hive, MinIO y
ClickHouse.

## 1. Entrar al proyecto

En el servidor:

```bash
cd /home/jkn/Projects/spark-cluster-lab
docker compose version
```

En desarrollo local, entra a la carpeta donde clonaste el repositorio.

## 2. Levantar el stack

Construye las imagenes locales de Spark/Hive y levanta todos los servicios:

```bash
docker compose up --build -d
```

Si algun puerto esta ocupado, edita el mapeo correspondiente en `compose.yaml`
antes de levantar el stack. Puedes revisar los defaults con:

```bash
for port in 17077 18080 18181 18182 19000 19001 19083 11000 11002 18123 19010 15433 19870 18088; do
  ss -ltn "( sport = :$port )" | grep -q LISTEN && echo "$port ocupado" || echo "$port libre"
done
```

Verifica el estado:

```bash
docker compose ps
```

Si algun servicio no queda en estado saludable o no arranca, revisa logs:

```bash
docker compose logs --tail=100 spark-master spark-worker minio polaris hive-metastore hiveserver2 clickhouse hadoop-namenode yarn-resourcemanager
```

## 3. Revisar las UIs y endpoints

| Servicio | Desde el host | Uso |
| --- | --- | --- |
| Spark Master UI | <http://localhost:18080> | Ver workers y aplicaciones. |
| Spark Master | `spark://localhost:17077` | Endpoint para jobs Spark. |
| YARN ResourceManager UI | <http://localhost:18088> | Ver aplicaciones y NodeManagers de YARN. |
| HDFS NameNode UI | <http://localhost:19870> | Ver namespace HDFS y DataNodes. |
| MinIO Console | <http://localhost:19001> | Ver buckets y objetos. |
| Polaris REST Catalog | `http://localhost:18181/api/catalog` | Catalogo REST de Iceberg. |
| Hive Metastore | `thrift://localhost:19083` | Catalogo Hive/Iceberg clasico. |
| HiveServer2 | `jdbc:hive2://localhost:11000` | SQL con Beeline/JDBC. |
| ClickHouse HTTP | <http://localhost:18123> | Consultas HTTP. |
| ClickHouse Native | `localhost:19010` | Cliente nativo. |

Credenciales locales:

| Servicio | Usuario | Password |
| --- | --- | --- |
| MinIO | `admin` | `pass1234` |
| Polaris root | `admin` | `pass1234` |
| Hive PostgreSQL | `admin` | `pass1234` |
| ClickHouse | `admin` | `pass1234` (el usuario `default` queda eliminado) |

## 4. Entender donde se guardan los datos

El lab usa un solo stack de Compose y una sola red interna. No levanta otro
cluster Spark por cada herramienta.

| Capa | Ruta | Persistencia |
| --- | --- | --- |
| Iceberg con Hive Metastore | `s3a://warehouse/iceberg` | Volumen Docker `minio_data`. |
| Iceberg con Polaris | `s3a://warehouse/polaris` | Volumen Docker `minio_data`. |
| Warehouse Hive | `s3a://warehouse/hive` | Volumen Docker `minio_data`. |
| Metadatos Hive | PostgreSQL `hive-postgres` | Volumen Docker `hive_postgres_data`. |
| Datos ClickHouse | `/var/lib/clickhouse` | Volumen Docker `clickhouse_data`. |
| Proyecto cliente | DDL, scripts, notebooks, datos sinteticos y jobs | Carpeta externa al repo de infraestructura. |
| Metadata HDFS (NameNode) | HDFS interno, `hdfs://hadoop-namenode:8020` | Volumen Docker `hdfs_namenode_data`. |

MinIO emula S3 y es el almacenamiento real del lakehouse en ambos modos
(Standalone y YARN). Este Compose tambien levanta un HDFS propio
(`hadoop-namenode` + `hadoop-datanode`), pero es efimero y solo sirve para el
shuffle/staging interno de YARN (los DataNode no persisten datos entre
reinicios). Spark y Hive siguen usando Hadoop S3A para leer y escribir tablas
Iceberg/Hive en MinIO.

El tamano de los nodos Spark worker se define en `compose.yaml`:

```yaml
x-cluster-config:
  worker-count: &worker-count 4
  worker-memory: &worker-memory "8g"
  worker-cores: &worker-cores "3"
```

## 5. Probar Spark

Entra al master:

```bash
docker compose exec spark-master bash
```

Desde dentro del contenedor:

```bash
/opt/spark/bin/spark-submit --version
```

Tambien puedes abrir Spark SQL directamente:

```bash
docker compose exec spark-master /opt/spark/bin/spark-sql \
  --master spark://spark-master:7077
```

Para probar el mismo job contra YARN en vez de Standalone, usa
`--master yarn` (o la variable `SPARK_MASTER_DEFAULT`, que ya viene fijada a
`yarn`):

```bash
docker compose exec spark-master spark-submit --master $SPARK_MASTER_DEFAULT --version
```

Verifica en la UI de YARN (<http://localhost:18088>) que la aplicacion
aparezca y que los `yarn-nodemanager` esten registrados. Enviar jobs en modo
YARN solo funciona desde dentro de la red de Compose (`docker compose exec
spark-master ...`), no desde un proyecto cliente externo.

## 6. Crear una tabla Iceberg con Hive Metastore

Dentro de `spark-sql`:

```sql
CREATE DATABASE IF NOT EXISTS iceberg.lab;

CREATE TABLE IF NOT EXISTS iceberg.lab.events (
  id BIGINT,
  event_name STRING,
  created_at TIMESTAMP
) USING iceberg;

INSERT INTO iceberg.lab.events VALUES
  (1, 'started', current_timestamp()),
  (2, 'finished', current_timestamp());

SELECT * FROM iceberg.lab.events;
```

Esa tabla usa el catalogo `iceberg`, respaldado por Hive Metastore, y escribe
archivos en MinIO bajo `s3a://warehouse/iceberg`.

## 7. Crear una tabla Iceberg con Polaris

Dentro de `spark-sql`:

```sql
CREATE NAMESPACE IF NOT EXISTS polaris.lab;

CREATE TABLE IF NOT EXISTS polaris.lab.events (
  id BIGINT,
  event_name STRING,
  created_at TIMESTAMP
) USING iceberg;

INSERT INTO polaris.lab.events VALUES
  (1, 'created-with-polaris', current_timestamp());

SELECT * FROM polaris.lab.events;
```

Esa tabla usa el catalogo `polaris`, respaldado por Apache Polaris REST Catalog,
y escribe archivos en MinIO bajo `s3a://warehouse/polaris`.

## 8. Ver objetos en MinIO

Abre:

```text
http://localhost:19001
```

Entra con:

```text
usuario: admin
password: pass1234
```

Busca el bucket `warehouse`. Deberias ver prefijos como `iceberg/`, `polaris/`
y `hive/` cuando ya hayas creado tablas.

## 9. Probar HiveServer2

Abre Beeline:

```bash
docker compose exec hiveserver2 beeline -u jdbc:hive2://localhost:10000
```

Ejecuta:

```sql
SHOW DATABASES;
```

## 10. Probar ClickHouse

Prueba el endpoint HTTP:

```bash
curl -u admin:pass1234 'http://localhost:18123/?query=SELECT%201'
```

O entra al cliente nativo:

```bash
docker compose exec clickhouse clickhouse-client --user admin --password pass1234
```

Dentro del cliente:

```sql
SELECT version();
```

## 11. Apagar el lab

Detener y eliminar contenedores y red, conservando datos:

```bash
docker compose down
```

Detener y eliminar tambien volumenes de MinIO, Postgres y ClickHouse:

```bash
docker compose down -v
```

Eliminar imagenes locales construidas por el proyecto:

```bash
docker compose down --rmi local
```

Este repo no guarda carpetas locales de datos, notebooks, JARs de usuario ni
warehouses. Los datos del lab viven en volumenes Docker y los scripts de trabajo
deben vivir en el proyecto cliente.

## 12. Siguiente paso

Para conectar otro proyecto, usa:

| Recurso | Valor desde el host |
| --- | --- |
| Spark master | `spark://localhost:17077` |
| MinIO endpoint | `http://localhost:19000` |
| Hive Metastore | `thrift://localhost:19083` |
| Polaris catalog | `http://localhost:18181/api/catalog` |
| Iceberg Hive warehouse | `s3a://warehouse/iceberg` |
| Iceberg Polaris warehouse | `s3a://warehouse/polaris` |

Si te conectas desde otra maquina, reemplaza `localhost` por la IP o DNS del
servidor donde corre Docker Compose.
