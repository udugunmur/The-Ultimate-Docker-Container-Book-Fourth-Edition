# Parte 2: Fundamentos de la Contenerización
## Capítulo 4: Creación y gestión de imágenes de contenedores

En el capítulo anterior, aprendimos qué son los contenedores y cómo ejecutarlos, detenerlos, eliminarlos, listarlos e inspeccionarlos. Extrajimos los registros de algunos contenedores, ejecutamos procesos dentro de contenedores activos y profundizamos en la anatomía de los contenedores. Cada vez que ejecutamos un contenedor, lo creamos a partir de una **imagen de contenedor**. En este capítulo, nos familiarizaremos a fondo con estas imágenes: aprenderemos qué son, cómo crearlas y cómo distribuirlas.

---

### Temas tratados en este capítulo:
- ¿Qué son las imágenes de Docker?
- Creación de imágenes de Docker
- *Lift and shift*: contenerización de una aplicación heredada (*legacy*)
- Compartir y distribuir imágenes
- Prácticas de seguridad en la cadena de suministro (*supply chain security*)

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Construir imágenes personalizadas de Docker usando Dockerfiles, aplicando las mejores prácticas de eficiencia y seguridad.
- Crear una imagen personalizada modificando interactivamente la capa de un contenedor y guardando los cambios con `commit`.
- Escribir un Dockerfile utilizando instrucciones clave como `FROM`, `COPY`, `RUN`, `CMD` y `ENTRYPOINT`.
- Exportar una imagen existente mediante `docker image save` e importarla en otro host de Docker con `docker image load`.
- Redactar un Dockerfile multi-etapa (*multi-stage build*) que minimice el tamaño de la imagen final incluyendo únicamente los binarios resultantes.
- Crear un Dockerfile para una aplicación heredada (*legacy*).
- Utilizar registros de Docker para almacenar, compartir y versionar imágenes realizando operaciones de `push` y `pull`.

---

### ¿Qué son las imágenes?

En Linux, todo es un archivo. Todo el sistema operativo es un sistema de archivos compuesto por carpetas y ficheros almacenados en el disco local. Una imagen de contenedor es esencialmente un gran archivo empaquetado (*tarball*) que contiene un sistema de archivos; más específicamente, contiene un **sistema de archivos por capas** (*layered filesystem*).

> **Tarball (.tar)**  
> Un *tarball* (archivo `.tar`) es un archivo único que agrupa múltiples ficheros o directorios. Es un formato habitual para distribuir software en sistemas tipo Unix (Linux, macOS) y se descomprime utilizando el comando `tar`. Suele comprimirse con `gzip` para reducir su tamaño.

#### El sistema de archivos por capas (*Layered Filesystem*)
Las imágenes de contenedor son las plantillas a partir de las cuales se crean los contenedores. No son un bloque monolítico, sino una **pila de capas**. La primera capa es la **capa base** (*base layer*).

*Figura 4.1: La imagen como una pila de capas*

Cada capa contiene ficheros y directorios que representan únicamente las **diferencias (deltas)** respecto a las capas inferiores. Docker utiliza un sistema de archivos de unión (*Union filesystem*) para fusionar este conjunto de capas en un único sistema de archivos virtual coherente. Un controlador de almacenamiento (*storage driver* o *graph driver*) gestiona la interacción entre capas.

**Las capas de una imagen son inmutables**: una vez generadas, no se pueden modificar jamás (solo eliminarse físicamente). Esta inmutabilidad permite almacenar las capas en caché sin riesgo de que queden desactualizadas y compartirlas de manera segura.

*Figura 4.2: Ejemplo de imagen personalizada basada en Alpine y Nginx*

En este ejemplo:
1. Capa base: Distribución Alpine Linux.
2. Segunda capa: Instalación de Nginx sobre Alpine.
3. Tercera capa: Archivos de la aplicación web (HTML, CSS, JavaScript).

Cada imagen parte de una imagen base, habitualmente una imagen oficial de Docker Hub (como Alpine, Ubuntu o CentOS), aunque también es posible crear imágenes desde cero utilizando `FROM scratch`.

