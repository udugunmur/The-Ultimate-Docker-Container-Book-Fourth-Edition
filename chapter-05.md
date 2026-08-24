# Parte 2: Fundamentos de la Contenerización
## Capítulo 5: Volúmenes de datos y configuración

En el capítulo anterior, aprendimos a construir y compartir nuestras imágenes de contenedores, haciendo hincapié en cómo crear imágenes lo más reducidas posible incluyendo únicamente los artefactos requeridos por la aplicación.

En este capítulo, aprenderemos a trabajar con **contenedores con estado (*stateful*)** —es decir, contenedores que consumen y producen datos—. También aprenderemos a configurar nuestros contenedores en tiempo de ejecución (*runtime*) y en tiempo de construcción de imágenes (*build time*) mediante variables de entorno y archivos de configuración.

---

### Temas tratados en este capítulo:
- Creación y montaje de volúmenes de datos
- Compartir datos entre contenedores
- Uso de volúmenes del host (*host volumes / bind mounts*)
- Definición de volúmenes en imágenes
- Configuración de contenedores
- Almacenamiento persistente y patrones de contenedores con estado

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Crear, eliminar y listar volúmenes de datos de Docker.
- Montar un volumen de datos existente en un contenedor.
- Generar datos persistentes desde un contenedor mediante volúmenes.
- Compartir datos entre múltiples contenedores usando volúmenes de datos.
- Montar cualquier carpeta del host dentro de un contenedor (*bind mount*).
- Definir el modo de acceso (lectura/escritura o solo lectura) para un contenedor al acceder a los datos de un volumen.
- Configurar variables de entorno para aplicaciones que se ejecutan en un contenedor.
- Parametrizar un Dockerfile utilizando argumentos de compilación (*build arguments*).

---

### Requisitos técnicos

Para este capítulo necesitas tener instalado Docker Desktop en tu equipo. No hay código complementario específico para descargar; crearemos la estructura directamente:

```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir chapter-05 && cd chapter-05
```

¡Comencemos!

---

### Creación y montaje de volúmenes de datos

Cualquier aplicación real consume o produce datos. Sin embargo, los contenedores están diseñados preferentemente para ser **sin estado (*stateless*)** y efímeros. Para gestionar la persistencia, Docker ofrece **volúmenes (*Docker volumes*)**. Los volúmenes permiten a los contenedores consumir, generar y modificar estado. Tienen un ciclo de vida independiente del ciclo de vida del contenedor: cuando un contenedor se elimina, el volumen continúa existiendo.

#### Modificación de la capa del contenedor
Cuando una aplicación dentro de un contenedor escribe en el sistema de archivos, las modificaciones ocurren en la **capa escribible del contenedor (*writable container layer*)**:

```bash
$ docker container run --name demo alpine \
  /bin/sh -c 'echo "This is a test" > sample.txt'
```

Comprobamos los cambios respecto a la imagen base:

```bash
$ docker container diff demo
```

Salida:
```text
A /sample.txt
```

La letra `A` (*Added*) indica que se añadió un archivo en la capa escribible del contenedor. Si eliminamos el contenedor, la capa escribible desaparece y los datos se pierden de forma irreversible. Para evitar esto, utilizamos volúmenes de Docker.

#### Creación de volúmenes

1. Crear un volumen llamado `sample`:
   ```bash
   $ docker volume create sample
   ```
   Salida:
   ```text
   sample
   ```

2. Inspeccionar el volumen para conocer su ubicación en el host:
   ```bash
   $ docker volume inspect sample
   ```
   *Figura 5.1: Inspeccionando el volumen Docker llamado sample*

   El campo `Mountpoint` indica la ruta de almacenamiento en el host (por ejemplo, `/var/lib/docker/volumes/sample/_data`).

*(También es posible crear volúmenes desde la interfaz gráfica de Docker Desktop en la pestaña **Volumes** > **Create Volume**).*

*Figura 5.2: Creación de un nuevo volumen Docker con Docker Desktop*  
*Figura 5.3: Lista de volúmenes Docker mostrada en Docker Desktop*

#### Controladores de volumen (*Volume Drivers*)
El controlador predeterminado es `local` (almacenamiento en el sistema de archivos del host). Existen controladores adicionales proporcionados por la comunidad y fabricantes:

