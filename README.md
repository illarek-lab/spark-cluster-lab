# Spark Cluster Lab

Infraestructura para levantar un laboratorio lakehouse con Docker Compose:
Apache Spark en modo Standalone, Apache Iceberg, Apache Polaris, Hive
Metastore, HiveServer2, MinIO como almacenamiento S3 compatible y ClickHouse
para analítica columnar. Los proyectos que usan este stack como cliente pueden
vivir fuera de aquí.

## Indice

- [Como comenzar](docs/getting-started.md): guia paso a paso para levantar el
  stack y probar Spark, Iceberg, Polaris, Hive, MinIO y ClickHouse.
- [Usar desde otro proyecto](docs/external-project.md): guia para conectar un
  proyecto cliente, como `spark-cluster-lab-test`, al servidor `100.85.61.29`.
- [Configuración del clúster](#configuración-del-clúster): tamaño de workers y
  recursos Spark.
- [Herramientas y componentes incluidos](#herramientas-y-componentes-incluidos):
  tabla de servicios, versiones y responsabilidades.
- [Almacenamiento lakehouse](#almacenamiento-lakehouse): rutas S3A, metadatos y
  persistencia.
- [Parar y limpiar](#parar-y-limpiar): comandos para apagar, eliminar volumenes
  e imagenes.

## Requisitos

- macOS, Linux o Windows con Docker Desktop instalado y en ejecución.
- Docker Compose v2 (`docker compose version`).

Comprueba Docker Compose antes de empezar:

```bash
docker compose version
```

## Ejecutar en el servidor

El proyecto debe estar previamente disponible en el servidor en:

```text
/home/jkn/Projects/spark-cluster-lab
```

Desde una terminal abierta directamente en ese servidor:

```bash
cd /home/jkn/Projects/spark-cluster-lab
docker compose version
docker compose up --build -d
docker compose ps
```

La interfaz web estará disponible en `http://100.85.61.29:8080` si el puerto
está permitido por el firewall y la red del servidor. El endpoint Spark será
`spark://100.85.61.29:7077` para clientes externos. Desde los propios
contenedores se usa `spark://spark-master:7077`.

También quedan disponibles MinIO, Polaris, Hive y ClickHouse en los puertos
indicados en [Servicios disponibles](#servicios-disponibles).

## Configuración del clúster

Los valores del clúster no son secretos, así que viven versionados dentro de
`compose.yaml`, en el bloque `x-cluster-config` (usa anchors YAML `&`/`*` en
vez de variables de entorno):

```yaml
x-cluster-config:
  worker-count: &worker-count 4
  worker-memory: &worker-memory "8g"
  worker-cores: &worker-cores "3"
```

Ese bloque es la fuente de verdad para el tamaño de los nodos worker:

| Parámetro | Valor actual | Qué controla | Dónde se aplica |
| --- | ---: | --- | --- |
| `worker-count` | `4` | Cantidad de nodos worker del clúster. | `deploy.replicas` del servicio `spark-worker`. |
| `worker-memory` | `8g` | Memoria máxima por worker. | `mem_limit` de Docker y `--memory` del worker Spark. |
| `worker-cores` | `3` | CPU asignada por worker. | `cpus` de Docker y `--cores` del worker Spark. |

- `worker-count` controla el número de réplicas del servicio `spark-worker`
  vía `deploy.replicas`, así que no hace falta el flag `--scale` al arrancar.
- `worker-memory` se reutiliza (con `*worker-memory`) tanto en `mem_limit`
  (límite de Docker) como en `--memory` del comando `spark-class` del worker.
- `worker-cores` se reutiliza (con `*worker-cores`) tanto en `cpus` (límite
  de Docker) como en `--cores` del comando `spark-class` del worker.

### Cómo elegir `worker-cores` y `worker-memory`

Dimensiona en función de los recursos reales del host, dejando margen para el
master, el motor Docker y el sistema operativo. Ejemplo con un servidor de
15 cores y 50 GB de RAM:

- RAM: `worker-count=4` × `worker-memory=8g` = 32 GB para los workers,
  dejando 18 GB libres para el master, Docker y el SO.
- Cores: `worker-count=4` × `worker-cores=3` = 12 cores para los workers,
  dejando 3 cores libres para el master, Docker y el SO. Sube `worker-cores`
  solo si el host no ejecuta nada más en paralelo; en un servidor compartido
  es más seguro dejar 2 en vez de 3.

Para aplicar cambios, edita `compose.yaml` y recrea los servicios:

```bash
docker compose down
docker compose up --build -d
```

## Estructura de carpetas y archivos

### Archivos de la raíz

- `compose.yaml`: define el master y los workers Spark, sus puertos, límites
  de recursos, volúmenes y red interna. El bloque `x-cluster-config` fija el
  número de réplicas del worker y sus límites de RAM/CPU mediante anchors
  YAML reutilizados en `deploy.replicas`, `mem_limit`, `cpus` y el comando
  `spark-class`.
- `.env`: no existe en este repositorio; la configuración del clúster no es
  secreta y está versionada directamente en `compose.yaml`.
- `.gitignore`: evita subir archivos generados y entornos virtuales; conserva
  reglas para `.env` por si en el futuro se necesita alguno con secretos.
- `README.md`: instrucciones de instalación, despliegue y operación.

### Carpetas de datos y ejecución

- `data/`: entrada y salida de datos compartida por el master y todos los
  workers. En el host se conserva aunque se eliminen los contenedores.
- `warehouse/`: almacenamiento local compartido por Spark para pruebas con
  filesystem del host. El warehouse principal de Iceberg usa MinIO en
  `s3a://warehouse/iceberg`.
- `jars/`: lugar para JARs adicionales de Spark o Iceberg. Se monta en el
  master como `/opt/spark/jars-extra`.
- `conf/`: contiene `spark-defaults.conf`, con el catálogo Iceberg, Hive
  Metastore, Polaris y el endpoint S3A de MinIO.
- `notebooks/`: reservado para notebooks de exploración y pruebas. Actualmente
  no contiene notebooks.

### Carpeta `docker/`

- `docker/spark/Dockerfile`: construye la imagen usada por el master y los
  workers. Parte de `apache/spark:3.5.6` y agrega JARs de Iceberg y S3A.
- `docker/hive/Dockerfile`: construye la imagen usada por Hive Metastore y
  HiveServer2. Parte de Java 8, descarga `apache-hive-3.1.0-bin.tar.gz` desde
  el archivo oficial de Apache y agrega el driver PostgreSQL y JARs S3A.

## Herramientas y componentes incluidos

| Tool / componente | Versión o definición | Dónde vive | Para qué se usa |
| --- | --- | --- | --- |
| Docker | Instalado en el host | Host | Ejecutar contenedores, listar imágenes, ver logs y limpiar recursos. |
| Docker Compose v2 | Instalado en el host | Host | Construir y operar el clúster definido en `compose.yaml`. |
| Apache Spark | `3.5.6` | Imagen `apache/spark:3.5.6` | Motor distribuido del clúster. |
| Apache Iceberg | `1.7.0` | JAR `iceberg-spark-runtime-3.5_2.12` | Formato de tablas lakehouse usado desde Spark. |
| Iceberg AWS bundle | `1.7.0` | JAR `iceberg-aws-bundle` | Integración Iceberg con almacenamiento S3 compatible. |
| Hadoop S3A | `hadoop-aws:3.3.4` | JAR en Spark y Hive | Permite leer/escribir rutas `s3a://...` hacia MinIO. |
| Apache Polaris | `latest` | Servicio `polaris` | Catálogo REST moderno para tablas Iceberg. |
| Polaris setup | `alpine/curl:8.21.0` | Servicio `polaris-setup` | Crea el catálogo `lakehouse` apuntando a MinIO. |
| Spark Master | `org.apache.spark.deploy.master.Master` | Servicio `spark-master` | Coordina workers y recibe aplicaciones Spark. |
| Spark Worker | `org.apache.spark.deploy.worker.Worker` | Servicio `spark-worker` | Ejecuta tareas Spark con los recursos definidos en `x-cluster-config`. |
| Spark Master UI | Puerto `8080` | `spark-master` | Ver estado del clúster, workers conectados y aplicaciones. |
| Spark Standalone endpoint | Puerto `7077` | `spark-master` | Recibir clientes Spark externos con `spark://<host>:7077`. |
| `spark-submit` | Incluido con Spark | `/opt/spark/bin/spark-submit` | Enviar jobs al clúster desde dentro del contenedor master. |
| `spark-class` | Incluido con Spark | `/opt/spark/bin/spark-class` | Arrancar los procesos internos de master y worker. |
| `pyspark` | Línea compatible `3.5.x` | Cliente externo o imagen Spark | Conectar proyectos Python al clúster. |
| Java runtime | Incluido por la imagen base de Spark | Imagen `apache/spark:3.5.6` | Requisito de ejecución de Spark. |
| MinIO | `RELEASE.2025-04-22T22-12-26Z` | Servicio `minio` | Emula S3 local para guardar datos del lakehouse. |
| MinIO Client | `RELEASE.2025-04-16T18-13-26Z` | Servicio `minio-init` | Crea automáticamente el bucket `warehouse`. |
| Hive Metastore | Hive `3.1.0` | Servicio `hive-metastore` | Catálogo central para tablas Iceberg/Hive vía Thrift. |
| HiveServer2 | Hive `3.1.0` | Servicio `hiveserver2` | Endpoint SQL Hive/Beeline para pruebas con Hive. |
| PostgreSQL | `16-alpine` | Servicio `hive-postgres` | Base de datos persistente para metadatos del metastore. |
| ClickHouse | `24.8.4.13` | Servicio `clickhouse` | Base columnar para analítica y pruebas de integración. |
| Bind mount `data/` | Carpeta local del host | `/data` en contenedores | Compartir entradas y salidas entre master y workers. |
| Bind mount `warehouse/` | Carpeta local del host | `/warehouse` en contenedores | Persistir pruebas locales fuera de MinIO. |
| Bind mount `jars/` | Carpeta local del host | `/opt/spark/jars-extra` en master | Agregar JARs externos al entorno del master. |
| Volumen `minio_data` | Volumen Docker | `/data` en MinIO | Persistir objetos S3, incluyendo `s3a://warehouse/iceberg`. |
| Volumen `hive_postgres_data` | Volumen Docker | PostgreSQL | Persistir metadatos de bases, tablas y particiones. |
| Volumen `clickhouse_data` | Volumen Docker | ClickHouse | Persistir bases y tablas de ClickHouse. |

## Arrancar todo

Desde la raíz del repositorio (`spark-cluster-lab/`):

```bash
docker compose up --build -d
```

El número de workers y sus límites de RAM/CPU salen de `x-cluster-config` en
`compose.yaml` (ver [Configuración del clúster](#configuración-del-clúster)).
La primera ejecución construye las imágenes locales de Spark y Hive. Para ver
el arranque en primer plano:

```bash
docker compose up --build
```

Servicios disponibles:

| Servicio | URL o endpoint desde el host | Endpoint interno Compose | Uso |
| --- | --- | --- | --- |
| Spark Master UI | <http://localhost:8080> | `http://spark-master:8080` | Ver workers y aplicaciones. |
| Spark Master | `spark://localhost:7077` | `spark://spark-master:7077` | Enviar jobs Spark. |
| MinIO S3 API | `http://localhost:9003` | `http://minio:9000` | Almacenamiento S3 compatible. |
| MinIO Console | <http://localhost:9001> | `http://minio:9001` | UI web de buckets y objetos. |
| Polaris REST Catalog | `http://localhost:8181/api/catalog` | `http://polaris:8181/api/catalog` | Catálogo Iceberg vía REST. |
| Polaris Management | `http://localhost:8181/api/management/v1` | `http://polaris:8181/api/management/v1` | Administración de catálogos y principals. |
| Polaris Health/Metrics | <http://localhost:8182/q/health> | `http://polaris:8182` | Healthcheck y métricas internas. |
| Hive Metastore | `thrift://localhost:9083` | `thrift://hive-metastore:9083` | Catálogo de tablas. |
| HiveServer2 | `jdbc:hive2://localhost:10000` | `jdbc:hive2://hiveserver2:10000` | SQL Hive/Beeline. |
| HiveServer2 UI | <http://localhost:10002> | `http://hiveserver2:10002` | UI web de HiveServer2. |
| ClickHouse HTTP | <http://localhost:8123> | `http://clickhouse:8123` | API HTTP de ClickHouse. |
| ClickHouse Native | `localhost:9010` | `clickhouse:9000` | Cliente nativo de ClickHouse. |

Credenciales locales:

| Servicio | Usuario | Password |
| --- | --- | --- |
| MinIO | `minioadmin` | `minioadmin` |
| Polaris root | `root` | `s3cr3t` |
| Hive PostgreSQL | `hive` | `hive` |
| ClickHouse | `default` | sin password |

Comprueba que los servicios estén levantados:

```bash
docker compose ps
docker compose logs --tail=100 spark-master spark-worker polaris hive-metastore minio clickhouse
```

En la UI del master debe aparecer un worker conectado.

## Almacenamiento lakehouse

Este lab tiene tres capas de almacenamiento:

| Capa | Ruta o volumen | Qué guarda | Persistencia |
| --- | --- | --- | --- |
| S3 compatible | `s3a://warehouse/iceberg` y `s3a://warehouse/polaris` en MinIO | Tablas Iceberg y datos lakehouse principales. | Volumen Docker `minio_data`. |
| Metadatos | PostgreSQL de Hive Metastore | Catálogos, tablas, particiones y ubicaciones. | Volumen Docker `hive_postgres_data`. |
| Filesystem local | `./data` y `./warehouse` | Datos de entrada/salida para pruebas simples. | Carpetas del host. |

MinIO emula S3. No hay NameNode/DataNode de Hadoop HDFS en este Compose; Spark y
Hive acceden al almacenamiento de objetos mediante el conector Hadoop S3A.

El bucket `warehouse` se crea automáticamente con el servicio `minio-init`.
Dentro de ese bucket:

- `s3a://warehouse/iceberg` es el warehouse del catálogo `iceberg`, respaldado
  por Hive Metastore.
- `s3a://warehouse/polaris` es el warehouse del catálogo `polaris`, respaldado
  por Apache Polaris REST Catalog.
- `s3a://warehouse/hive` queda reservado como warehouse Hive.

La configuración de Spark está en `conf/spark-defaults.conf`:

```properties
spark.sql.catalog.iceberg.type hive
spark.sql.catalog.iceberg.uri thrift://hive-metastore:9083
spark.sql.catalog.iceberg.warehouse s3a://warehouse/iceberg
spark.sql.catalog.polaris.catalog-impl org.apache.iceberg.rest.RESTCatalog
spark.sql.catalog.polaris.uri http://polaris:8181/api/catalog
spark.sql.catalog.polaris.warehouse lakehouse
spark.hadoop.fs.s3a.endpoint http://minio:9000
```

## Comandos de Docker Compose

Ejecuta estos comandos desde el servidor, en `~/spark-cluster-lab`.

```bash
# Validar y mostrar la configuración final
docker compose config

# Construir imágenes
docker compose build
docker compose build --no-cache

# Iniciar en segundo plano o en primer plano
docker compose up -d
docker compose up
docker compose up --build -d

# Estado y logs
docker compose ps
docker compose logs -f
docker compose logs -f spark-master
docker compose logs --tail=100 spark-worker

# Abrir una shell o ejecutar un comando en un servicio
docker compose exec spark-master bash
docker compose exec spark-master /opt/spark/bin/spark-submit --version

# Reiniciar un servicio
docker compose restart spark-master
docker compose restart spark-worker

# Detener y eliminar contenedores y red del proyecto
docker compose down

# Detener y eliminar también las imágenes locales del proyecto
docker compose down --rmi local

# Detener y eliminar contenedores, red y volúmenes del proyecto
docker compose down -v

# Ver imágenes usadas por los servicios del Compose
docker compose images

# Ver procesos dentro de los servicios
docker compose top
```

## Ejecutar un job Spark

Para lanzar un script Python desde el host contra el master, usa `spark-submit`
de una instalación local de Spark compatible con la imagen, o ejecuta el
comando dentro del contenedor:

```bash
docker compose exec spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  /ruta/al/job.py
```

Si el job está en otro proyecto, monta o copia su ruta dentro del servicio
antes de ejecutar `spark-submit`; el Compose actual monta `data/`, `warehouse/`
y `jars/`.

## Probar Iceberg sobre MinIO

Abre una sesión SQL de Spark dentro del master:

```bash
docker compose exec spark-master /opt/spark/bin/spark-sql \
  --master spark://spark-master:7077
```

Crea una tabla Iceberg usando Hive Metastore como catálogo:

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

Prueba también Polaris como catálogo REST de Iceberg:

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

Los archivos de esas tablas quedan en MinIO bajo el bucket `warehouse`, en los
prefijos `iceberg/` y `polaris/`. Puedes verlos en
<http://localhost:9001> con usuario y password `minioadmin`.

## Usar Hive y ClickHouse

Hive Metastore queda disponible para Spark/Iceberg en:

```text
thrift://hive-metastore:9083
```

Polaris queda disponible como catálogo REST de Iceberg en:

```text
http://polaris:8181/api/catalog
```

Para abrir Beeline contra HiveServer2:

```bash
docker compose exec hiveserver2 beeline -u jdbc:hive2://localhost:10000
```

Para probar ClickHouse por HTTP:

```bash
curl 'http://localhost:8123/?query=SELECT%201'
```

Para entrar al cliente nativo de ClickHouse:

```bash
docker compose exec clickhouse clickhouse-client
```

## Usar el clúster desde otros proyectos

Este repositorio levanta la infraestructura compartida del lab; no está
pensado para contener la lógica de negocio. Cualquier proyecto, en otra carpeta
o máquina, se conecta como cliente Spark sin pertenecer a este Compose.

Requisitos para que un proyecto externo se conecte:

- Alcanzar por red el puerto `7077` del host donde corre `spark-master`
  (mismo host: `localhost`; otra máquina: la IP del servidor, si el firewall
  lo permite).
- Fijar su `pyspark` a la misma línea `3.5.x` que el clúster (ver
  [Notas de compatibilidad](#notas-de-compatibilidad)); una versión mayor o
  menor distinta falla al conectar por el protocolo Standalone.
- Apuntar el `master` de su `SparkSession`/`SparkContext` a
  `spark://<host>:7077`.

Ejemplo mínimo desde un proyecto externo con PySpark instalado:

```python
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("mi-otro-proyecto")
    .master("spark://localhost:7077")  # o spark://<ip-del-servidor>:7077
    .getOrCreate()
)
```

Este clúster expone MinIO como almacenamiento remoto S3 compatible. Para
proyectos externos, usa `http://<host>:9003` como endpoint S3, bucket
`warehouse`, access key `minioadmin` y secret key `minioadmin`. Si el proyecto
corre fuera de la red de Compose, usa `spark://<host>:7077`,
`thrift://<host>:9083` y `s3a://warehouse/iceberg` con endpoint
`http://<host>:9003`.

## Parar y limpiar

Parar los contenedores manteniendo los datos locales:

```bash
docker compose down
```

Ese comando elimina los contenedores del stack y la red creada por Compose. Es
el comando recomendado cuando ya no necesitas tener el lab levantado.

Si quieres asegurarte de borrar contenedores huérfanos de ejecuciones previas:

```bash
docker compose down --remove-orphans
```

Parar y eliminar también las imágenes construidas por este Compose:

```bash
docker compose down --rmi local
```

Compose elimina bien los contenedores, la red, los volúmenes y las imágenes
construidas por el proyecto. Las imágenes base descargadas desde registros
pueden estar compartidas con otros proyectos; si quieres borrarlas, primero
revisa `docker compose images` y luego elimina explícitamente las que ya no
uses:

```bash
docker image rm apache/spark:3.5.6
docker image rm spark-cluster-lab-hive:3.1.0
docker image rm apache/polaris:latest
docker image rm postgres:16-alpine
docker image rm minio/minio:RELEASE.2025-04-22T22-12-26Z
docker image rm minio/mc:RELEASE.2025-04-16T18-13-26Z
docker image rm clickhouse/clickhouse-server:24.8.4.13
```

Si Docker indica que alguna imagen está en uso, primero detén y elimina los
contenedores con `docker compose down`, y luego vuelve a ejecutar
`docker image rm`.

Los directorios `data/`, `warehouse/` y `jars/` son bind mounts del host, por
lo que `docker compose down` no borra su contenido. Los datos de MinIO,
PostgreSQL y ClickHouse viven en volúmenes Docker. Para eliminar también esos
volúmenes:

```bash
docker compose down -v
```

Para eliminar los datos locales del host, hazlo explícitamente y con cuidado:

```bash
rm -rf data/* warehouse/*
```

## Notas de compatibilidad

- El contenedor usa Apache Spark 3.5.6.
- Iceberg se instala en Spark con `iceberg-spark-runtime-3.5_2.12:1.7.0` y
  `iceberg-aws-bundle:1.7.0`.
- Polaris se levanta como catálogo REST local para desarrollo. El catálogo
  `lakehouse` se crea automáticamente con `polaris-setup` y apunta a MinIO.
- MinIO emula S3; este Compose no levanta HDFS.
- El protocolo Standalone de Spark (`spark://host:puerto`) exige que cliente y
  clúster compartan versión mayor.menor de Spark. Cualquier proyecto cliente
  debe fijar su versión de PySpark a `3.5.x` (por ejemplo,
  `pyspark>=3.5.6,<3.6`) para conectarse a este clúster sin errores de
  serialización.
- Aunque existen carpetas `jars/` y `notebooks/`, actualmente no hay JAR ni
  notebook incluido en el repositorio. Los JARs base de Iceberg/S3A se agregan
  en los Dockerfiles.

## Solución rápida de problemas

Si algún puerto está ocupado, detén el proceso que lo usa o cambia el mapeo de
puertos en `compose.yaml`. Los puertos publicados por defecto son `7077`,
`8080`, `8181`, `8182`, `9001`, `9003`, `9010`, `9083`, `10000`, `10002`, `8123`,
y `5433`.

Si el worker no aparece en la UI o alguno de los servicios no arranca, revisa
los logs:

```bash
docker compose logs spark-master spark-worker hive-metastore hiveserver2 minio clickhouse
```

Para reconstruir la imagen sin caché:

```bash
docker compose build --no-cache
docker compose up -d
```
