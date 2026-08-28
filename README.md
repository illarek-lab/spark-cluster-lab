# Spark Cluster Lab

Infraestructura para levantar un clúster pequeño de Apache Spark con Docker
Compose. Este repositorio solo define el clúster (master + workers); los
proyectos que lo usan como cliente viven fuera de aquí.

## Requisitos

- macOS, Linux o Windows con Docker Desktop instalado y en ejecución.
- Docker Compose v2 (`docker compose version`).

Comprueba Docker antes de empezar:

```bash
docker --version
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
docker --version
docker compose version
docker compose up --build -d
docker compose ps
```

La interfaz web estará disponible en `http://100.85.61.29:8080` si el puerto
está permitido por el firewall y la red del servidor. El endpoint Spark será
`spark://100.85.61.29:7077` para clientes externos. Desde los propios
contenedores se usa `spark://spark-master:7077`.

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
- `warehouse/`: ubicación del almacén de tablas, por ejemplo tablas Iceberg.
  También se monta en todos los nodos para que compartan el mismo contenido.
- `jars/`: lugar para JARs adicionales de Spark o Iceberg. Se monta en el
  master como `/opt/spark/jars-extra`.
- `conf/`: reservado para archivos de configuración de Spark, catálogos y
  otras configuraciones del laboratorio. Actualmente no contiene archivos.
- `notebooks/`: reservado para notebooks de exploración y pruebas. Actualmente
  no contiene notebooks.

### Carpeta `docker/`

- `docker/spark/Dockerfile`: construye la imagen usada por el master y los
  workers. Parte de `apache/spark:3.5.6`.
- `docker/minio/`: reservada para una futura definición de MinIO (almacenamiento
  compatible con S3). Actualmente está vacía y no hay ningún servicio `minio`
  en `compose.yaml`.

## Herramientas y componentes incluidos

| Tool / componente | Versión o definición | Dónde vive | Para qué se usa |
| --- | --- | --- | --- |
| Docker | Instalado en el host | Host | Ejecutar contenedores, listar imágenes, ver logs y limpiar recursos. |
| Docker Compose v2 | Instalado en el host | Host | Construir y operar el clúster definido en `compose.yaml`. |
| Apache Spark | `3.5.6` | Imagen `apache/spark:3.5.6` | Motor distribuido del clúster. |
| Spark Master | `org.apache.spark.deploy.master.Master` | Servicio `spark-master` | Coordina workers y recibe aplicaciones Spark. |
| Spark Worker | `org.apache.spark.deploy.worker.Worker` | Servicio `spark-worker` | Ejecuta tareas Spark con los recursos definidos en `x-cluster-config`. |
| Spark Master UI | Puerto `8080` | `spark-master` | Ver estado del clúster, workers conectados y aplicaciones. |
| Spark Standalone endpoint | Puerto `7077` | `spark-master` | Recibir clientes Spark externos con `spark://<host>:7077`. |
| `spark-submit` | Incluido con Spark | `/opt/spark/bin/spark-submit` | Enviar jobs al clúster desde dentro del contenedor master. |
| `spark-class` | Incluido con Spark | `/opt/spark/bin/spark-class` | Arrancar los procesos internos de master y worker. |
| `pyspark` | Línea compatible `3.5.x` | Cliente externo o imagen Spark | Conectar proyectos Python al clúster. |
| Java runtime | Incluido por la imagen base de Spark | Imagen `apache/spark:3.5.6` | Requisito de ejecución de Spark. |
| Bind mount `data/` | Carpeta local del host | `/data` en contenedores | Compartir entradas y salidas entre master y workers. |
| Bind mount `warehouse/` | Carpeta local del host | `/warehouse` en contenedores | Persistir tablas o datos administrados por jobs. |
| Bind mount `jars/` | Carpeta local del host | `/opt/spark/jars-extra` en master | Agregar JARs externos al entorno del master. |

## Arrancar todo

Desde la raíz del repositorio (`spark-cluster-lab/`):

```bash
docker compose up --build -d
```