| Controlador (Driver) | Fabricante | Descripción y uso principal |
| :--- | :--- | :--- |
| **`local`** | Docker | Driver predeterminado que almacena datos en el host local. Adecuado para desarrollo y despliegues en un solo nodo. |
| **`nfs`** | Docker | Utiliza Network File System (NFS) para compartir volúmenes entre múltiples hosts. |
| **`cifs`** | Docker | Utiliza el protocolo CIFS/Samba para montar recursos compartidos basados en Windows. |
| **`rclone`** | Rclone | Permite montar servicios de almacenamiento en la nube (Google Drive, Dropbox, etc.) como volúmenes. |
| **`rexray/ebs`** | REX-Ray | Integra volúmenes persistentes en bloques de AWS EBS. |
| **`flocker`** | ClusterHQ | Gestiona volúmenes para clústeres facilitando la migración de datos entre hosts. |
| **`portworx`** | Portworx | Almacenamiento granular de alto rendimiento para contenedores empresariales (replicación, snapshots, cifrado). |
| **`glusterfs`** | Gluster | Sistema de archivos de red distribuido y escalable. |
| **`azurefile`** | Microsoft Azure | Conecta contenedores con Azure File Storage. |
| **`gce-pd`** | Google Cloud | Integra Persistent Disks de Google Compute Engine con Docker. |

*Tabla 5.1: Controladores de volumen populares en Docker*

---

### Montaje de un volumen en un contenedor

1. Montar el volumen `sample` en la carpeta `/data` de un contenedor Alpine:
   ```bash
   $ docker container run --name test -it \
     -v sample:/data \
     alpine /bin/sh
   ```

2. Crear archivos dentro de la carpeta montada:
   ```bash
   / # cd /data
   /data # echo "Some data" > data.txt
   /data # echo "Some more data" > data2.txt
   ```
   Sal del contenedor con `Ctrl + D`.

3. Eliminar el contenedor `test`:
   ```bash
   $ docker container rm test
   ```

4. Montar el **mismo volumen** en un contenedor completamente distinto (CentOS 7) y en una ruta diferente (`/app/data`):
   ```bash
   $ docker container run --name test2 -it --rm \
     -v sample:/app/data \
     centos:7 /bin/bash
   ```
   *Figura 5.4: Montaje del volumen sample en un contenedor CentOS 7*

5. Comprobar los archivos dentro de `/app/data`:
   *Figura 5.5: Listado de archivos del volumen sample dentro del contenedor CentOS*

   Los archivos `data.txt` y `data2.txt` creados por el contenedor Alpine están intactos.

> **Importante**: La carpeta del contenedor donde se monta un volumen queda excluida del *Union filesystem*. Cualquier cambio realizado en esa carpeta se escribe directamente en el almacenamiento respaldado por el volumen y no en la capa escribible del contenedor.

Sal del contenedor con `Ctrl + D`.

---

### Eliminación de volúmenes

Para eliminar un volumen:
```bash
$ docker volume rm sample
```
*(Docker impedirá la eliminación si el volumen está siendo utilizado por algún contenedor activo o detenido).*

- **Listar volúmenes existentes**:
  ```bash
  $ docker volume ls
  ```
- **Eliminar contenedores y sus volúmenes anónimos asociados**:
  ```bash
  $ docker container rm -v -f $(docker container ls -aq)
  ```

---

### Acceso a la estructura interna de volúmenes en Docker Desktop (macOS / Windows)

En macOS y Windows, los contenedores se ejecutan dentro de una máquina virtual ligera gestionada por Docker Desktop. Para acceder directamente al sistema de archivos raíz de esa VM donde residen los volúmenes, podemos utilizar un contenedor con privilegios especiales y la herramienta `nsenter`:

1. Crear dos volúmenes de prueba:
   ```bash
   $ docker volume create sample
   $ docker volume create sample-2
   ```

2. En macOS/Windows, la ruta `/var/lib/docker/volumes/` no existe directamente en el host:
   ```bash
   $ cd /var/lib/docker/volumes/sample/_data
   ```
   *Figura 5.6: Acceso al directorio de volúmenes en un Mac*

3. Acceder al espacio de nombres de la VM del host mediante un contenedor Debian privilegiado:
   ```bash
   $ docker container run -it --privileged --pid=host \
     debian nsenter -t 1 -m -u -n -i sh
   ```

4. Listar los volúmenes dentro de la VM:
   ```bash
   / # ls -l /var/lib/docker/volumes
   ```
   *Figura 5.7: Lista de volúmenes Docker vía nsenter*

