# Parte 2: Fundamentos de la Contenerización
## Capítulo 8: Aumento de la productividad con trucos y consejos de Docker

Este capítulo presenta diversos consejos, trucos y conceptos avanzados que resultan indispensables al contenerizar aplicaciones distribuidas complejas o al utilizar Docker para automatizar tareas sofisticadas. También aprenderás a aprovechar los contenedores para ejecutar tu entorno de desarrollo completo dentro de ellos.

---

### Temas tratados en este capítulo:
- Mantener limpio tu entorno de Docker
- Uso de archivos `.dockerignore`
- Ejecución de tareas administrativas simples dentro de un contenedor
- Limitación del uso de recursos de un contenedor (CPU, memoria, I/O y PIDs)
- Buenas prácticas de seguridad: evitar ejecutar contenedores como root
- Ejecución de comandos del CLI de Docker desde dentro de un contenedor (*Docker-outside-of-Docker*)
- Optimización del proceso de construcción de imágenes (*caching*)
- Escaneo de vulnerabilidades (CVEs) y detección de secretos
- Ejecución de un entorno de desarrollo completo dentro de un contenedor (*Dev Containers*)

---

### Objetivos de aprendizaje
Tras leer este capítulo, serás capaz de:
- Restaurar y limpiar eficazmente tu entorno de Docker eliminando recursos huérfanos.
- Utilizar archivos `.dockerignore` para acelerar las construcciones, reducir el tamaño de las imágenes y reforzar la seguridad.
- Ejecutar herramientas y utilidades (Perl, Python, etc.) directamente en contenedores sin instalarlas en tu máquina host.
- Limitar el consumo de recursos de CPU, memoria, I/O y procesos en tiempo de ejecución.
- Fortalecer la seguridad del sistema ejecutando contenedores con usuarios sin privilegios (`non-root`).
- Ejecutar comandos de Docker desde dentro de un contenedor mediante el montaje del socket de Docker.
- Acelerar y optimizar el proceso de construcción de imágenes aprovechando la caché de capas.
- Analizar imágenes en busca de vulnerabilidades conocidas (CVEs) y filtraciones de credenciales mediante Snyk y Docker Scout.
- Ejecutar un entorno de desarrollo integrado completo (con VS Code) dentro de un contenedor.

---

### Requisitos técnicos

Para seguir los ejemplos prácticos de este capítulo, necesitarás tener instalado Docker Desktop y Visual Studio Code (VS Code).

Prepara la carpeta del capítulo en tu repositorio local:

```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir chapter-08 && cd chapter-08
```

El código de ejemplo y soluciones se encuentra disponible en: [https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-08/solutions](https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-08/solutions).

---

### Mantener limpio tu entorno de Docker

#### Eliminación de imágenes colgantes (*Dangling Images*)
Una imagen colgante (*dangling image*) es una imagen no utilizada que ha perdido sus etiquetas (aparece como `<none>:<none>`), habitualmente porque se construyó una nueva versión utilizando el mismo nombre y tag:

```bash
$ docker image prune -f
```
*(El flag `-f` o `--force` evita la confirmación interactiva).*

#### Eliminación de contenedores detenidos
Para eliminar un contenedor específico:
```bash
$ docker container rm <container-id|container-name>
```

Para eliminar todos los contenedores detenidos de forma masiva:
```bash
$ docker container prune --force
```

#### Eliminación de volúmenes no utilizados
Los volúmenes almacenan datos persistentes. Su eliminación es irreversible, por lo que se recomienda ejecutar la limpieza interactiva:

```bash
$ docker volume prune
```
Salida interactiva:
```text
WARNING! This will remove all local volumes not used by at least one container.
Are you sure you want to continue? [y/N]
```

En entornos de producción, elimina volúmenes de forma individual y controlada:
```bash
$ docker volume rm <volume-name>
```

---

### Uso de un archivo `.dockerignore`