> **Docker Hub**: Registro público central para compartir imágenes de contenedores disponible en [https://hub.docker.com/](https://hub.docker.com/).

#### La capa escribible del contenedor (*Writable Container Layer*)
Cuando Docker Engine crea un contenedor a partir de una imagen, añade una **capa escribible de lectura/escritura (r/w)** en la parte superior de la pila de capas inmutables (de solo lectura).

*Figura 4.3: La capa escribible del contenedor*  
*Figura 4.4: Múltiples contenedores compartiendo las mismas capas de imagen*

Gracias a la inmutabilidad, múltiples contenedores pueden compartir las mismas capas de imagen subyacentes. Cada contenedor solo necesita su propia capa fina de lectura/escritura, lo que reduce enormemente el consumo de disco y memoria RAM y permite arranques casi instantáneos.

#### Copia en escritura (*Copy-on-Write*)
Docker utiliza la estrategia de **Copy-on-Write (CoW)** para optimizar el acceso y modificación de archivos:
- Si un proceso dentro del contenedor lee un fichero de una capa inferior, lo lee directamente.
- Si desea modificar un fichero de una capa inferior de solo lectura, Docker **copia primero el fichero hacia la capa superior escribible** y luego aplica la modificación sobre esa copia.

*Figura 4.5: Imagen de Docker utilizando Copy-on-Write*

#### Controladores de almacenamiento (*Storage Drivers / Graph Drivers*)
Los controladores de almacenamiento fusionan las capas en un sistema de archivos raíz unificado dentro del *mount namespace* del contenedor. El controlador recomendado y estándar hoy en día es **`overlay2`** debido a su alto rendimiento y eficiencia.

---

### Creación de imágenes de Docker

Existen tres métodos para crear una imagen:
1. **Interactivo**: Modificar manualmente un contenedor en ejecución y confirmar los cambios con `commit`.
2. **Declarativo (el estándar recomendado)**: Definir las instrucciones en un archivo **`Dockerfile`** y construir la imagen con el Docker builder.
3. **Importación**: Cargar una imagen desde un archivo *tarball*.

---

### Creación interactiva de imágenes

Iniciamos un contenedor base interactivo (por ejemplo, Alpine 3.21):

```bash
$ docker container run -it \
  --name sample \
  alpine:3.21 /bin/sh
```

*Figura 4.6: Contenedor Alpine en modo interactivo*

Por defecto, Alpine no incluye `curl`. Lo instalamos dentro del contenedor:

```bash
/ # apk update && apk add curl
```

*Figura 4.7: Instalando curl en Alpine*

Probamos `curl` consultando los encabezados HTTP de Google:

*Figura 4.8: Uso de curl desde el contenedor*

Salimos del contenedor con `exit` o `Ctrl + D`. Al listar los contenedores, veremos el contenedor en estado `Exited`:

```bash
$ docker container ls -a | grep sample
```

*Figura 4.9: El contenedor Docker personalizado*

Para inspeccionar los cambios en el sistema de archivos respecto a la imagen base:

```bash
$ docker container diff sample
```

*Figura 4.10: Salida del comando docker container diff*

*(Donde `A` indica archivo/directorio añadido, `C` modificado y `D` eliminado).*

Guardamos las modificaciones en una nueva imagen llamada `my-alpine`:

```bash
$ docker container commit sample my-alpine
```

Salida generada:
```text
sha256:78a1a06bef41e0cbe9d2228d9715a1dbb87...
```

Verificamos la imagen creada:

```bash
$ docker image ls
```

*Figura 4.11: Listado de imágenes de Docker*

Para examinar el historial de capas de la imagen:

```bash
$ docker image history my-alpine
```

*Figura 4.12: Historial de la imagen Docker my-alpine*

La capa superior corresponde al paquete `curl` que agregamos, mientras que las inferiores corresponden a la imagen base de Alpine.

---

### Uso de Dockerfiles

El método interactivo es útil para prototipos, pero carece de repetibilidad y automatización. El enfoque estándar y profesional es el uso de un archivo **`Dockerfile`**: un manifiesto de texto declarativo con las instrucciones paso a paso para construir la imagen.

Ejemplo de Dockerfile para una aplicación Python 3.12:

```dockerfile
FROM python:3.12
RUN mkdir -p /app
WORKDIR /app
COPY ./requirements.txt /app/
RUN pip install -r requirements.txt
CMD ["python", "main.py"]
```

*Figura 4.13: Relación entre un Dockerfile y las capas de una imagen*

---

### Instrucciones clave de un Dockerfile

#### La instrucción `FROM`
Define la imagen base sobre la que se construirá la nueva imagen:
```dockerfile
FROM ubuntu:24.10
```
Para crear imágenes ultra-minimalistas que solo contengan un binario compilado estáticamente:
```dockerfile
FROM scratch
```
*(Nota: `FROM scratch` no genera ninguna capa en la imagen final).*

#### La instrucción `RUN`
Ejecuta comandos durante el proceso de construcción (*build time*) dentro de una nueva capa:
- En CentOS / RHEL:
  ```dockerfile
  RUN yum install -y wget
  ```
- En Debian / Ubuntu:
  ```dockerfile
  RUN apt-get update && apt-get install -y wget
  ```
- Concatenación de comandos para optimizar capas y limpiar cachés:
  ```dockerfile
  RUN apt-get update \
    && apt-get install -y --no-install-recommends \
      ca-certificates \
      libexpat1 \
      libffi6 \
      libgdbm3 \
      libreadline7 \
      libsqlite3-0 \
      libssl1.1 \
    && rm -rf /var/lib/apt/lists/*
  ```

#### Las instrucciones `COPY` y `ADD`
Copian archivos y directorios desde el host hacia el sistema de archivos de la imagen:
```dockerfile
COPY . /app
COPY ./web /app/web
COPY sample.txt /data/my-sample.txt
ADD sample.tar /app/bin/
ADD http://example.com/sample.txt /data/
```

- **`COPY`**: Copia archivos y directorios locales.
- **`ADD`**: Además de copiar, desempaqueta automáticamente archivos comprimidos (`.tar`) y permite descargar archivos desde URLs remotas.
- Soporta comodines (*wildcards*):
  ```dockerfile
  COPY ./sample* /mydir/
  ```
- Asignación de permisos y propietarios con `--chown`:
  ```dockerfile
  ADD --chown=11:22 ./data/web* /app/data/
  ```

#### La instrucción `WORKDIR`
Establece el directorio de trabajo para las instrucciones posteriores (`RUN`, `CMD`, `ENTRYPOINT`, `COPY`, `ADD`):
```dockerfile
WORKDIR /app/bin
```
*(Nota: `RUN cd /app/bin` no persiste el cambio de directorio entre capas; siempre debe utilizarse `WORKDIR`).*

#### Las instrucciones `CMD` y `ENTRYPOINT`
Definen el comando que se ejecutará cuando el contenedor se inicie (*run time*).

- **`ENTRYPOINT`**: Define el comando o ejecutable principal fijo.
- **`CMD`**: Proporciona los parámetros o argumentos por defecto para el `ENTRYPOINT`.

Ejemplo:
```dockerfile
FROM alpine:3.21
RUN apk update && apk add curl
ENTRYPOINT ["ping"]
CMD ["-c","3","8.8.8.8"]
```

- **Forma exec (recomendada)**: `ENTRYPOINT ["ejecutable", "param1"]` (formato JSON array).
- **Forma shell**: `CMD comando param1 param2` (ejecutado bajo `/bin/sh -c`).

Construcción y prueba de la imagen:
```bash
$ docker image build -t pinger .
```
*Figura 4.14: Construcción de la imagen Docker pinger*

Ejecución por defecto:
```bash
$ docker container run --rm -it pinger
```
*Figura 4.15: Salida del contenedor pinger*

Sobrescribir los argumentos de `CMD` al iniciar el contenedor:
```bash
$ docker container run --rm -it pinger -w 5 127.0.0.1
```

Sobrescribir el `ENTRYPOINT` completo:
```bash
$ docker container run --rm -it --entrypoint ash pinger
```

---

### Ejemplo de un Dockerfile completo (Node.js)

```dockerfile
FROM node:23-bookworm
RUN mkdir -p /app
WORKDIR /app
COPY package.json /app/
RUN npm install
COPY . /app
ENTRYPOINT ["npm"]
CMD ["start"]
```

---

### Construcción práctica de una imagen

1. Crear la estructura de directorios:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
   $ mkdir chapter-04 && cd chapter-04
   $ mkdir sample-1 && cd sample-1
   ```

2. Crear un archivo `Dockerfile`:
   ```dockerfile
   FROM ubuntu:25.04
   RUN apt-get update && apt-get install -y wget
   ```

3. Construir la imagen:
   ```bash
   $ docker image build -t my-ubuntu .
   ```
   *Figura 4.16: Construyendo nuestra primera imagen personalizada desde Ubuntu 25.04*

---

### Construcciones multi-etapa (*Multi-stage Builds*)

Las construcciones multi-etapa permiten compilar el código fuente en una etapa intermedia con todas las herramientas necesarias (SDKs, compiladores) y copiar únicamente los artefactos binarios resultantes en una imagen final mínima (`scratch` o `alpine`), reduciendo drásticamente el tamaño y la superficie de ataque.

#### Ejemplo con una aplicación en C (`hello.c`):
```c
#include <stdio.h>
int main (void) {
    printf ("Hello, world!\n");
    return 0;
}
```

#### Dockerfile sin optimizar (260 MB):
```dockerfile
FROM alpine:3.12
RUN apk update && \
    apk add --update alpine-sdk
RUN mkdir /app
WORKDIR /app
COPY . /app
RUN mkdir bin
RUN gcc -Wall hello.c -o bin/hello
ENTRYPOINT ["/app/bin/hello"]
```

```bash
$ docker image build -t hello-world .
$ docker image ls | grep hello-world
```
*Figura 4.17: Construcción de la imagen Docker para la aplicación C*  
*Figura 4.18: Tamaño de la imagen Docker sin optimizar (260 MB)*

#### Dockerfile multi-etapa optimizado (`Dockerfile.multi-step` - 136 KB):
```dockerfile
FROM alpine:3.21 AS build
RUN apk update && \
    apk add --update alpine-sdk
RUN mkdir /app
WORKDIR /app
COPY . /app
RUN mkdir bin
# Compile statically so runtime has no dependencies
RUN gcc -static -O2 hello.c -o bin/hello

FROM scratch
COPY --from=build /app/bin/hello /app/hello
ENTRYPOINT ["/app/hello"]
```

Construcción y comparación de tamaños:
```bash
$ docker image build -t hello-world-small -f Dockerfile.multi-step .
$ docker image ls | grep hello-world
```
*Figura 4.19: Comparación de tamaños de las imágenes de Docker*

El tamaño se redujo de **260 MB a 136 KB** (una reducción de más de 3 órdenes de magnitud).

---

### Buenas prácticas en Dockerfiles

1. **Los contenedores deben ser efímeros**: Minimizar los tiempos de inicialización y parada.
2. **Aprovechar la caché de capas**: Colocar las instrucciones que cambian con menor frecuencia (como la instalación de dependencias con `package.json` o `requirements.txt`) antes de las instrucciones que copian el código fuente que cambia constantemente.
3. **Minimizar el número de capas**: Concatenar comandos `RUN` relacionados con `&&` y limpiar archivos temporales en la misma capa.
4. **Utilizar `.dockerignore`**: Excluir ficheros innecesarios (`.git`, `node_modules`, archivos de logs, etc.) del contexto de construcción.
5. **Instalar solo los paquetes indispensables**.
6. **Usar multi-stage builds**.

---

### Guardar y cargar imágenes (*Save and Load*)

Las imágenes pueden exportarse e importarse como archivos comprimidos `.tar`:

- **Exportar una imagen a un tarball**:
  ```bash
  $ mkdir backup
  $ docker image save -o ./backup/my-alpine.tar my-alpine
  ```
  *Figura 4.20: Exportación de una imagen como tarball*

- **Importar una imagen desde un tarball**:
  ```bash
  $ docker image load -i ./backup/my-alpine.tar
  ```
  Salida:
  ```text
  Loaded image: my-alpine:latest
  ```

---

### Contenerización de una aplicación heredada (*Lift and Shift*)

El enfoque *Lift and Shift* permite modernizar aplicaciones empresariales tradicionales (Java, .NET, C/C++, Python, etc.) empaquetándolas en contenedores sin necesidad de reescribir su lógica interna.

#### Pasos para la contenerización:
1. **Inventario y análisis de dependencias externas**: Identificar bases de datos, APIs de terceros, sistemas de mensajería (ESB), colas, etc.
2. **Preparación del código fuente y comandos de compilación**: Establecer el directorio raíz del proyecto y comandos de construcción tradicionales (`mvn clean install`, `msbuild`, `make`).
3. **Gestión de configuración**: Separar configuraciones de tiempo de compilación (*build-time*), entorno (*environment*) y tiempo de ejecución (*runtime*).
4. **Gestión de secretos**: No almacenar contraseñas ni cadenas de conexión en texto claro; obtenerlas en tiempo de ejecución desde almacenes seguros (como HashiCorp Vault, AWS Secrets Manager o Azure Key Vault).
5. **Redacción del Dockerfile**:
   - Seleccionar la imagen base adecuada.
   - Copiar el código fuente mediante `COPY` y configurar `.dockerignore`.
   - Compilar la aplicación dentro de la imagen (`RUN mvn --clean install`).
   - Declarar variables de entorno (`ENV`) y puertos expuestos (`EXPOSE`).
   - Definir el punto de entrada con `ENTRYPOINT` o mediante un script `docker-entrypoint.sh` con permisos de ejecución (`chmod +x`).

#### Retorno de inversión (*ROI*):
- Reducción de más del **50% en costos de mantenimiento**.
- Reducción de hasta un **90% en el tiempo entre despliegues de nuevas versiones**.

---

### Compartir y distribuir imágenes

#### Estructura del nombre de una imagen (*Namespaces*):
El nombre totalmente cualificado (*Fully Qualified Name*) de una imagen sigue la siguiente estructura:

```text
<registry URL>/<User or Org>/<name>:<tag>
```

| Componente | Descripción |
| :--- | :--- |
| **`<registry URL>`** | URL del registro de contenedores (por defecto `docker.io` para Docker Hub; otros incluyen Google GCR/Artifact Registry, AWS ECR, Azure ACR, Red Hat Quay, JFrog Artifactory). |
| **`<User or Org>`** | Identificador de usuario u organización en el registro. |
| **`<name>`** | Nombre del repositorio o imagen. |
| **`<tag>`** | Etiqueta o versión de la imagen (por defecto `latest`). |

*Tabla 4.1: Elementos del espacio de nombres de una imagen Docker*

Ejemplos:

| Nombre de imagen | Descripción |
| :--- | :--- |
| `alpine` | Imagen oficial en Docker Hub con la etiqueta `latest`. |
| `ubuntu:22.04` | Imagen oficial de Ubuntu con la versión 22.04. |
| `hashicorp/vault` | Imagen de Vault de la organización HashiCorp en Docker Hub. |
| `acme/web-api:12.0` | Versión 12.0 de la API web de la organización Acme. |
| `gcr.io/jdoe/sample-app:1.1` | Imagen en Google Container Registry del usuario jdoe. |

*Tabla 4.2: Ejemplos de nombres de imágenes Docker válidos*

#### Publicación (*Push*) de imágenes a Docker Hub
1. Etiquetar la imagen local con el prefijo de tu usuario de Docker Hub:
   ```bash
   $ docker image tag alpine:latest gnschenker/alpine:1.0
   ```
2. Iniciar sesión en Docker Hub:
   ```bash
   $ docker login -u gnschenker -p <tu_contraseña>
   ```
3. Subir la imagen al registro:
   ```bash
   $ docker image push gnschenker/alpine:1.0
   ```

---

### Prácticas de seguridad en la cadena de suministro (*Supply Chain Security*)

1. **Uso de imágenes oficiales y verificadas**: Utilizar imágenes base curadas y mantenidas por fuentes de confianza.
2. **Escaneo continuo de vulnerabilidades**: Integrar herramientas de escaneo como **Trivy**, **Clair** o **Docker Scout** en el pipeline.
3. **Firma y verificación de imágenes**: Emplear firmas criptográficas (Docker Content Trust / Notary / Cosign) para validar la autenticidad e integridad.
4. **Principio de menor privilegio**: Ejecutar contenedores con usuarios sin privilegios (no utilizar `root`).
5. **Actualizaciones y parches periódicos**: Actualizar regularmente las imágenes base para incorporar parches de seguridad.

---

### Resumen

En este capítulo analizamos:
- La arquitectura por capas de las imágenes de Docker, la inmutabilidad de sus capas y el funcionamiento de los *storage drivers* como `overlay2`.
- Los métodos de creación interactiva (`commit`), declarativa (`Dockerfile`) y mediante archivos de archivo (`save`/`load`).
- La optimización drástica del tamaño de las imágenes utilizando construcciones multi-etapa (*multi-stage builds*).
- La metodología *Lift and Shift* para modernizar aplicaciones empresariales heredadas.
- La nomenclatura de imágenes, gestión de etiquetas y publicación en registros públicos y privados.
- Los principios fundamentales para garantizar la seguridad en la cadena de suministro de imágenes de contenedores.

---

### Preguntas

1. **¿Cuál es la función principal de los *graph drivers* (controladores de almacenamiento) de Docker?**
2. **¿Para qué se utiliza un Dockerfile?**
3. **¿Cómo puedes crear una imagen de Docker de forma interactiva?**
4. **¿Cuáles son dos beneficios importantes de utilizar construcciones multi-etapa (*multi-stage builds*) en los Dockerfiles?**
5. **¿Cuáles son tres buenas prácticas para proteger la cadena de suministro de imágenes de Docker?**
6. **¿Cómo crearías un Dockerfile basado en Ubuntu 25.04 que instale `ping` y lo ejecute al iniciar el contenedor haciendo ping a `127.0.0.1` por defecto?**
7. **¿Cómo crearías una imagen basada en `alpine:latest` con `curl` instalado y etiquetada como `my-alpine:1.0`?**
8. **Crea un Dockerfile multi-etapa para generar una imagen de tamaño mínimo para una aplicación Hello World en C o Go.**
9. **Nombra tres características esenciales de una imagen de contenedor Docker.**
10. **¿Qué comando se utiliza para exportar una imagen Docker como un archivo tarball?**
11. **Deseas subir una imagen llamada `foo:1.0` a tu cuenta personal `jdoe` en Docker Hub. ¿Cuál es el procedimiento correcto?**
    - a.
      ```bash
      $ docker container push foo:1.0
      ```
    - b.
      ```bash
      $ docker image tag foo:1.0 jdoe/foo:1.0
      $ docker image push jdoe/foo:1.0
      ```
    - c.
      ```bash
      $ docker login -u jdoe -p <your password>
      $ docker image tag foo:1.0 jdoe/foo:1.0
      $ docker image push jdoe/foo:1.0
      ```
    - d.
      ```bash
      $ docker login -u jdoe -p <your password>
      $ docker container tag foo:1.0 jdoe/foo:1.0
      $ docker container push jdoe/foo:1.0
      ```
    - e.
      ```bash
      $ docker login -u jdoe -p <your password>
      $ docker image push foo:1.0 jdoe/foo:1.0
      ```

---

### Respuestas

1. **Función de los storage drivers**:  
   Fusionan múltiples capas inmutables de solo lectura en un único sistema de archivos raíz coherente para el contenedor.

2. **Propósito del Dockerfile**:  
   Es un archivo de manifiesto de texto utilizado para definir y construir imágenes de contenedores de forma declarativa, automatizada y reproducible.

3. **Creación interactiva**:  
   Iniciando un contenedor interactivo desde una imagen base, realizando las modificaciones deseadas en su sistema de archivos y guardando el estado resultante como una nueva imagen mediante `docker container commit`.

4. **Beneficios de multi-stage builds**:  
   - Reducción drástica del tamaño de la imagen final al excluir compiladores, herramientas de desarrollo y SDKs.
   - Reducción de la superficie de ataque y mejora de la seguridad en entornos productivos.

5. **Buenas prácticas de seguridad en la cadena de suministro**:
   - Escaneo periódico de vulnerabilidades (CVEs).
   - Uso exclusivo de imágenes oficiales o verificadas de fuentes confiables.
   - Firma y verificación criptográfica de imágenes (Docker Content Trust / Cosign).

6. **Dockerfile con Ubuntu 25.04 y ping**:
   ```dockerfile
   FROM ubuntu:25.04
   RUN apt-get update && \
       apt-get install -y iputils-ping
   ENTRYPOINT ["ping"]
   CMD ["127.0.0.1"]
   ```
   Construcción:
   ```bash
   $ docker image build -t mypinger .
   ```

7. **Imagen Alpine con curl (`my-alpine:1.0`)**:
   ```dockerfile
   FROM alpine:latest
   RUN apk update && \
       apk add curl
   ```
   Construcción:
   ```bash
   $ docker image build -t my-alpine:1.0 .
   ```

8. **Dockerfile multi-etapa mínimo en Go**:
   ```dockerfile
   FROM golang:1.23 AS builder
   WORKDIR /app
   # Disable modules so no go.mod is needed
   ENV GO111MODULE=off
   COPY main.go .
   RUN CGO_ENABLED=0 GOOS=linux \
       go build -ldflags="-s -w" -o hello

   FROM scratch
   COPY --from=builder /app/hello /hello
   ENTRYPOINT ["/hello"]
   ```

9. **Características esenciales de una imagen Docker**:
   - Es inmutable.
   - Está compuesta por una o múltiples capas apiladas.
   - Contiene todos los archivos, librerías y dependencias necesarias para ejecutar la aplicación empaquetada.

10. **Exportación a tarball**:  
    ```bash
    $ docker image save -o image.tar <image-name>
    ```

11. **La respuesta correcta es C**:  
    Primero se inicia sesión en el registro (`docker login`), luego se etiqueta la imagen con el espacio de nombres de usuario (`docker image tag foo:1.0 jdoe/foo:1.0`) y finalmente se sube al registro (`docker image push jdoe/foo:1.0`).