5. Inspeccionar el contenido del volumen `sample`:
   ```bash
   / # cd /var/lib/docker/volumes/sample/_data
   /var/lib/docker/volumes/sample/_data # ls -l
   ```
   *Figura 5.8: Lista de archivos en el volumen sample*

6. En otra terminal, escribir datos en el volumen desde un contenedor:
   ```bash
   $ docker container run --rm -it \
     -v sample:/data alpine /bin/sh
   ```
   ```bash
   / # echo "Hello world" > /data/sample.txt
   / # echo "Other message" > /data/other.txt
   ```
   Sal del contenedor con `Ctrl + D`.

7. Verificar en la sesión de `nsenter`:
   ```bash
   / # cd /var/lib/docker/volumes/sample/_data
   / # ls -al
   ```
   *Figura 5.9: Volumen sample conteniendo los archivos creados en el contenedor Alpine*

8. Escribir un archivo directamente desde el host hacia el volumen:
   ```bash
   / # echo "I love Docker" > docker.txt
   / # ls -l
   ```
   *Figura 5.10: Volumen con archivo generado directamente en el host*

9. Leer el nuevo archivo desde otro contenedor:
   ```bash
   $ docker container run --rm \
     -v sample:/data \
     centos:7 ls -l /data
   ```
   *Figura 5.11: Lista de archivos observada desde el contenedor*

Para salir de la sesión privilegiada de `nsenter`, presiona `Ctrl + D` dos veces.

---

### Compartir datos entre contenedores

Para evitar condiciones de carrera (*race conditions*) cuando múltiples contenedores acceden a un volumen compartido, es una buena práctica designar un único contenedor escritor y montar el volumen en **modo de solo lectura (`:ro`)** para los contenedores lectores.

1. Iniciar el contenedor escritor (modo lectura/escritura por defecto):
   ```bash
   $ docker container run -it --name writer \
     -v shared-data:/data \
     alpine /bin/sh
   ```
   Crear un archivo:
   ```bash
   / # echo "I can create a file" > /data/sample.txt
   ```
   Salir con `Ctrl + D`.

2. Iniciar el contenedor lector con el volumen montado en solo lectura (`:ro`):
   ```bash
   $ docker container run -it --name reader \
     -v shared-data:/app/data:ro \
     ubuntu:25.04 /bin/bash
   ```

3. Verificar la lectura de archivos:
   ```bash
   $ ls -l /app/data
   ```
   *Figura 5.12: Listado de archivos de un volumen de solo lectura*

4. Intentar escribir en el volumen de solo lectura:
   ```bash
   # echo "Try to break read/only" > /app/data/data.txt
   ```
   Salida generada:
   ```text
   bash: /app/data/data.txt: Read-only file system
   ```

5. Limpieza de contenedores y volúmenes:
   ```bash
   $ docker container rm -f $(docker container ls -aq)
   $ docker volume rm $(docker volume ls -q)
   ```

---

### Uso de volúmenes del host (*Bind Mounts*)

Los **montajes vinculados (*bind mounts*)** montan una ruta absoluta específica del sistema de archivos del host dentro del contenedor. Son ampliamente utilizados por desarrolladores para lograr una experiencia de edición y continuación (*edit-and-continue*) sin reconstruir la imagen en cada cambio de código.

```bash
$ docker container run --rm -it \
  -v $(pwd)/src:/app/src \
  alpine:latest /bin/sh
```

*Figura 5.13: Permiso del sistema operativo para acceder a carpetas locales*

#### Ejemplo práctico: Desarrollo web con Nginx y Bind Mounts
1. Crear el proyecto web local:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
   $ cd chapter-05
   $ mkdir my-web && cd my-web
   $ echo "<h1>Personal Website</h1>" > index.html
   ```

2. Crear un archivo `Dockerfile`:
   ```dockerfile
   FROM nginx:alpine
   COPY . /usr/share/nginx/html
   ```

3. Construir la imagen:
   ```bash
   $ docker image build -t my-website:1.0 .
   ```
   *Figura 5.14: Construcción de la imagen Docker para el servidor web Nginx*

4. Ejecutar el contenedor montando el directorio actual con un bind mount:
   ```bash
   $ docker container run -d \
     --name my-site \
     -v $(pwd):/usr/share/nginx/html \
     -p 8080:80 \
     my-website:1.0
   ```

5. Accede a `http://localhost:8080/index.html`. Cualquier cambio que realices en el archivo local `index.html` en tu editor se reflejará inmediatamente en el navegador al recargar, sin reiniciar el contenedor.