El archivo `.dockerignore` indica al motor de Docker qué archivos y directorios deben excluirse al enviar el contexto de construcción (*build context*) al daemon.

#### Beneficios clave:
- **Acelera la compilación**: Reduce drásticamente la cantidad de datos transferidos al daemon.
- **Reduce el tamaño de la imagen final**.
- **Mejora la seguridad**: Evita empaquetar accidentalmente archivos sensibles (archivos `.env`, claves privadas, certificados, etc.).
- **Evita invalidaciones innecesarias de caché**.

#### Ejemplo de `.dockerignore`:
```ignore
# Ignore everything
**
# Allow specific directories
!my-app/
!scripts/
# Ignore specific files within allowed directories
my-app/*.log
scripts/temp/
```

---

### Ejecución de tareas administrativas simples en un contenedor

#### Ejecución de un script Perl sin instalar Perl en el host:
1. Crear el directorio de trabajo:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-08
   $ mkdir simple-task && cd simple-task
   $ code .
   ```

2. Crear un archivo `sample.txt` con espacios al inicio de cada línea:
   ```text
      1234567890
      This is some text
        another line of text
      more text
    final line
   ```

3. Ejecutar el comando Perl dentro de un contenedor efímero:
   ```bash
   $ docker container run --rm -it \
     -v $(pwd):/usr/src/app \
     -w /usr/src/app \
     perl:slim sh -c "cat sample.txt | perl -lpe 's/^\s*//'"
   ```
   Salida (espacios iniciales eliminados):
   ```text
   1234567890
   This is some text
   another line of text
   more text
   final line
   ```

4. Para ejecutar scripts heredados incompatibles con la versión del host:
   ```bash
   $ docker container run -it --rm \
     -v $(pwd):/usr/src/app \
     -w /usr/src/app \
     perl:<old-version> perl your-old-perl-script.pl
   ```

#### Ejecución de un script en Python (`stats.py`):
1. Crear `stats.py`:
   ```python
   import sys

   def count_stats(filename):
       with open(filename, 'r') as file:
           text = file.read()
           lines = text.splitlines()
           words = text.split()
           letters = len(text)
           return len(lines), len(words), letters

   if __name__ == "__main__":
       if len(sys.argv) != 2:
           print("Usage: python stats.py <filename>")
           sys.exit(1)
       
       filename = sys.argv[1]
       lines, words, letters = count_stats(filename)
       print(f"Lines: {lines} Words: {words} Letters: {letters}")
   ```
   *Figura 8.1: Script en Python para calcular estadísticas de un texto*

2. Ejecutar mediante un contenedor Python 3 Alpine:
   ```bash
   $ docker container run --rm -it \
     -v $(pwd):/usr/src/app \
     -w /usr/src/app \
     python:3-alpine python stats.py sample.txt
   ```
   Salida:
   ```text
   Lines: 5 Words: 13 Letters: 121
   ```

---

### Limitación del uso de recursos de un contenedor

Docker utiliza los **cgroups (control groups)** del kernel de Linux para restringir y aislar el consumo de recursos de hardware.

#### Limitación de memoria RAM:
1. Iniciar un contenedor restringido a 512 MB:
   ```bash
   $ docker container run --rm -it \
     --name stress-test \
     --memory 512M \
     ubuntu:22.04 /bin/bash
   ```
2. Instalar la herramienta `stress`:
   ```bash
   /# apt-get update && apt-get install -y stress
   ```
3. En otra terminal, monitorizar el consumo con `docker stats` (*Figura 8.2*).
4. Generar presión de memoria con 3 trabajadores de 256 MB cada uno:
   ```bash
   /# stress -m 3
   ```
   El consumo de memoria se aproximará a 512 MB pero nunca superará dicho límite.

#### Limitación de CPU:
Restringir un contenedor a medio núcleo (0.5 CPUs):
```bash
$ docker run -it --cpus=0.5 --name cpu-limited-container ubuntu:latest
/# apt update && apt install -y stress
/# stress --cpu 1 --timeout 60
```
*(Al consultar `docker stats`, el uso de CPU se mantendrá estrangulado al 50%).*

#### Limitación de I/O de disco (*Block I/O* - Linux nativo):
Asignar pesos relativos de prioridad de E/S con `--blkio-weight` (de 10 a 1000):
```bash
$ docker run -it --blkio-weight=900 --name io-limited-container-high ubuntu:latest
$ docker run -it --blkio-weight=100 --name io-limited-container-low ubuntu:latest
/# apt update && apt install -y fio
```

#### Limitación del número de procesos (*PIDs limit*):
Evita ataques de denegación de servicio por bombas fork (*fork bombs*):
```bash
$ docker run -it --pids-limit=10 --name pids-limited-container ubuntu:latest
```
Al intentar crear procesos en bucle (`while true; do sleep 1000 & done`), el sistema bloquea nuevas bifurcaciones al alcanzar el límite:
```text
bash: fork: retry: Resource temporarily unavailable
```

---

### Evitar ejecutar contenedores como root

Por defecto, los procesos dentro de un contenedor se ejecutan con privilegios de `root` (UID 0). En caso de escape de contenedor, esto representa un riesgo de seguridad crítico.

#### Comparativa: root frente a non-root:

1. **Ejecución como root (por defecto)**:
   ```bash
   $ docker run -it --name root-container ubuntu:latest bash
   /# whoami
   root
   /# touch /root-owned-file
   /# id
   uid=0(root) gid=0(root) groups=0(root)
   /# exit
   $ docker rm root-container
   ```

2. **Ejecución como usuario sin privilegios (`nobody`)**:
   ```bash
   docker run -it --user nobody --name nonroot-container ubuntu:latest bash
   nobody@...:/$ whoami
   nobody
   nobody@...:/$ touch /root-owned-file
   touch: cannot touch '/root-owned-file': Permission denied
   nobody@...:/$ id
   uid=65534(nobody) gid=65534(nogroup)
   nobody@...:/$ exit
   $ docker rm nonroot-container
   ```

> **En producción**: Declara un usuario dedicado en tu Dockerfile mediante la instrucción `USER` (por ejemplo, `USER appuser`).

---

### Ejecución de comandos del CLI de Docker desde dentro de Docker

Para automatizar tareas de Docker desde un contenedor (por ejemplo, en agentes de Jenkins o GitLab CI), podemos vincular el socket de Docker del host (`/var/run/docker.sock`):

```bash
$ docker image pull docker:cli
$ docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker:cli sh
```

En contenedores Windows (PowerShell):
```powershell
PS> docker container run `
  --name docker-cli `
  -v \\.\pipe\docker_engine:\\.\pipe\docker_engine `
  docker:cli
```

