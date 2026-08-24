# Parte 2: Fundamentos de la Contenerización
## Capítulo 6: Depuración de código en ejecución dentro de contenedores

En el capítulo anterior, aprendimos a trabajar con contenedores con estado que consumen y producen datos, y a configurar nuestros contenedores en tiempo de ejecución y en tiempo de construcción de imágenes mediante variables de entorno y archivos de configuración.

En este capítulo, presentaremos las técnicas utilizadas habitualmente para permitir que un desarrollador evolucione, modifique, depure y pruebe su código mientras se ejecuta dentro de un contenedor. Con estas técnicas, disfrutarás de un flujo de desarrollo ágil y sin fricciones, similar al desarrollo nativo en local.

---

### Temas tratados en este capítulo:
- Evolución y pruebas de código en ejecución dentro de un contenedor
- Reinicio automático del código tras cambios (*auto-restarting*)
- Depuración de código línea por línea dentro de un contenedor
- Instrumentación de tu código para generar información de logging significativa
- Uso de OpenTelemetry y Jaeger para monitorización y resolución de problemas

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Montar el código fuente del host dentro de un contenedor en ejecución.
- Configurar una aplicación en un contenedor para que se reinicie automáticamente tras modificar el código.
- Configurar Visual Studio Code (VS Code) para depurar aplicaciones en Java, Node.js, Python o .NET línea por línea dentro de un contenedor.
- Registrar eventos clave de la aplicación mediante niveles de logging estructurados.
- Configurar una aplicación distribuida para trazabilidad (*distributed tracing*) utilizando el estándar OpenTracing / OpenTelemetry y Jaeger.

---

### Requisitos técnicos

Para seguir los ejemplos prácticos de este capítulo, necesitarás Docker Desktop (en macOS, Windows o Linux) y un editor de código (preferiblemente VS Code).

Prepara tu entorno creando la carpeta correspondiente:

```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir chapter-06 && cd chapter-06
```

Las soluciones completas de los ejemplos están disponibles en `chapter-06/solutions` o en GitHub: [https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-06/solutions](https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-06/solutions).

---

### Evolución y pruebas de código en ejecución dentro de un contenedor

Asegúrate de tener Node.js y npm instalados:
- En macOS: `$ brew install node`
- En Windows: `$ choco install -y nodejs`

#### Creación de la aplicación Node.js de ejemplo:
1. Crear el directorio del proyecto e inicializarlo con npm:
   ```bash
   $ mkdir node-sample && cd node-sample
   $ npm init --yes
   ```
   *Figura 6.1 – Contenido del archivo package.json de la aplicación Node.js de ejemplo*

2. Instalar Express.js:
   ```bash
   $ npm install express --save
   ```
   Esto añadirá la dependencia a `package.json`:
   ```json
   "dependencies": {
     "express": "^5.1.0"
   }
   ```

3. Abrir el proyecto en VS Code:
   ```bash
   $ code .
   ```

4. Crear el archivo `index.js`:
   ```javascript
   const express = require('express');
   const app = express();
   const port = 3000;

   app.get('/', (req, res) => {
     res.send('Hello World!');
   });

   app.listen(port, '0.0.0.0', () => {
     console.log(`Example app listening at 0.0.0.0:${port}`);
   });
   ```
   *Figura 6.2 – Contenido del archivo index.js*

5. Ejecutar la aplicación en local:
   ```bash
   $ node index.js
   ```
   Salida:
   ```text
   Example app listening at 0.0.0.0:3000
   ```
   *(La dirección `0.0.0.0` indica que el servidor escuchará en todas las interfaces de red IPv4 disponibles).*

6. Accede a `http://localhost:3000` en tu navegador para ver el mensaje (*Figura 6.3*). Detén la aplicación con `Ctrl + C`.

#### Contenerización de la aplicación Node.js:
1. Crear un archivo `Dockerfile`:
   ```dockerfile
   FROM node:23-alpine
   WORKDIR /app
   COPY package*.json ./
   RUN npm install
   COPY . .
   EXPOSE 3000
   CMD ["node", "index.js"]
   ```
   *Figura 6.4 – Dockerfile para la aplicación Node.js de ejemplo*