6. Limpieza:
   ```bash
   $ docker container rm -f my-site
   ```

---

### Definición de volúmenes en imágenes

La instrucción **`VOLUME`** en un Dockerfile declara que ciertas rutas del contenedor deben ser gestionadas como volúmenes persistentes desacoplados del *Union filesystem*.

Sintaxis en Dockerfile:
```dockerfile
VOLUME /app/data
VOLUME /app/data, /app/profiles, /app/config
VOLUME ["/app/data", "/app/profiles", "/app/config"]
```

#### Ejemplo con MongoDB:
1. Descargar la imagen oficial de MongoDB:
   ```bash
   $ docker image pull mongo:8.0.8
   ```
   *Figura 5.15: Descarga de la imagen de MongoDB*

2. Inspeccionar las definiciones de volúmenes en la imagen:
   ```bash
   $ docker image inspect \
     --format='{{json .Config.Volumes}}' \
     mongo:8.0.8 | jq .
   ```
   *Figura 5.16: Sección de volúmenes de la configuración de MongoDB*

   *(Muestra que MongoDB declara `/data/configdb` y `/data/db` como volúmenes).*

3. Ejecutar MongoDB e inspeccionar los puntos de montaje creados automáticamente:
   ```bash
   $ docker run --name my-mongo -d mongo:8.0.8
   $ docker inspect --format '{{json .Mounts}}' my-mongo | jq .
   ```
   *Figura 5.17: Inspeccionando los volúmenes de MongoDB*

4. Limpieza:
   ```bash
   $ docker rm -f my-mongo
   ```

---

### Configuración de contenedores

Las variables de entorno permiten desacoplar la configuración del código empaquetado, permitiendo que la misma imagen se ejecute en desarrollo, pruebas y producción.

#### Definir variables con `--env` o `-e` en tiempo de ejecución:
```bash
$ docker container run --rm -it \
  --env LOG_DIR=/var/log/my-log \
  --env MAX_LOG_FILES=5 \
  --env MAX_LOG_SIZE=1G \
  alpine /bin/sh
```
Comprobación dentro del contenedor:
```bash
/ # export | grep LOG
```
*Figura 5.19: Variables de entorno definidas mediante el parámetro --env*

#### Uso de archivos de configuración con `--env-file`:
1. Crear el subdirectorio y el archivo de configuración:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
   $ cd chapter-05
   $ mkdir config-file && cd config-file
   ```
2. Crear el archivo `development.config`:
   ```text
   LOG_DIR=/var/log/my-log
   MAX_LOG_FILES=5
   MAX_LOG_SIZE=1G
   ```
3. Ejecutar el contenedor pasando el archivo:
   ```bash
   $ docker container run --rm -it \
     --env-file ./development.config \
     alpine sh -c "export | grep LOG"
   ```
   *Figura 5.20: Uso de un archivo para definir variables de entorno*

#### Definir variables de entorno por defecto en el Dockerfile (`ENV`):
```dockerfile
FROM alpine:latest
ENV LOG_DIR=/var/log/my-log
ENV MAX_LOG_FILES=5
ENV MAX_LOG_SIZE=1G
```

Construcción y ejecución:
```bash
$ docker image build -t my-alpine .
$ docker container run --rm -it \
  my-alpine sh -c "export | grep LOG"
```
*Figura 5.21: Variables de entorno definidas en la imagen Docker*

Sobrescribir valores en tiempo de ejecución:
```bash
$ docker container run --rm -it \
  --env MAX_LOG_SIZE=2G \
  --env MAX_LOG_FILES=10 \
  my-alpine sh -c "export | grep LOG"
```
*Figura 5.22: Variables de entorno sobrescritas*

#### Variables de entorno en tiempo de construcción (`ARG` y `--build-arg`):
Las instrucciones `ARG` permiten parametrizar el proceso de construcción de la imagen:

```dockerfile
ARG BASE_IMAGE_VERSION=12.7-stretch
FROM node:${BASE_IMAGE_VERSION}
WORKDIR /app
COPY packages.json .
RUN npm install
COPY . .
CMD npm start
```

Sobrescribir la versión de la imagen base al compilar:
```bash
$ docker image build \
  --build-arg BASE_IMAGE_VERSION=12.7-alpine \
  -t my-node-app-test .