El número de workers y sus límites de RAM/CPU salen de `x-cluster-config` en
`compose.yaml` (ver [Configuración del clúster](#configuración-del-clúster)).
La primera ejecución construye la imagen de Spark. Para ver el arranque en
primer plano:

```bash
docker compose up --build
```

Servicios disponibles:

- Spark Master UI: <http://localhost:8080>
- Spark Master: `spark://localhost:7077` desde el host, o
  `spark://spark-master:7077` desde otro contenedor de Compose.

Comprueba que ambos servicios están levantados:

```bash
docker compose ps
docker compose logs --tail=100 spark-master spark-worker
```

En la UI del master debe aparecer un worker conectado.

## Comandos de Docker y Compose

Ejecuta estos comandos desde el servidor, en `~/spark-cluster-lab` cuando
indiquen `docker compose`.

### Docker

```bash
# Información del motor y versión instalada
docker version
docker info

# Imágenes locales
docker image ls
docker image prune

# Contenedores en ejecución o detenidos
docker ps
docker ps -a

# Logs y procesos de un contenedor
docker logs -f spark-master
docker logs --tail=100 spark-worker
docker inspect spark-master
docker top spark-master

# Ejecutar una shell dentro del contenedor
docker exec -it spark-master bash

# Recursos consumidos
docker stats

# Redes y volúmenes
docker network ls
docker volume ls

# Eliminar un contenedor detenido o una imagen
docker rm NOMBRE_O_ID
docker rmi IMAGEN_O_ID
```

### Docker Compose

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

## Usar el clúster desde otros proyectos

Este repositorio solo levanta el clúster (master + workers); no está pensado
para contener la lógica de negocio. Cualquier proyecto, en otra carpeta o
máquina, se conecta como cliente Spark sin pertenecer a este Compose.

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

Este clúster no expone un servicio de almacenamiento remoto: `data/` y
`warehouse/` son bind mounts del host donde corre Docker, visibles como
`/data` y `/warehouse` dentro de los contenedores. Un proyecto que corre en
otra máquina no ve esas rutas directamente; para compartir datos entre
máquinas hace falta un almacenamiento accesible por red (por ejemplo, el
`docker/minio/` reservado, aún sin implementar) en vez de depender del
filesystem del host.

## Parar y limpiar

Parar los contenedores manteniendo los datos locales:

```bash
docker compose down
```

Ese comando elimina los contenedores de `spark-master`, las réplicas de
`spark-worker` y la red creada por Compose. Es el comando recomendado cuando ya
no necesitas tener el clúster levantado.

Si quieres asegurarte de borrar contenedores huérfanos de ejecuciones previas:

```bash
docker compose down --remove-orphans
```

Parar y eliminar también las imágenes construidas por este Compose:

```bash
docker compose down --rmi local
```

Eliminar contenedores detenidos manualmente, si quedaron fuera de Compose:

```bash
docker ps -a
docker rm NOMBRE_O_ID
```

Limpiar recursos Docker no usados por ningún proyecto:

```bash
docker system prune
```

Los directorios `data/`, `warehouse/` y `jars/` son bind mounts del host, por
lo que `docker compose down` no borra su contenido. Para eliminar los datos,
hazlo explícitamente y con cuidado:

```bash
rm -rf data/* warehouse/*
```

## Notas de compatibilidad

- El contenedor usa Apache Spark 3.5.6.
- El protocolo Standalone de Spark (`spark://host:puerto`) exige que cliente y
  clúster compartan versión mayor.menor de Spark. Cualquier proyecto cliente
  debe fijar su versión de PySpark a `3.5.x` (por ejemplo,
  `pyspark>=3.5.6,<3.6`) para conectarse a este clúster sin errores de
  serialización.
- Aunque existen carpetas `conf/`, `jars/` y `notebooks/`, actualmente no hay
  una configuración adicional, JAR ni notebook incluido en el repositorio.

## Solución rápida de problemas

Si los puertos `7077` o `8080` están ocupados, detén el proceso que los usa o
cambia el mapeo de puertos en `compose.yaml` (por ejemplo, `18080:8080`).

Si el worker no aparece en la UI, revisa los logs:

```bash
docker compose logs spark-master spark-worker
```

Para reconstruir la imagen sin caché:

```bash
docker compose build --no-cache
docker compose up -d
```