> [!WARNING]
> **Docker-in-Docker (DinD)**: Ejecutar el daemon completo de Docker dentro de un contenedor requiere modo privilegiado y acarrea riesgos de seguridad, problemas de estabilidad del sistema de archivos y pérdida de rendimiento. Se desaconseja salvo para desarrollo interno del motor de Docker.

#### Ejemplo: Automatización de un pipeline de construcción y despliegue
1. Copiar el proyecto `library`:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-08
   $ cp -r ../chapter-07/library .
   $ code library
   ```

2. Crear `pipeline.sh`:
   ```bash
   #!/bin/sh
   set -e

   # Build the image
   docker build -t $REPOSITORY:$TAG .

   # Run the unit tests
   docker run --rm $REPOSITORY:$TAG gradle test

   # Login to Docker Hub
   echo "$HUB_PWD" | docker login -u "$HUB_USER" --password-stdin

   # Push the image
   docker push $REPOSITORY:$TAG
   ```
   *Figura 8.3: Script para compilar, probar y publicar la aplicación Java*

   Hacer ejecutable: `$ chmod +x ./pipeline.sh`

3. Crear `run-pipeline.sh`:
   ```bash
   #!/bin/sh
   docker run --rm \
     -v /var/run/docker.sock:/var/run/docker.sock \
     -v $(pwd):/workspace \
     -w /workspace \
     -e HUB_USER="<your-docker-hub-username>" \
     -e HUB_PWD="<your-docker-hub-password>" \
     -e REPOSITORY="<your-docker-hub-username>/library" \
     -e TAG="1.0" \
     docker:cli ./pipeline.sh
   ```
   *Figura 8.4: Comando para ejecutar el pipeline dentro del contenedor Docker CLI*

   Ejecución:
   ```bash
   $ chmod +x ./run-pipeline.sh
   $ ./run-pipeline.sh
   ```

---

### Optimización del proceso de construcción (*Caching de capas*)

#### Dockerfile no optimizado (invalida la caché en cada cambio de código):
```dockerfile
FROM node:23-alpine
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "index.js"]
```
*Figura 8.5: Dockerfile no optimizado para Node.js*

#### Dockerfile optimizado (aprovecha la caché de dependencias):
```dockerfile
FROM node:23-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["node", "index.js"]
```
*Figura 8.6: Dockerfile optimizado para Node.js*

Al separar la copia de `package.json` de la copia del código fuente, `npm install` solo se vuelve a ejecutar cuando cambian las dependencias, reduciendo los tiempos de construcción a fracciones de segundo.

---

### Escaneo de vulnerabilidades y secretos

#### Herramientas populares de escaneo:
- **Trivy, Clair, Anchore**: Escáneres de CVEs en capas de paquetes de sistema y dependencias de aplicación.
- **Docker Scout**: Herramienta de análisis integrada en Docker Desktop.
- **Snyk**: Plataforma de seguridad para dependencias y contenedores.
- **AquaSec / Sysdig**: Detección de credenciales y secretos filtrados.

#### Uso de Snyk:
```bash
$ npm install -g snyk
$ snyk auth
$ snyk test --docker <image-name>
$ snyk test --file=path/to/Dockerfile
$ snyk protect --docker <image-name>
```

#### Uso de Docker Scout:
1. Verificar disponibilidad:
   ```bash
   $ docker scout version
   ```
2. Analizar una imagen vulnerable:
   ```bash
   $ docker image pull gnschenker/whoami-python:1.0
   $ docker scout cves gnschenker/whoami-python:1.0
   ```
   *Figura 8.7: Escaneo de vulnerabilidades en gnschenker/whoami-python:1.0*

3. Actualizar la imagen base en el Dockerfile a una versión reciente (por ejemplo, `python:3.14-slim`), reconstruir y volver a escanear:
   ```bash
   $ docker image build -t <username>/whoami-python:1.1 .
   $ docker scout cves <username>/whoami-python:1.1
   ```
   *Figura 8.8: Escaneo de la imagen whoami reconstruida*

---

### Ejecución de un entorno de desarrollo en un contenedor (*Dev Containers*)

Mediante la extensión **Remote Development / Dev Containers** de VS Code, puedes ejecutar el servidor de desarrollo, compiladores, depuradores y SDKs íntegramente dentro de un contenedor Docker, manteniendo los archivos de código sincronizados con tu host.

#### Pasos de configuración:
1. Instalar la extensión **Remote Development** en VS Code (*Figura 8.9*).
2. Abrir la paleta de comandos y seleccionar **Remote-Containers: Open Folder in Container...** (*Figura 8.10*).
3. Seleccionar la carpeta `chapter-08/library` y elegir la opción **From 'Dockerfile'** (*Figura 8.11*).
4. VS Code levantará el contenedor de desarrollo y montará las extensiones en el servidor remoto del contenedor (*Figura 8.12*).
5. Abre una terminal dentro de VS Code (`Shift + Ctrl + '`): observarás que la sesión se ejecuta como root dentro del contenedor (`root@<id>:/workspaces/...`).
6. Añadir un nuevo controlador `DefaultController.java` en `src/main/java/com/example/library/controllers/`:
   ```java
   package com.example.library.controllers;

   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RestController;

   @RestController
   public class DefaultController {

       @GetMapping("/")
       public String index() {
           return "Library component";
       }
   }
   ```
   *Figura 8.13: Adición de un controlador por defecto dentro del Dev Container*