```

- **`ENV` / `--env` / `--env-file`**: Se evalúan en **tiempo de ejecución (*runtime*)**.
- **`ARG` / `--build-arg`**: Se evalúan en **tiempo de construcción (*build time*)**.

---

### Almacenamiento persistente y patrones de contenedores con estado

#### Comparativa: Volúmenes vs Bind Mounts
- **Volúmenes (*Volumes*)**: Gestionados completamente por Docker (`/var/lib/docker/volumes/`). Son portables, seguros y la opción recomendada para producción y persistencia de bases de datos.
- **Montajes vinculados (*Bind Mounts*)**: Mapean rutas directas del host. Dependen de la estructura del sistema de archivos local; ideales para entornos de desarrollo local.

#### Patrones de gestión de estado:
1. **Volúmenes con nombre (*Named Volumes*)**: Desacoplan los datos del ciclo de vida del contenedor y permiten compartir almacenamiento fácilmente.
2. **Contenedores de volumen de datos (*Data Volume Containers*)**: Patrón histórico sustituido en gran medida por los volúmenes con nombre.
3. **StatefulSets (Kubernetes)**: Para entornos orquestados a escala, gestionan despliegues ordenados asignando volúmenes persistentes únicos a cada Pod.
4. **Plugins de volumen**: Permiten integrar almacenamiento en bloque o distribuido en nubes públicas (AWS EBS, GCP PD, Azure Files, Portworx).

#### Buenas prácticas para almacenamiento persistente:
- Utilizar volúmenes gestionados por Docker para datos en producción.
- Establecer políticas regulares de copia de seguridad (*backup*) y restauración.
- Monitorizar el consumo de espacio en disco.
- Proteger y restringir permisos sobre volúmenes que contengan información confidencial.

---

### Resumen

En este capítulo exploramos:
- La creación, inspección, montaje y eliminación de volúmenes de datos de Docker.
- El comportamiento de la capa escribible del contenedor frente a las carpetas montadas en volúmenes.
- El uso de montajes de host (*bind mounts*) para desarrollo ágil interactivo.
- La declaración de volúmenes en imágenes mediante la instrucción `VOLUME`.
- La configuración dinámica de aplicaciones mediante variables de entorno en runtime (`ENV`, `--env`, `--env-file`) y en tiempo de construcción (`ARG`, `--build-arg`).
- Los patrones arquitectónicos para gestionar cargas de trabajo con estado de manera segura y escalable.

---

### Lecturas adicionales

- **Uso de volúmenes**: [http://dockr.ly/2EUjTml](http://dockr.ly/2EUjTml)
- **Gestión de datos en Docker**: [http://dockr.ly/2EhBpzD](http://dockr.ly/2EhBpzD)
- **Volúmenes en Play with Docker (PWD)**: [http://bit.ly/2sjIfDj](http://bit.ly/2sjIfDj)
- **Página de manual de nsenter en Linux**: [https://bit.ly/2MEPG0n](https://bit.ly/2MEPG0n)
- **Establecer variables de entorno**: [https://docs.docker.com/reference/cli/docker/](https://docs.docker.com/reference/cli/docker/)
- **Interacción entre ARG y FROM**: [https://dockr.ly/2OrhZgx](https://dockr.ly/2OrhZgx)

---

### Preguntas

1. **¿Cuál es la diferencia principal entre los volúmenes de Docker y los montajes vinculados (*bind mounts*)?**
2. **¿Por qué se recomiendan generalmente los volúmenes sobre los *bind mounts* para la persistencia de datos en Docker?**
3. **¿Cómo garantiza Docker la persistencia de los datos cuando se elimina un contenedor?**
4. **¿Cuál es un caso de uso común para los *bind mounts* en Docker?**
5. **¿Se puede compartir un volumen entre múltiples contenedores? Si es así, ¿cómo?**
6. **¿Cómo crearías un volumen de datos con nombre llamado `my-products` utilizando el controlador por defecto?**
7. **¿Cómo ejecutarías un contenedor Alpine montando el volumen `my-products` en modo de solo lectura dentro de la carpeta `/data`?**
8. **¿Cómo localizarías la carpeta asociada con el volumen `my-products` en el host y crearías un archivo `sample.txt` dentro de ella?**
9. **¿Cómo ejecutarías otro contenedor Alpine montando el volumen `my-products` en la carpeta `/app-data` en modo lectura/escritura y creando un archivo `hello.txt`?**
10. **¿Cómo montarías una carpeta del host como `~/my-project` dentro de un contenedor?**
11. **¿Cómo eliminarías todos los volúmenes no utilizados del sistema?**
12. **¿Cómo puedes inspeccionar los detalles de un volumen de Docker?**
13. **¿Cuáles son las implicaciones de seguridad al utilizar *bind mounts*?**
14. **La lista de variables de entorno que ve una aplicación en un contenedor es idéntica a si se ejecutara directamente en el host.**
    - a. Verdadero
    - b. Falso
15. **Tu aplicación requiere una lista extensa de variables de entorno para su configuración. ¿Cuál es el método más simple para proporcionárselas al contenedor?**

---

### Respuestas

1. **Diferencia entre volúmenes y bind mounts**:  
   Los volúmenes son gestionados íntegramente por Docker y almacenados en una zona dedicada del host (`/var/lib/docker/volumes/` en Linux), ofreciendo mayor portabilidad y aislamiento. Los *bind mounts* vinculan un directorio o archivo arbitrario del host directamente dentro del contenedor, dependiendo fuertemente de la estructura del sistema de archivos local.

2. **Ventajas de los volúmenes**:  
   Son más portables, facilitan la gestión de copias de seguridad, permiten la integración con drivers de almacenamiento externos y no exponen rutas sensibles del sistema anfitrión.

3. **Garantía de persistencia**:  
   Docker desacopla el almacenamiento del ciclo de vida del contenedor. Los datos del volumen se escriben directamente en el host o almacenamiento externo fuera del *Union filesystem*, manteniéndose intactos tras eliminar el contenedor.

4. **Caso de uso de bind mounts**:  
   Entornos de desarrollo local para sincronizar código fuente o archivos de configuración en tiempo real sin tener que reconstruir la imagen tras cada cambio.

5. **Compartir volúmenes**:  
   Sí, especificando el mismo nombre de volumen en el flag `-v` o `--mount` al iniciar cada uno de los contenedores.

6. **Crear volumen con nombre**:  
   ```bash
   $ docker volume create my-products
   ```

7. **Montaje en solo lectura**:  
   ```bash
   $ docker container run -it --rm \
     -v my-products:/data:ro \
     alpine /bin/sh
   ```

8. **Localizar e interactuar con la ruta del volumen**:  
   - Obtener ruta:
     ```bash
     $ docker volume inspect my-products | grep Mountpoint
     ```
     Salida: `"Mountpoint": "/var/lib/docker/volumes/my-products/_data"`
   - En macOS/Windows (vía nsenter):
     ```bash
     $ docker container run -it --privileged --pid=host \
       debian nsenter -t 1 -m -u -n -i sh
     / # cd /var/lib/docker/volumes/my-products/_data
     / # echo "I love Docker" > sample.txt
     ```
   - Validar desde contenedor:
     ```bash
     $ docker container run --rm \
       --volume my-products:/data \
       alpine ls -l /data
     ```

9. **Montaje en lectura/escritura y creación de archivo**:  
   ```bash
   $ docker run -it --rm -v my-products:/data:ro \
     alpine /bin/sh
   / # cd /data
   /data # cat sample.txt
   ```
   En otra terminal:
   ```bash
   $ docker run -it --rm -v my-products:/app-data \
     alpine /bin/sh
   / # cd /app-data
   /app-data # echo "Hello other container" > hello.txt
   /app-data # exit
   ```

10. **Montaje de carpeta del host**:  
    ```bash
    $ docker container run -it --rm \
      -v $HOME/my-project:/app/data \
      alpine /bin/sh
    ```

11. **Eliminar volúmenes no utilizados**:  
    ```bash
    $ docker volume prune
    ```

12. **Inspeccionar volumen**:  
    ```bash
    $ docker volume inspect my_volume
    ```

13. **Implicaciones de seguridad de bind mounts**:  
    Permiten al contenedor acceder y potencialmente modificar archivos críticos del sistema operativo host. Deben usarse con permisos restringidos y preferentemente en modo solo lectura (`:ro`).

14. **La respuesta es Falso (b)**:  
    Cada contenedor es un entorno aislado (*sandbox*) con su propio conjunto independiente de variables de entorno.

15. **Configuración con lista extensa de variables**:  
    Agrupar todas las variables en un archivo de configuración (`.config` o `.env`) y proporcionarlo al comando de ejecución mediante `--env-file`:
    ```bash
    $ docker container run --rm -it \
      --env-file ./development.config \
      alpine sh -c "export"
    ```