2. Construir la imagen:
   ```bash
   $ docker image build -t sample-app .
   ```

3. Ejecutar el contenedor:
   ```bash
   $ docker container run --rm -it --init \
     --name my-sample-app \
     -p 3000:3000 \
     sample-app
   ```
   *(El flag `--init` asegura que la señal `Ctrl + C` se transmita correctamente al proceso dentro del contenedor).*

4. Añadir un nuevo endpoint `/hobbies` en `index.js`:
   ```javascript
   const hobbies = [
     'Swimming',
     'Diving',
     'Jogging',
     'Cooking',
     'Singing'
   ];

   app.get('/hobbies', (req,res)=>{
     res.send(hobbies);
   })
   ```

5. Probar el nuevo endpoint en local con `node index.js` y acceder a `http://localhost:3000/hobbies`.

6. Para probar en el contenedor sin optimizaciones, tendríamos que reconstruir la imagen (`docker image build -t sample-app .`) y volver a ejecutar el contenedor. Este ciclo repetitivo añade demasiada fricción al desarrollo.

---

### Montaje de código en evolución en el contenedor en ejecución

Para evitar reconstruir la imagen en cada cambio, montamos la carpeta local del host dentro del contenedor utilizando la opción `--volume`:

```bash
$ docker container run --rm -it --init \
  --volume $(pwd):/app \
  -p 3000:3000 \
  sample-app
```

#### Probando cambios inmediatos:
1. Añadir el endpoint `/status` a `index.js`:
   ```javascript
   app.get('/status', (req,res)=>{
     res.send('OK');
   })
   ```
2. Ejecutar el contenedor con el volumen montado (sin reconstruir la imagen):
   ```bash
   $ docker container run --rm -it --init \
     --name my-sample-app \
     --volume $(pwd):/app \
     -p 3000:3000 \
     sample-app
   ```
3. Probar el endpoint con `curl`:
   ```bash
   $ curl localhost:3000/status
   OK
   ```
   En PowerShell:
   ```powershell
   PS> (iwr -Uri http://localhost:3000/status).Content
   ```
4. Si modificamos el texto a `OK, all good` y guardamos, el cambio se propaga al sistema de archivos del contenedor (verificable con `docker container exec my-sample-app cat index.js`), pero Node.js requiere reiniciarse para cargarlo.
5. Al reiniciar el contenedor y consultar de nuevo:
   ```bash
   $ curl http://localhost:3000/status
   OK, all good
   ```

---

### Reinicio automático del código tras cambios (*Auto-restarting*)

#### Auto-reinicio en Node.js con `nodemon`:
1. Instalar `nodemon` globalmente:
   ```bash
   $ npm install -g nodemon
   ```
2. Probar en local con `nodemon`:
   *Figura 6.5 – Ejecución de la aplicación Node.js con nodemon*
3. Añadir el endpoint `/colors`:
   ```javascript
   app.get('/colors', (req,res)=>{
     res.send(['red','green','blue']);
   })
   ```
   `nodemon` detecta el cambio y reinicia la aplicación automáticamente:
   ```text
   [nodemon] restarting due to changes...
   [nodemon] starting `node index.js`
   Application listening at 0.0.0.0:3000
   ```
4. Crear un `Dockerfile.dev` para desarrollo con `nodemon`:
   ```dockerfile
   FROM node:23-alpine
   RUN npm install -g nodemon
   WORKDIR /app
   COPY package*.json ./
   RUN npm install
   COPY . .
   EXPOSE 3000
   CMD ["nodemon", "index.js"]
   ```
   *Figura 6.6 – Dockerfile utilizado para desarrollo en Node.js*

5. Construir y ejecutar la imagen de desarrollo:
   ```bash
   $ docker image build \
     -f Dockerfile.dev \
     -t node-demo-dev .
   $ docker container run --rm -it --init \
     -v $(pwd):/app \
     -p 3000:3000 \
     node-demo-dev
   ```
   Cualquier cambio guardado en el host reiniciará automáticamente el servidor dentro del contenedor.
6. Limpieza:
   ```bash
   $ docker container rm -f $(docker container ls -aq)
   ```

---

