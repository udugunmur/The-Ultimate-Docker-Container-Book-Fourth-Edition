# Parte 2: Fundamentos de la Contenerización
## Capítulo 3: Dominando los Contenedores

En el capítulo anterior, aprendiste a preparar de manera óptima tu entorno de trabajo para el uso productivo y sin fricciones de Docker. En este capítulo, nos pondremos manos a la obra y aprenderemos todo lo importante que necesitas saber al trabajar con contenedores.

---

### Temas tratados en este capítulo:
- Ejecución del primer contenedor
- Iniciar, detener y eliminar contenedores
- Inspeccionar contenedores
- Ejecutar comandos dentro de un contenedor en ejecución
- Conectarse (*attach*) a un contenedor en ejecución
- Obtener los logs de un contenedor
- La anatomía de los contenedores

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Ejecutar, detener y eliminar un contenedor basado en una imagen existente, como Nginx, BusyBox o Alpine.
- Listar todos los contenedores en el sistema.
- Inspeccionar los metadatos de un contenedor en ejecución o detenido.
- Recuperar los registros (*logs*) producidos por una aplicación que se ejecuta dentro de un contenedor.
- Ejecutar un proceso como `/bin/sh` en un contenedor que ya está en ejecución.
- Conectar una terminal a un contenedor en ejecución.
- Explicar con tus propias palabras, a una persona interesada sin perfil técnico, los fundamentos internos de un contenedor.
- Explicar cómo los *namespaces* de Linux proporcionan aislamiento de procesos y cómo los *cgroups* gestionan la asignación de recursos, conformando la base de la contenerización.

---

### Requisitos técnicos

Para este capítulo, debes tener Docker Desktop instalado en tu estación de trabajo Linux, macOS o PC con Windows. En macOS, utiliza la aplicación Terminal, y en Windows, utiliza la consola PowerShell o Git Bash para probar los comandos que aprenderás.

---

### Ejecución del primer contenedor

Antes de comenzar, queremos asegurarnos de que Docker esté instalado correctamente en tu sistema y listo para aceptar tus órdenes. Abre una nueva ventana de terminal y escribe el siguiente comando (nota: no escribas el signo `$`, ya que representa el prompt de tu terminal):

```bash
$ docker version
```

Si todo funciona correctamente, deberías ver en la terminal la versión del cliente y del servidor de Docker instalados en tu equipo.

*Figura 3.1 – Salida del comando docker version*

Como puedes ver, en el MacBook Pro M2 del autor se encuentra instalada la versión 27.4.0. Si esto no funciona en tu equipo, algo en tu instalación no está bien. Por favor, asegúrate de haber seguido las instrucciones del capítulo anterior sobre cómo instalar Docker Desktop en tu sistema.

Ahora estás listo para entrar en acción. Escribe el siguiente comando en tu ventana de terminal y presiona la tecla Enter:

```bash
$ docker container run alpine echo "Hello World"
```

Al ejecutar el comando anterior por primera vez, deberías ver una salida en tu ventana de terminal similar a esta:

*Figura 3.2 – Ejecutando un contenedor Alpine por primera vez*

¡Ha sido muy sencillo! Intentemos ejecutar exactamente el mismo comando de nuevo:

```bash
$ docker container run alpine echo "Hello World"
```

La segunda, tercera o enésima vez que ejecutes el comando anterior, solo deberías ver esta salida en tu terminal:

```text
Hello World
```

Intenta razonar por qué la primera vez que ejecutas un comando ves una salida diferente a todas las veces posteriores. No te preocupes si no logras deducirlo; explicaremos las razones en detalle en las siguientes secciones de este capítulo.

---

### Iniciar, detener y eliminar contenedores

Has ejecutado con éxito un contenedor en la sección anterior. Ahora, queremos investigar en detalle qué ocurrió exactamente y por qué. Veamos nuevamente el comando que utilizamos:

```bash
$ docker container run alpine echo "Hello World"
```

Este comando contiene varias partes:
1. **`docker`**: El nombre de la herramienta de interfaz de línea de comandos (CLI) de Docker, que utilizamos para interactuar con Docker Engine.
2. **`container`**: Indica el contexto con el que estamos trabajando (como `container`, `image` o `volume`). Como queremos ejecutar un contenedor, nuestro contexto es `container`.
3. **`run`**: El comando específico que deseamos ejecutar en el contexto dado.
4. **`alpine`**: La imagen a partir de la cual se creará el contenedor. `alpine` es una imagen minimalista de Docker basada en Alpine Linux, con un índice de paquetes completo y un tamaño de solo unos 8 MB. Es una imagen oficial mantenida por el proyecto de código abierto de Alpine y Docker.
5. **`echo "Hello World"`**: El proceso o tarea que se ejecutará dentro del contenedor una vez iniciado.