7. Iniciar la aplicación (`./mvnw spring-boot:run`) y comprobar en el navegador `http://localhost:8080`.
8. Al salir, VS Code generará la carpeta `.devcontainer/devcontainer.json` para reproducir este entorno en cualquier otro equipo.

---

### Resumen

En este capítulo aprendimos:
- Comandos para la poda y limpieza rigurosa de imágenes, contenedores y volúmenes.
- El uso de `.dockerignore` para optimizar el contexto de construcción y proteger secretos.
- La ejecución de scripts administrativos (Perl, Python) en entornos efímeros sin dependencias en el host.
- El control de consumo de recursos con límites de CPU, memoria, I/O y número de PIDs mediante cgroups.
- El endurecimiento (*hardening*) de imágenes evitando el uso del usuario `root`.
- La orquestación y automatización de pipelines CI/CD montando el socket `/var/run/docker.sock`.
- Técnicas de ordenación de capas para maximizar el uso de la caché de Docker.
- El escaneo preventivo de vulnerabilidades conocidas (CVEs) con Snyk y Docker Scout.
- La creación de entornos de desarrollo completos reproducibles con Dev Containers en VS Code.

---

### Preguntas

1. **¿Cuáles son las razones para ejecutar un entorno de desarrollo completo dentro de un contenedor?**
2. **¿Por qué se debe evitar ejecutar aplicaciones dentro de un contenedor como usuario `root`?**
3. **¿Por qué montarías el socket de Docker (`/var/run/docker.sock`) dentro de un contenedor?**
4. **Al podar recursos de Docker para liberar espacio en disco, ¿por qué los volúmenes requieren especial precaución?**
5. **¿Por qué querrías ejecutar ciertas tareas administrativas dentro de un contenedor en lugar de hacerlo directamente en el host?**