### Auto-reinicio para Java y Spring Boot

#### Instalación de JDK 21:
- **macOS (Homebrew)**:
  ```bash
  brew install openjdk@21
  echo 'export PATH="/opt/homebrew/opt/openjdk@21/bin:$PATH"' >> ~/.zprofile
  source ~/.zprofile
  brew link --force --overwrite openjdk@21
  java -version
  ```
- **Windows (Chocolatey)**:
  ```powershell
  choco install openjdk21
  java -version
  ```

#### Configuración del proyecto Spring Boot:
1. Generar el proyecto en [https://start.spring.io](https://start.spring.io/): Maven, Java 21, Spring Boot 3.5+, dependencia `Spring Web` (*Figura 6.7*).
2. Descomprimir en `chapter-06/java-springboot-demo`.
3. Generar el Maven Wrapper y abrir en VS Code:
   ```bash
   $ cd chapter-06/java-springboot-demo
   $ mvn wrapper:wrapper
   $ code .
   ```
4. Configurar en VS Code `files.autoSave`: `afterDelay` (1000 ms).
5. Ejecutar la aplicación desde `DemoApplication.java` (*Figura 6.8* y *Figura 6.9*).
6. Añadir el endpoint `/species`:
   ```java
   @GetMapping("/species")
   public List<String> getSpecies() {
       return List.of("Mouse", "Rat", "Cat", "Dog");
   }
   ```
   *Figura 6.10 – Código completo del ejemplo Spring Boot*  
   *Figura 6.11 – Uso de Thunder Client para probar la aplicación*

7. Añadir `spring-boot-devtools` en `pom.xml` para auto-recarga:
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-devtools</artifactId>
       <optional>true</optional>
   </dependency>
   ```
   *Figura 6.12 – Añadiendo referencia a Spring Boot DevTools*

8. Crear el `Dockerfile` del proyecto:
   ```dockerfile
   FROM maven:3.9-eclipse-temurin-21-alpine
   WORKDIR /app
   COPY pom.xml .
   RUN mvn dependency:go-offline
   COPY src ./src
   CMD ["mvn", "spring-boot:run"]
   ```
   *Figura 6.13 – Dockerfile para la demo de Spring Boot*

9. Construir y ejecutar el contenedor:
   ```bash
   $ docker image build -t java-demo .
   $ docker container run --name java-demo --rm \
     -p 8080:8080 \
     -v $(pwd)/.:/app \
     java-demo
   ```
   Al modificar el código en `DemoApplication.java` (por ejemplo, añadiendo `"Crocodile"` o `"Penguin"`), Spring Boot recompilará y reiniciará la aplicación dentro del contenedor.

---

### Auto-reinicio para Python (Flask)

#### Instalación y configuración:
- En macOS: `$ brew install python`
- En Windows: `$ choco install python`
- Verificar: `$ python3 --version`

1. Inicializar proyecto:
   ```bash
   $ mkdir python-demo && cd python-demo
   $ code .
   ```
2. Crear `requirements.txt`:
   ```text
   flask
   ```
3. Crear `main.py`:
   ```python
   from flask import Flask
   app = Flask(__name__)

   @app.route("/")
   def hello():
       return "Hello World!"

   if __name__ == "__main__":
       app.run(host="0.0.0.0", port=5000)
   ```
   *Figura 6.14 – Contenido del archivo main.py*

4. Crear entorno virtual e instalar dependencias:
   ```bash
   $ python3 -m venv .venv
   $ source .venv/bin/activate
   $ pip install -r requirements.txt
   ```
5. Habilitar auto-reinicio con `debug=True`:
   ```python
   app.run(host = "0.0.0.0", port = 5000, debug = True)
   ```
   o ejecutando nodemon: `$ nodemon --exec python main.py` (*Figura 6.15*).
6. Añadir endpoint `/colors`:
   ```python
   from flask import Flask, jsonify

   @app.route("/colors")
   def colors():
       return jsonify(["red", "green", "blue"])
   ```
   *Figura 6.16 – nodemon detectando cambios en el código Python*

7. Crear el `Dockerfile-dev`:
   ```dockerfile
   FROM python:3.13-slim
   WORKDIR /app
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .
   ENV FLASK_APP=main.py
   ENV FLASK_ENV=development
   CMD ["flask", "run", "--host=0.0.0.0", "--reload"]
   ```
   *Figura 6.17 – Dockerfile para la aplicación Python de ejemplo*

8. Construir y ejecutar:
   ```bash
   $ docker image build -f Dockerfile-dev \
     -t python-sample .
   $ docker container run --rm \
     -p 5000:5000 \
     -v $(pwd)/.:/app \
     python-sample
   ```
   *Figura 6.18 – Ejecución de la aplicación Python contenerizada*

---

### Auto-reinicio para .NET (C#)

#### Instalación y configuración:
- macOS: `$ brew install --cask dotnet-sdk`
- Windows: `$ choco install -y dotnet-sdk`
- Verificar: `$ dotnet --list-sdks`

1. Crear el proyecto Web API:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
   $ cd chapter-06
   $ dotnet new webapi -o csharp-sample
   $ cd csharp-sample
   $ code .
   ```
   *Figura 6.19 – Solicitud de carga de dependencias faltantes para .NET*  
   *Figura 6.20 – Estructura del proyecto .NET en el explorador de VS Code*

2. Ejecutar con `dotnet watch`:
   ```bash
   $ dotnet watch run
   ```
   *Figura 6.21 – Ejecución de la Web API en el host*  
   *Figura 6.23 – Ejecución con la tarea dotnet watch*  
   *Figura 6.24 – Auto-reinicio de la aplicación .NET ante cambios*

3. Probar con `curl`:
   ```bash
   $ curl http://localhost:5080/WeatherForecast
   ```

4. Crear el `Dockerfile-dev`:
   ```dockerfile
   FROM mcr.microsoft.com/dotnet/sdk:9.0
   WORKDIR /app
   COPY . .
   RUN dotnet restore
   CMD ["dotnet", "watch", "run", "--urls", "http://0.0.0.0:5000"]
   ```
   *Figura 6.25 – Dockerfile para la aplicación .NET de ejemplo*

5. Construir y ejecutar el contenedor:
   ```bash
   $ docker image build -f Dockerfile-dev \
     -t csharp-sample .
   $ docker container run --rm \
     --name csharp-sample \
     -p 5000:5000 \
     -v $(pwd):/app \
     csharp-sample
   ```
   *Figura 6.26 – Ejecución de la aplicación .NET con hot reload*

6. Limpieza:
   ```bash
   $ docker container rm --force csharp-sample
   ```

---

### Depuración de código línea por línea dentro de un contenedor

#### Depuración de una aplicación Node.js con VS Code:
1. Iniciar el contenedor exponiendo el puerto de depuración `9229` y habilitando el flag `--inspect=0.0.0.0`:
   ```bash
   $ docker container run --rm -it --init \
     --name node-demo \
     -p 3000:3000 \
     -p 9229:9229 \
     -v $(pwd):/app \
     node-demo-dev node --inspect=0.0.0.0 index.js
   ```
2. Crear `.vscode/launch.json`:
   ```json
   {
     "version": "0.2.0",
     "configurations": [
       {
         "type": "node",
         "request": "attach",
         "name": "Docker: Attach to Node",
         "port": 9229,
         "address": "localhost",
         "localRoot": "${workspaceFolder}",
         "remoteRoot": "/app"
       }
     ]
   }
   ```
   *Figura 6.27 – Configuración de lanzamiento para depurar Node.js*

3. Establecer un punto de interrupción (*breakpoint*) en `index.js` (*Figura 6.28*).
4. En VS Code, abrir la vista **Run and Debug** (`Cmd + Shift + D` / `Ctrl + Shift + D`), seleccionar **Docker: Attach to Node** y hacer clic en el botón verde de inicio (*Figura 6.29*).
5. Ejecutar `curl localhost:3000/colors` y observar cómo la ejecución se detiene en el breakpoint (*Figura 6.30*).
6. Para soportar auto-reinicio y reconexión automática del depurador con `nodemon`:
   ```bash
   $ docker container run --rm --init \
     --name node-demo \
     -p 3000:3000 \
     -p 9229:9229 \
     -v $(pwd):/app \
     node-demo-dev nodemon --inspect=0.0.0.0 index.js
   ```
   Añadir `"restart": true` a la configuración en `launch.json`:
   ```json
   {
     "type": "node",
     "request": "attach",
     "name": "Docker: Attach to Node",
     "remoteRoot": "/app",
     "restart": true
   }
   ```
   *Figura 6.31 – Inicio de Node.js con nodemon y depuración activa*  
   *Figura 6.32 – nodemon reiniciando y reconectando el depurador automáticamente*

---

### Depuración de una aplicación .NET con VS Code

1. Instalar la extensión oficial de **C# para VS Code**.
2. Abrir la paleta de comandos (`Cmd + Shift + P` o `Ctrl + Shift + P`) y ejecutar `Docker: Add Docker Files to Workspace`.
3. Seleccionar **ASP.NET Core**, **Linux** y el puerto correspondiente.
4. En `.vscode/launch.json`, configurar la acción `dockerServerReadyAction`:
   *Figura 6.33 – Modificación de la configuración de lanzamiento de Docker*
5. Seleccionar la tarea **Containers: .NET Launch** en la vista de depuración y presionar F5 (*Figura 6.34*).
6. VS Code construirá la imagen, iniciará el contenedor y detendrá la ejecución en los breakpoints definidos en `Program.cs` (*Figura 6.35*).

---

### Instrumentación de código para generar información de logging significativa

En producción no es viable depurar interactivamente. Las aplicaciones deben emitir registros estructurados hacia `STDOUT` y `STDERR`.

#### Niveles de severidad de logs:

| Nivel de Log | Descripción |
| :--- | :--- |
| **`TRACE`** | Información de grano muy fino para capturar cada detalle del comportamiento. |
| **`DEBUG`** | Información granular de diagnóstico para localizar problemas potenciales. |
| **`INFO`** | Hitos y eventos normales del ciclo de vida (arranque, parada, peticiones). |
| **`WARNING`** | Situaciones inusuales o posibles anomalías que no impiden el funcionamiento. |
| **`ERROR`** | Errores graves que impiden completar una operación o tarea importante. |
| **`FATAL`** | Fallo catastrófico que requiere la detención inmediata de la aplicación. |

*Tabla 6.1 – Lista de niveles de severidad en registros*

#### Instrumentación en Python:
```python
import logging

logging.basicConfig(level=logging.WARNING)
logger = logging.getLogger("my-app")
handler = logging.StreamHandler()
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
handler.setFormatter(formatter)
logger.addHandler(handler)
```
*Figura 6.36 – Definición de un logger en Python*  
*Figura 6.37 – Instrumentación de un método con logging INFO*  
*Figura 6.38 – Generación de un mensaje WARNING*  
*Figura 6.39 – Salida filtrada por severidad en la consola*

#### Instrumentación en .NET (C#):
```bash
$ dotnet add package Microsoft.Extensions.Logging
```
*Figura 6.40 – Integración de logging en .NET*  
*Figura 6.41 – Registro de mensaje INFO*  
*Figura 6.42 – Registro de mensaje WARNING*  
*Figura 6.43 – Salida de logs de la aplicación .NET*

---

### Uso de OpenTelemetry y Jaeger para monitorización y resolución de problemas

En sistemas distribuidos basados en microservicios, el seguimiento de transacciones de extremo a extremo (*distributed tracing*) permite identificar cuellos de botella y latencias entre servicios mediante **OpenTelemetry** y **Jaeger**.

#### Instrumentación de .NET con OpenTelemetry y Jaeger:
1. Instalar paquetes de OpenTelemetry:
   ```bash
   dotnet add package OpenTelemetry.Extensions.Hosting
   dotnet add package OpenTelemetry.Instrumentation.AspNetCore
   dotnet add package OpenTelemetry.Instrumentation.Http
   dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
   dotnet add package OpenTelemetry.Exporter.Console
   ```

2. Configurar OpenTelemetry en `Program.cs` (*Figura 6.44*).

3. Iniciar el servidor central de Jaeger con Docker:
   ```bash
   docker run -d --name jaeger \
     -e COLLECTOR_OTLP_ENABLED=true \
     -p 16686:16686 \
     -p 4317:4317 \
     jaegertracing/all-in-one:latest
   ```

4. Ejecutar la aplicación .NET (`dotnet run`) y enviar peticiones a los endpoints.

5. Abrir la interfaz de Jaeger en `http://localhost:16686` para visualizar las trazas (*Figura 6.45*, *Figura 6.49*, *Figura 6.50*).

6. Limpieza:
   ```bash
   $ docker container rm -f jaeger
   ```

#### Instrumentación distribuida en Java Spring Boot (Service1 -> Service2):
1. Navegar al proyecto `chapter-06/solutions/simple-tracing-java`.
2. Iniciar Jaeger y los dos microservicios:
   ```bash
   $ ./docker.sh
   $ ./start-both-services.sh
   ```
3. Generar peticiones distribuidas:
   ```bash
   $ curl localhost:8080/hello
   $ curl localhost:8081/nested
   $ curl localhost:8080/test
   ```
4. Consultar en Jaeger (`http://localhost:16686`) la traza distribuida completa donde `service1` invoca internamente a `service2` (*Figura 6.51*).
5. Limpieza:
   ```bash
   $ docker rm -f $(docker ps -q --filter ancestor=jaegertracing/all-in-one:1.48)
   ```

#### Orquestación con Docker Compose (`docker-compose.yml`):
```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:1.48
    container_name: jaeger-tracing
    ports:
      - "16686:16686" # Jaeger UI
      - "14250:14250" # gRPC endpoint for traces
  service2:
    build:
      context: .
      dockerfile: service2/Dockerfile
    container_name: service2-app
    ports:
      - "8081:8081"
    environment:
      - OTEL_EXPORTER_JAEGER_ENDPOINT=http://jaeger:14250
      - OTEL_SERVICE_NAME=service2
  service1:
    build:
      context: .
      dockerfile: service1/Dockerfile
    container_name: service1-app
    ports:
      - "8080:8080"
    environment:
      - OTEL_EXPORTER_JAEGER_ENDPOINT=http://jaeger:14250
      - OTEL_SERVICE_NAME=service1
      - SERVICE2_URL=http://service2:8081
```

Ejecución de todo el entorno:
```bash
$ docker compose up -d
```

---

### Resumen

En este capítulo aprendimos a:
- Montar código fuente en vivo en contenedores mediante `--volume $(pwd):/app` para eliminar la necesidad de reconstruir imágenes tras cada cambio.
- Integrar herramientas de recarga automática en caliente (*hot-reloading*): `nodemon` (Node.js/Python), `Spring Boot DevTools` (Java), `Flask --reload` (Python) y `dotnet watch` (.NET).
- Configurar VS Code para conectarse y depurar aplicaciones línea por línea dentro de contenedores exponiendo puertos de depuración (`--inspect` / `9229`).
- Estructurar logs de aplicación hacia `STDOUT`/`STDERR` mediante niveles de severidad (`TRACE`, `DEBUG`, `INFO`, `WARNING`, `ERROR`, `FATAL`).
- Implementar observabilidad y trazabilidad distribuida con OpenTelemetry y Jaeger para monitorizar microservicios complejos.

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

1. **¿Qué opción de `docker run` permite mapear tu carpeta de código local dentro de un contenedor para que los cambios se reflejen inmediatamente?**
2. **Tras montar tu código en un contenedor, modificas y guardas un archivo. ¿Por qué la aplicación no toma el cambio automáticamente y qué pasos manuales debes realizar?**
3. **En una imagen de desarrollo de Node.js, ¿qué herramienta puedes instalar y ejecutar en lugar de `node index.js` para reiniciar la aplicación ante cambios de código? ¿Cómo modificas tu Dockerfile?**
4. **¿Qué dependencia debes añadir al archivo `pom.xml` (o `build.gradle`) de un proyecto Spring Boot para habilitar el reinicio automático dentro de un contenedor?**
5. **¿Cuáles fueron los dos enfoques mostrados para auto-reiniciar una aplicación Flask en Python dentro de un contenedor?**
6. **¿Qué comando de .NET reemplaza a `dotnet run` para vigilar cambios de archivos y aplicar *hot-reload* dentro de un contenedor?**
7. **¿Cuándo y por qué depurarías código línea por línea dentro de un contenedor?**
8. **Al depurar una aplicación Node.js en un contenedor con VS Code, ¿qué flag añades al comando `node` y qué puerto debes publicar en Docker?**
9. **En el Dockerfile de desarrollo recomendado para Node.js, ¿por qué instalamos `nodemon` y cambiamos `CMD` a `["nodemon", "--legacy-watch", "index.js"]`?**
10. **¿Por qué es fundamental instrumentar el código con buena información de depuración y logging?**
11. **¿Por qué es preferible escribir los mensajes de log hacia `STDOUT`/`STDERR` al ejecutar en contenedores y cómo se consultan desde el host?**
12. **¿Cuál es el propósito de instrumentar microservicios con APIs de OpenTracing / OpenTelemetry exportando hacia Jaeger, y qué imagen de Docker se utiliza habitualmente para levantar un backend local de Jaeger?**

---

### Respuestas

1. **Montaje de código en vivo**:  
   Utilizando el flag `--volume $(pwd):/app` (o `-v $(pwd):/app`). Cualquier cambio guardado en local estará disponible inmediatamente en `/app` dentro del contenedor.

2. **Fricción por reinicio manual**:  
   El proceso de la aplicación en memoria (como `node index.js`) no recarga los archivos automáticamente. Es necesario:
   - Detener el contenedor con `Ctrl + C`.
   - Volver a ejecutar `docker container run ...` para reiniciar el proceso.

3. **Auto-reinicio con `nodemon`**:  
   Instalar `nodemon` y configurar el `Dockerfile.dev`:
   ```dockerfile
   RUN npm install -g nodemon
   CMD ["nodemon", "--legacy-watch", "index.js"]
   ```

4. **Spring Boot DevTools**:  
   Añadiendo en `pom.xml`:
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-devtools</artifactId>
       <optional>true</optional>
   </dependency>
   ```

5. **Reinicio en Flask**:
   - Activar el recargador nativo de Flask con `FLASK_ENV=development`, `FLASK_DEBUG=1` (o `debug=True` / `flask run --reload`).
   - Ejecutar la aplicación con nodemon: `nodemon --exec python main.py`.

6. **Hot-reload en .NET**:  
   Utilizar el comando `dotnet watch run` en lugar de `dotnet run`.

7. **Motivos para depurar en contenedor**:  
   Cuando un error no puede reproducirse en el host local debido a diferencias de entorno, librerías o dependencias del sistema operativo, o cuando no es posible cubrir un escenario complejo mediante pruebas unitarias o de integración.

8. **Configuración de depuración en Node.js**:  
   Añadir `--inspect=0.0.0.0` al comando de Node y mapear el puerto de depuración `-p 9229:9229`.

9. **Propósito de `nodemon` en el Dockerfile de desarrollo**:  
   Permite vigilar el código montado desde el host y reiniciar automáticamente el servidor en cuanto se detecta un cambio, eliminando reconstrucciones y reinicios manuales.

10. **Importancia del logging**:  
    En entornos productivos no es posible acoplar un depurador interactivo. Los logs detallados y estructurados son la principal fuente de información para diagnosticar fallos y entender el comportamiento del sistema.

11. **Salida a STDOUT/STDERR**:  
    Permite que Docker capture los eventos a través de sus controladores de registro (*logging drivers*, como `json-file`). Se consultan en el host con `docker container logs -f <contenedor>` o se redirigen a sistemas centralizados (Loki, Elasticsearch, CloudWatch).

12. **Trazabilidad distribuida con Jaeger**:  
    Permite rastrear peticiones a través de múltiples microservicios, midiendo latencias y detectando cuellos de botella. Para desarrollo local se utiliza la imagen:
    ```bash
    docker run -d --name jaeger \
      -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 \
      -p 5775:5775/udp \
      -p 6831:6831/udp \
      -p 6832:6832/udp \
      -p 5778:5778 \
      -p 16686:16686 \
      -p 14268:14268 \
      -p 9411:9411 \
      jaegertracing/all-in-one:1.42
    ```