*Figura 3.3 – Explicación del comando docker run*

Ahora que comprendemos las partes del comando, ejecutemos otro contenedor con un proceso diferente en su interior:

```bash
$ docker container run quay.io/centos/centos echo "Hello from centos"
```

Deberías ver una salida en tu terminal similar a la siguiente:

*Figura 3.4 – Ejecutando el comando echo dentro de un contenedor CentOS*

En este caso, la imagen utilizada es `quay.io/centos/centos` y el proceso ejecutado es `echo "Hello from centos"`.

Analicemos la salida en detalle:
1. `Unable to find image 'quay.io/centos/centos:latest' locally`: Docker no encontró la imagen en la caché local del sistema, por lo que procede a descargarla (*pull*) del registro configurado (en este caso, explícitamente desde `quay.io`).
2. `latest: Pulling from centos/centos`: Se descarga la capa correspondiente a la etiqueta `latest`.
3. `4ff8fa80ba5d: Pull complete Digest: sha256:... Status: Downloaded newer image for quay.io/centos/centos:latest`: Confirmación de la descarga y verificación de integridad.
4. `Hello from centos`: La salida generada por el proceso que ejecutamos dentro del contenedor.

Si volvemos a ejecutar el comando anterior, las líneas de descarga ya no aparecerán porque la imagen ya se encuentra en la caché local.

#### ¿Qué ocurre exactamente cuando ejecutas un contenedor?
Cuando ejecutas `docker container run`:
1. Docker comprueba si la imagen solicitada está disponible localmente; si no es así, la descarga del registro.
2. Crea un nuevo contenedor asignándole un sistema de archivos aislado y una interfaz de red.
3. Le asigna una dirección IP interna y configura los mapeos de puertos si se especificaron.
4. Inicia el contenedor ejecutando el comando o proceso indicado en un entorno aislado y consistente.

---

### Ejecución de un contenedor de preguntas de trivia aleatorias