---

### Respuestas

1. **Motivos para usar Dev Containers**:  
   Permiten trabajar en equipos donde no se tienen permisos para instalar herramientas locales, aíslan dependencias incompatibles entre proyectos, garantizan que todos los miembros del equipo utilicen exactamente las mismas versiones de SDKs y facilitan la experimentación con nuevas tecnologías sin alterar el sistema operativo anfitrión.

2. **Peligros de ejecutar como `root`**:  
   La mayoría de las aplicaciones de negocio no requieren privilegios elevados. Ejecutar como root amplía la superficie de ataque; si un atacante vulnera la aplicación y logra escapar del contenedor, obtendría acceso como superusuario en el host. El principio de menor privilegio mitiga este riesgo ejecutando como usuario no privilegiado (e.g. `USER appuser` o `--user nobody`).

3. **Uso del socket de Docker**:  
   Permite que una aplicación dentro del contenedor (como un servidor de integración continua Jenkins o un script de pipeline) interactúe con el daemon de Docker del host para construir imágenes, levantar contenedores auxiliares o publicar artefactos en registros.

4. **Precaución con los volúmenes**:  
   Los volúmenes almacenan datos persistentes de estado que a menudo son críticos para el negocio y tienen un ciclo de vida superior al de los propios contenedores. Ejecutar `docker volume prune` elimina irreversiblemente los datos de todos los volúmenes no asociados a un contenedor en ejecución.

5. **Ventajas de tareas administrativas en contenedores**:  
   - **Aislamiento**: No interfieren con librerías o dependencias del host.
   - **Portabilidad y consistencia**: Se ejecutan idénticamente en cualquier sistema operativo que soporte Docker (macOS, Linux, Windows).
   - **Versionado y limpieza**: Permite ejecutar versiones antiguas o específicas de intérpretes (Perl, Python, Node.js) sin ensuciar el host, eliminándose por completo al finalizar con `--rm`.