Para las siguientes secciones, necesitamos un contenedor que se ejecute continuamente en segundo plano y genere contenido. Para ello, utilizaremos una API gratuita que devuelve preguntas de trivia aleatorias: [https://the-trivia-api.com](https://the-trivia-api.com/).

El objetivo es tener un proceso ejecutándose dentro de un contenedor que genere una nueva pregunta cada 2 segundos y la envíe a `STDOUT`. El siguiente script realiza exactamente esa tarea:

```bash
while :
do
  curl -s https://the-trivia-api.com/v2/questions\?limit\=1 | jq '.[0].question'
  sleep 2
done
```

En PowerShell, el comando equivalente es:

```powershell
while ($true) {
  Invoke-WebRequest -Uri "https://the-trivia-api.com/v2/questions\?limit\=1" -Method GET -UseBasicParsing | Select-Object -ExpandProperty Content | ConvertFrom-Json | Select-Object -ExpandProperty 0 | Select-Object -ExpandProperty question
  Start-Sleep -Seconds 2
}
```

*(Nota: `ConvertFrom-Json` requiere el módulo `Microsoft.PowerShell.Utility`).*

Para instalar `jq` (utilizado para formatear salidas JSON):
- En Windows (Chocolatey): `$ choco install jq`
- En macOS (Homebrew): `$ brew install jq`

Para simplificar, utilizaremos la imagen preconstruida `fundamentalsofdocker/trivia:ed4`, que contiene toda la lógica empaquetada. Ejecuta el contenedor en segundo plano (como demonio):

```bash
$ docker container run --detach \
  --name trivia fundamentalsofdocker/trivia:ed4
```

- **`--detach`** (o `-d`): Indica a Docker que ejecute el proceso del contenedor en segundo plano.
- **`--name`**: Asigna un nombre explícito al contenedor (`trivia`). Si no se especifica, Docker asigna automáticamente un nombre aleatorio compuesto por un adjetivo y el apellido de una figura científica célebre (por ejemplo, `boring_borg`).
- **`fundamentalsofdocker/trivia:ed4`**: Imagen utilizada, con la etiqueta `ed4` que denota la cuarta edición del libro.

Comprueba que el contenedor está en ejecución:

```bash
$ docker container ls -l
```

*Figura 3.6 – Detalles del último contenedor ejecutado*

La columna `STATUS` mostrará `Up 8 seconds`, indicando que lleva 8 segundos activo.

Para detener y eliminar el contenedor de trivia:

```bash
$ docker rm --force trivia
```

---

### Listar contenedores

A medida que trabajamos con contenedores, acumulamos varios en el sistema. Ejecutemos algunos contenedores de prueba:

```bash
$ docker container run alpine echo "hello world"
$ docker container run --detach \
  quay.io/centos/centos:stream9 sleep 3600
$ docker container run --detach --name trivia fundamentalsofdocker/trivia:ed4
```

Para ver qué contenedores están en ejecución:

```bash
$ docker container ls
```

*Figura 3.7 – Lista de todos los contenedores en ejecución en el sistema*

Por defecto, Docker muestra siete columnas:

| Columna | Descripción |
| :--- | :--- |
| **Container ID** | Versión corta del identificador único del contenedor (hash criptográfico SHA-256 de 64 caracteres en su versión completa). |
| **Image** | Nombre de la imagen a partir de la cual se instanció el contenedor. |
| **Command** | Comando utilizado para ejecutar el proceso principal en el contenedor. |
| **Created** | Fecha y hora de creación del contenedor. |
| **Status** | Estado actual del contenedor (`created`, `restarting`, `running`, `removing`, `paused`, `exited` o `dead`). |
| **Ports** | Puertos del contenedor mapeados hacia el host. |
| **Names** | Nombre asignado al contenedor (debe ser único en el host). |

*Tabla 3.1 – Descripción de las columnas del comando docker container ls*

#### Parámetros útiles de listado:
- **Listar todos los contenedores (activos y detenidos)**:
  ```bash
  $ docker container ls --all
  ```
- **Listar únicamente los identificadores (IDs)**:
  ```bash
  $ docker container ls --quiet
  ```
- **Eliminación masiva de todos los contenedores**:
  ```bash
  $ docker container rm --force $(docker container ls --all --quiet)
  ```
  *(Este comando elimina de forma forzada todos los contenedores definidos en el sistema, incluidos los detenidos).*
- **Ayuda del comando**:
  ```bash
  $ docker container ls --help
  ```

---

### Detener e iniciar contenedores

Detener e iniciar contenedores son operaciones esenciales para gestionar el ciclo de vida de nuestras aplicaciones.

1. Inicia el contenedor de trivia nuevamente:
   ```bash
   $ docker container run -d --name trivia \
     fundamentalsofdocker/trivia:ed4
   ```

2. Detén el contenedor:
   ```bash
   $ docker container stop trivia
   ```
   *(Nota: Observarás que el comando tarda unos 10 segundos en completarse. Docker envía una señal `SIGTERM` al proceso principal dentro del contenedor. Si el proceso no finaliza en 10 segundos, Docker envía `SIGKILL` para forzar su terminación).*

3. Obtener el ID del contenedor de forma automatizada:
   - **Bash**:
     ```bash
     $ export CONTAINER_ID=$(docker container ls -a | \
       grep trivia | awk '{print $1}')
     $ echo $CONTAINER_ID
     ```
   - **PowerShell**:
     ```powershell
     $ $CONTAINER_ID = docker container ls -a | `
       Select-String "trivia" | `
       Select-Object -ExpandProperty Line | `
       ForEach-Object { $_ -split ' ' } | `
       Select-Object -First 1
     $ Write-Output $CONTAINER_ID
     ```

4. Detener usando la variable de ID:
   ```bash
   $ docker container stop $CONTAINER_ID
   ```

5. Iniciar un contenedor detenido:
   ```bash
   $ docker container start $CONTAINER_ID
   ```
   o bien:
   ```bash
   $ docker container start trivia
   ```

---

### Eliminar contenedores

Los contenedores detenidos (`Exited`) continúan ocupando espacio en disco. Para eliminarlos:

```bash
$ docker container rm <container ID>
```
o por nombre:
```bash
$ docker container rm <container name>
```

Para forzar la eliminación de un contenedor aunque esté en ejecución:
```bash
$ docker container rm <container ID> --force
```

Elimina el contenedor de trivia antes de continuar:
```bash
$ docker container rm -f trivia
```

---

### Inspeccionar contenedores

El comando `docker container inspect` proporciona información detallada y de bajo nivel sobre un contenedor en formato JSON (configuración de red, puntos de montaje, variables de entorno, comandos de arranque, etc.).

1. Ejecuta el contenedor:
   ```bash
   $ docker container run --detach --name trivia \
     fundamentalsofdocker/trivia:ed4
   ```

2. Inspecciona los metadatos:
   ```bash
   $ docker container inspect trivia
   ```
   *Figura 3.8 – Inspeccionando el contenedor trivia*

3. **Filtrar la salida mediante plantillas Go y `jq`**:
   Para extraer únicamente el nodo del estado del contenedor:
   ```bash
   $ docker container inspect -f "{{json .State}}" trivia | jq .
   ```
   *Figura 3.9 – Nodo State de la salida de inspect*

---

### Ejecutar comandos dentro de un contenedor en ejecución

El comando `docker container exec` permite ejecutar un nuevo proceso dentro de un contenedor en ejecución sin interrumpir su proceso principal. Es ideal para tareas de diagnóstico, depuración e inspección.

1. **Abrir una shell interactiva dentro del contenedor**:
   ```bash
   $ docker container exec -i -t trivia /bin/sh
   ```
   - **`-i`** (`--interactive`): Mantiene `STDIN` abierto.
   - **`-t`** (`--tty`): Asigna una pseudo-terminal (TTY).
   - **`/bin/sh`**: El proceso a ejecutar.

   El prompt cambiará a `/app #`. Comprueba los procesos activos dentro del contenedor:
   ```bash
   /app # ps
   ```
   *Figura 3.10 – Ejecutando comandos dentro del contenedor trivia en ejecución*

   *(El proceso con PID 1 es el comando principal del contenedor).*  
   Para salir del contenedor, presiona `Ctrl + D`.

2. **Ejecutar comandos de forma no interactiva**:
   ```bash
   $ docker container exec trivia ps
   ```
   *Figura 3.11 – Lista de procesos dentro del contenedor trivia*

3. **Definir variables de entorno con `-e`**:
   ```bash
   $ docker container exec -it \
     -e MY_VAR="Hello World" \
     trivia /bin/sh
   ```
   Dentro del contenedor:
   ```bash
   /app # echo $MY_VAR
   ```
   *Figura 3.12 – Contenedor trivia con variable de entorno personalizada*  
   Sal del contenedor con `Ctrl + D` y elimínalo:
   ```bash
   $ docker container rm --force trivia
   ```

---

### Conectarse (*attach*) a un contenedor en ejecución

El comando `docker container attach` conecta la entrada, salida y error estándar (`STDIN`, `STDOUT`, `STDERR`) de tu terminal directamente al **proceso principal** del contenedor.

1. Ejecuta una nueva instancia de trivia en modo interactivo:
   ```bash
   $ docker container run -it \
     --name trivia fundamentalsofdocker/trivia:ed4
   ```
2. En otra ventana de terminal, conéctate al contenedor:
   ```bash
   $ docker container attach trivia
   ```
   - Para desconectarte sin detener el contenedor, usa la combinación de teclas **`Ctrl + P` seguido de `Ctrl + Q`**.
   - Presionar `Ctrl + C` enviará una señal de interrupción al proceso principal y detendrá el contenedor.

3. Limpieza:
   ```bash
   $ docker container rm --force trivia
   ```

#### Conectarse a un servidor web Nginx
1. Ejecuta Nginx en segundo plano:
   ```bash
   $ docker run -d --name nginx -p 8080:80 nginx:alpine
   ```
2. Verifica el acceso en tu terminal:
   ```bash
   $ curl -4 localhost:8080
   ```
   *Figura 3.13 – Mensaje de bienvenida de Nginx*

3. Conecta tu terminal a los logs en vivo de Nginx:
   ```bash
   $ docker container attach nginx
   ```
4. En otra ventana de terminal, genera tráfico:
   - **Bash**:
     ```bash
     $ for n in {1..10}; do curl -4 localhost:8080; done
     ```
   - **PowerShell**:
     ```powershell
     PS> for ($n = 1; $n -le 10; $n++) { curl -4 http://localhost:8080 }
     ```
   *Figura 3.14 – Registros de acceso de Nginx*

5. Detén y elimina el contenedor:
   ```bash
   $ docker container rm -f nginx
   ```

---

### Obtener los logs de un contenedor

Las aplicaciones contenerizadas deben emitir sus registros hacia `STDOUT` y `STDERR`. Docker captura estos flujos y los pone a disposición mediante el comando `docker container logs`.

1. Inicia el contenedor de trivia:
   ```bash
   $ docker container run --detach \
     --name trivia fundamentalsofdocker/trivia:ed4
   ```
2. Leer los logs completos:
   ```bash
   $ docker container logs trivia
   ```
3. Obtener solo las últimas líneas:
   ```bash
   $ docker container logs --tail 5 trivia
   ```
4. Seguir los logs en tiempo real (*follow*):
   ```bash
   $ docker container logs --tail 5 --follow trivia
   ```
   *(Presiona `Ctrl + C` para detener el seguimiento).*

5. Limpieza:
   ```bash
   $ docker container rm --force trivia
   ```

#### Mecanismos de registro (*Logging Drivers*)

Docker soporta múltiples controladores de registro configurables a nivel global o por contenedor:

| Driver | Descripción |
| :--- | :--- |
| **`none`** | No produce salida de registro para el contenedor. |
| **`json-file`** | Driver predeterminado. Almacena los logs en archivos JSON locales en el host. |
| **`journald`** | Redirige los logs al demonio `journald` del sistema operativo host. |
| **`syslog`** | Redirige los logs al demonio `syslog` del host. |
| **`gelf`** | Envía mensajes a endpoints compatibles con Graylog Extended Log Format (Graylog, Logstash). |
| **`fluentd`** | Envía mensajes al demonio `fluentd` instalado en el host. |
| **`awslogs`** | Envía los logs directamente a Amazon CloudWatch Logs. |
| **`splunk`** | Envía los logs a la plataforma Splunk. |

*Tabla 3.2 – Lista de logging drivers en Docker*

#### Uso de un driver de logging específico por contenedor:
```bash
$ docker container run --name test -it \
  --log-driver none \
  busybox sh -c \
  'for N in 1 2 3; do echo "Hello $N"; done'
```
Si intentas leer los logs con `docker container logs test`, obtendrás:
```text
Error response from daemon: configured logging driver does not support reading
```
Limpieza:
```bash
$ docker container rm test
```

#### Configuración avanzada: Cambiar el driver de logs por defecto en Linux (con Vagrant)
1. Instalar VirtualBox y Vagrant:
   - macOS: `$ brew install --cask virtualbox vagrant`
   - Windows: `$ choco install -y virtualbox vagrant`
2. Iniciar e ingresar a una máquina virtual Ubuntu 24.04:
   ```bash
   $ vagrant init bento/ubuntu-24.04
   $ vagrant up
   $ vagrant ssh
   ```
3. Instalar Docker Engine dentro de la VM:
   ```bash
   $ sudo apt-get update
   $ sudo apt-get install -y ca-certificates curl gnupg lsb-release
   $ sudo mkdir -p /etc/apt/keyrings
   $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   $ echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   $ sudo apt update
   $ sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   $ sudo usermod -aG docker $USER
   ```
4. Configurar la rotación de logs en `/etc/docker/daemon.json`:
   ```bash
   $ cd /etc/docker
   $ sudo vi daemon.json
   ```
   Contenido:
   ```json
   {
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "10m",
       "max-file": "3"
     }
   }
   ```
5. Recargar la configuración sin reiniciar el demonio:
   ```bash
   $ sudo kill -SIGHUP $(pidof dockerd)
   ```
6. Destruir el entorno de pruebas:
   ```bash
   $ vagrant destroy -f
   ```

---

### La anatomía de los contenedores

Los contenedores **no son máquinas virtuales ligeras**. Son procesos especialmente encapsulados y protegidos que se ejecutan en el sistema host compartiendo el mismo núcleo (kernel) de Linux.

- **Tiempo de inicio**: Los contenedores inician en **milisegundos**, frente a los segundos o minutos de una VM.
- **Ciclo de vida**: Los contenedores son **efímeros** por naturaleza.

*Figura 3.17 – Arquitectura de alto nivel de Docker*

#### Componentes estructurales clave:

1. **Namespaces de Linux**:
   - Abstraen recursos globales y crean la ilusión de aislamiento completo.
   - Tipos de namespaces:
     - **PID**: Aísla el árbol de procesos (el proceso principal dentro del contenedor ve PID 1, mientras que en el host tiene un PID regular como 334).
     - **NET**: Aísla interfaces de red y tablas de enrutamiento.
     - **MNT**: Aísla puntos de montaje del sistema de archivos.
     - **IPC**: Aísla recursos de comunicación entre procesos.
     - **UTS**: Aísla el nombre de host y nombre de dominio.
     - **USER**: Mapea usuarios y grupos dentro y fuera del contenedor.

2. **Control Groups (cgroups)**:
   - Limitan, contabilizan y aíslan el uso de recursos físicos (**CPU, memoria RAM, ancho de banda de red, I/O**).
   - Previenen el problema del "vecino ruidoso" (*noisy neighbor*).

3. **Sistema de archivos de unión (*Union Filesystem* / UnionFS)**:
   - Superpone múltiples capas de archivos de solo lectura y una capa superior de lectura/escritura para formar un único sistema de archivos coherente.

4. **Plomería de contenedores (*Container Plumbing*)**:
   - **`runc`**: Implementación de bajo nivel de la especificación OCI (*Open Container Initiative*) para crear y ejecutar contenedores.
   - **`containerd`**: Runtime de alto nivel que gestiona el ciclo de vida completo (descarga de imágenes, supervisión de procesos, red y almacenamiento).

---

### Resumen

En este capítulo aprendimos a:
- Ejecutar, detener, iniciar y eliminar contenedores basados en imágenes existentes.
- Inspeccionar metadatos detallados en formato JSON y filtrarlos con plantillas Go y `jq`.
- Ejecutar procesos interactivos o en segundo plano dentro de contenedores activos (`exec`) y acoplar terminales (`attach`).
- Gestionar registros y configurar distintos controladores de logs (*logging drivers*).
- Analizar los componentes esenciales del núcleo de Linux (*namespaces*, *cgroups*, *UnionFS*, *runc* y *containerd*) que hacen posible la contenerización.

---

### Lecturas adicionales

- **Primeros pasos con contenedores**: [https://docs.docker.com/get-started/](https://docs.docker.com/get-started/)
- **Referencia de comandos de contenedores en Docker**: [http://dockr.ly/2iLBV2I](http://dockr.ly/2iLBV2I)
- **Aislamiento con user namespaces**: [http://dockr.ly/2gmyKdf](http://dockr.ly/2gmyKdf)
- **Limitación de recursos en contenedores**: [http://dockr.ly/2wqN5Nn](http://dockr.ly/2wqN5Nn)

---

### Preguntas

1. **¿Qué dos características principales de Linux hacen posible la contenerización proporcionando aislamiento de procesos y gestión de recursos?**
2. **¿Cuáles son los estados posibles de un contenedor Docker?**
3. **¿Qué comando se utiliza para mostrar todos los contenedores en ejecución en tu host de Docker?**
4. **¿Cómo puedes listar únicamente los IDs de todos los contenedores Docker?**
5. **¿Cuál es la diferencia entre `docker container exec` y `docker container attach`?**
6. **¿Cómo ejecutas un contenedor en modo desacoplado (*detached*) y por qué elegirías hacerlo?**

---

### Respuestas

1. **Namespaces y Control Groups (cgroups)**:  
   Los *namespaces* proporcionan aislamiento al otorgar a cada contenedor su propia vista de recursos (procesos, red, sistemas de archivos), mientras que los *cgroups* limitan y controlan los recursos físicos (CPU, memoria, E/S) que pueden consumir los procesos.

2. **Estados posibles de un contenedor**:
   - **Created**: Creado pero aún no iniciado.
   - **Restarting**: En proceso de reinicio.
   - **Running**: Ejecutando activamente su proceso principal.
   - **Paused**: Procesos suspendidos temporalmente.
   - **Exited**: Ha finalizado su ejecución y su proceso principal se ha detenido.
   - **Dead**: Docker intentó detenerlo pero no pudo finalizarlo limpiamente.

3. **Listar contenedores en ejecución**:
   ```bash
   $ docker container ls
   ```
   *(o la variante corta tradicional `docker ps`). Para ver también los detenidos se añade `--all` o `-a`).*

4. **Listar únicamente IDs de todos los contenedores**:
   ```bash
   $ docker container ls -a -q
   ```

5. **Diferencia entre `exec` y `attach`**:
   - **`docker container exec`**: Inicia un nuevo proceso independiente dentro de un contenedor en ejecución (por ejemplo, una shell `/bin/sh` para depuración) sin interferir con el proceso principal.
   - **`docker container attach`**: Conecta los flujos estándar (`STDIN`, `STDOUT`, `STDERR`) de tu terminal directamente al proceso principal (PID 1) del contenedor.

6. **Ejecución en modo desacoplado (*detached*)**:
   Se utiliza el flag `--detach` o `-d`:
   ```bash
   docker container run -d --name my_container my_image
   ```
   Se utiliza para ejecutar contenedores como servicios en segundo plano (como servidores web o bases de datos) sin bloquear la sesión de la terminal.

