# Parte 3: Fundamentos de la Orquestación
## Capítulo 12: Envío de logs y monitorización de contenedores

En el capítulo anterior, presentamos Docker Compose como una herramienta declarativa indispensable para definir y ejecutar aplicaciones multi-contenedor en un único host.

En este capítulo, pasamos de simplemente ejecutar contenedores a **entender y operar su comportamiento en producción**. Analizaremos por qué el registro de eventos (*logging*) y la monitorización (*monitoring*) son esenciales en cualquier entorno contenerizado y cómo implementarlos eficazmente. Aprenderás a recolectar, enviar y visualizar registros con la pila **Elastic Stack (Filebeat, Elasticsearch y Kibana)**, a instrumentar aplicaciones en múltiples lenguajes (Go, Python y C#) para exponer métricas compatibles con **Prometheus**, y a crear paneles de control visuales y alertas en **Grafana**. Finalmente, ampliaremos el concepto hacia la **observabilidad integral** (logs, métricas y trazas) y la monitorización de seguridad en tiempo de ejecución.

---

### Temas tratados en este capítulo:
- Por qué importan el registro de eventos y la monitorización
- Envío y centralización de logs de contenedores
- Configuración de controladores de registro (*log drivers*) y políticas de rotación/retención
- Envío de logs del daemon de Docker
- Consulta y filtrado de logs centralizados con Kibana
- Recolección y sondeo (*scraping*) de métricas con Prometheus
- Monitorización de aplicaciones y métricas de sistema con Node Exporter y Grafana
- Observabilidad integral y monitorización de seguridad

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Configurar controladores de registro y políticas de rotación de logs para evitar el agotamiento de disco.
- Recolectar y enviar logs de contenedores y del daemon a una instancia centralizada de Elastic Stack.
- Consultar, filtrar y crear visualizaciones de logs agregados en Kibana.
- Instrumentar servicios en diferentes lenguajes para exponer métricas a Prometheus.
- Diseñar paneles interactivos y configurar alertas en Grafana.
- Comprender los tres pilares de la observabilidad (logs, métricas y trazas) y su relación con la seguridad en tiempo de ejecución.

---

### Requisitos técnicos

El código fuente de este capítulo se encuentra en el repositorio del libro:  
[https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-12/solutions](https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-12/solutions)

Prepara el entorno local:
```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir chapter-12 && cd chapter-12
```

> **Diferencias entre plataformas (Linux vs. macOS/Windows)**:  
> En Linux nativo, Filebeat puede leer directamente los archivos de logs de Docker desde `/var/lib/docker/containers`.  
> En macOS y Windows, Docker Desktop ejecuta el motor dentro de una máquina virtual ligera aislada, haciendo inaccesible esa ruta desde el host. Para garantizar compatibilidad universal, este capítulo utiliza un patrón multiplataforma: las aplicaciones escriben sus logs en un volumen compartido montado que Filebeat procesa en tiempo real.

---

### Por qué importan el registro de eventos y la monitorización

Al igual que la cabina de un avión o el centro de control de una central nuclear cuentan con cientos de sensores e instrumentos para medir el estado del sistema en tiempo real, las aplicaciones distribuidas deben instrumentarse con "sensores" para medir su comportamiento:

- **Métricas funcionales**: Indicadores específicos del negocio (por ejemplo, pedidos procesados por minuto, canciones más reproducidas, registros de nuevos usuarios).
- **Métricas no funcionales**: Indicadores de rendimiento del sistema e infraestructura (por ejemplo, latencia media de endpoints, tasa de errores HTTP 4xx/5xx, consumo de memoria RAM y uso de CPU).

**Prometheus** es el estándar de código abierto (graduado por la CNCF) más utilizado para recolectar, almacenar y consultar métricas de series temporales en entornos de contenedores.

---

### Envío y gestión de logs de contenedores

Por defecto, Docker captura la salida estándar (`STDOUT`) y de error (`STDERR`) de cada contenedor, haciéndolas accesibles mediante `docker container logs <id>`.

#### Configuración de controladores de registro (*Logging Drivers*)
Docker soporta múltiples drivers: `json-file` (por defecto), `syslog`, `journald`, `fluentd`, entre otros.

#### 1. Configuración global en el daemon (`daemon.json`):
En `/etc/docker/daemon.json` (o en Docker Desktop -> Settings -> Docker Engine):
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  }
}
```
*Figura 12.1 y Figura 12.2 – Configuración global del log driver en Docker Desktop*

- `max-size`: Tamaño máximo de cada archivo de log antes de rotar (por ejemplo, `10m`).
- `max-file`: Cantidad máxima de archivos rotados a conservar (los más antiguos se eliminan automáticamente).

#### 2. Configuración individual por contenedor:
```bash
$ docker run --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=5 \
  <image_name> ...
```

---

### Arquitectura de registro con Elastic Stack (ELK)

La pila de Elastic Stack centraliza y procesa los eventos del sistema:
- **Filebeat**: Agente ligero de recolección y reenvío de logs (*log shipper*).
- **Elasticsearch**: Motor distribuido de indexación, búsqueda y analítica en tiempo real.
- **Kibana**: Interfaz web visual para consultar, filtrar y construir paneles de logs.

#### Manifiesto Compose para Elastic Stack en Linux nativo:
```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.2
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.15.2
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    ports:
      - "5601:5601"

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.15.2
    container_name: filebeat
    user: root
    depends_on:
      - elasticsearch
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
```
*Figura 12.3: Docker Compose para Elastic Stack y Filebeat en Linux*

---

### Patrón multiplataforma de envío de logs (macOS / Windows / Linux)

1. Crear la estructura del laboratorio:
   ```bash
   $ mkdir mac-or-windows && cd mac-or-windows
   $ mkdir app && cd app
   $ npm init -y
   $ npm install --save express
   ```

2. Crear `app/index.js`:
   ```javascript
   const express = require('express');
   const app = express();
   const port = process.env.PORT || 3000;

   app.use((req, res, next) => {
     console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
     next();
   });

   app.get('/', (req, res) => {
     res.send('Hello from Node.js App!');
   });

   app.get('/test', (req, res) => {
     console.log('[INFO] Test route accessed');
     res.send('Test route response');
   });

   app.use((req, res) => {
     console.error(`[ERROR] 404 Not Found: ${req.url}`);
     res.status(404).send('Not Found');
   });

   app.listen(port, () => {
     console.log(`Server is running at http://0.0.0.0:${port}`);
   });
   ```
   *Figura 12.4: Código de la aplicación Node.js*

3. Crear el script de entrada `app/entrypoint.sh` para bifurcar la salida con `tee`:
   ```bash
   #!/bin/sh
   set -e

   if [ -n "$LOGGING_FILE" ]; then
     mkdir -p "$(dirname "$LOGGING_FILE")"
     : > "$LOGGING_FILE"
     exec npm start 2>&1 | tee -a "$LOGGING_FILE"
   else
     exec npm start 2>&1
   fi
   ```
   *Figura 12.5: Script entrypoint.sh para redirigir logs al volumen*

   Hacer ejecutable: `$ chmod +x ./entrypoint.sh`

4. Crear `app/Dockerfile`:
   ```dockerfile
   FROM node:20-alpine
   WORKDIR /usr/src/app
   COPY package*.json ./
   RUN npm install
   COPY index.js entrypoint.sh ./
   RUN chmod +x entrypoint.sh
   EXPOSE 3000
   ENTRYPOINT ["./entrypoint.sh"]
   ```
   *Figura 12.6: Dockerfile de la aplicación*

5. Crear `mac-or-windows/compose.yml`:
   ```yaml
   services:
     app:
       build: ./app
       container_name: app
       ports:
         - "3000:3000"
       environment:
         - LOGGING_FILE=/usr/src/app/logs/app.log
       volumes:
         - app_logs:/usr/src/app/logs

     elasticsearch:
       image: docker.elastic.co/elasticsearch/elasticsearch:8.15.2
       container_name: elasticsearch
       environment:
         - discovery.type=single-node
         - xpack.security.enabled=false
       ports:
         - "9200:9200"

     filebeat:
       image: docker.elastic.co/beats/filebeat:8.15.2
       container_name: filebeat
       user: root
       depends_on:
         - elasticsearch
       volumes:
         - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
         - app_logs:/usr/src/app/logs:ro

   volumes:
     app_logs:
   ```
   *Figura 12.7: compose.yml con volumen compartido para Filebeat*

6. Crear `mac-or-windows/filebeat.yml`:
   ```yaml
   filebeat.inputs:
     - type: log
       enabled: true
       paths:
         - "/usr/src/app/logs/*.log"
       fields:
         service.name: "node-app"
       fields_under_root: true

   output.elasticsearch:
     hosts: ["http://elasticsearch:9200"]
   ```
   *Figura 12.8: Configuración de Filebeat para recolectar logs del volumen*

7. Compilar, ejecutar y generar tráfico:
   ```bash
   $ docker compose build app
   $ docker compose up --detach
   $ for i in {1..20}; do \
       curl -s http://localhost:3000 > /dev/null && \
       curl -s http://localhost:3000/test > /dev/null; \
     done
   ```

---

### Envío de logs del daemon de Docker

Los logs del daemon registran incidencias a nivel de plataforma (errores de red, asignación de recursos o fallos de orquestación):

- **En Linux**: Ubicados habitualmente en `/var/log/docker.log` o accesibles con `journalctl -u docker`.
- **En macOS**:
  ```bash
  $ sudo log stream --predicate 'senderImagePath CONTAINS "Docker"'
  $ sudo log show --predicate 'senderImagePath CONTAINS "Docker"' --style syslog --info --last 1d > docker_daemon_logs.log
  ```
- **En Windows 11**: Ubicados en `C:\ProgramData\DockerDesktop\service\DockerDesktopVM.log` o consultables mediante PowerShell:
  ```powershell
  Get-Content -Path "C:\ProgramData\DockerDesktop\service\DockerDesktopVM.log" -Tail 50
  ```

---

### Consulta y filtrado de logs con Kibana

#### Laboratorio práctico multi-servicio (`kibana-lab`)

1. Crear la estructura del proyecto:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-12
   $ mkdir kibana-lab && cd kibana-lab
   $ mkdir node-api ruby-worker
   ```

2. Crear el servicio `node-api` (`node-api/server.js`, `node-api/entrypoint.sh` y `node-api/Dockerfile`):
   ```javascript
   const express = require('express');
   const app = express();

   app.get('/', (req, res) => {
     console.log(`[INFO] Request received at /`);
     res.send('Hello from Node.js API');
   });

   app.get('/error', (req, res) => {
     console.error('[ERROR] Simulated error occurred!');
     res.status(500).send('Something went wrong');
   });

   app.listen(8080, '0.0.0.0', () => {
     console.log('Node.js API running on port 8080');
   });
   ```

   ```bash
   #!/bin/sh
   set -e

   if [ -n "$LOGGING_FILE" ]; then
     mkdir -p "$(dirname "$LOGGING_FILE")"
     : > "$LOGGING_FILE"
     exec node server.js 2>&1 | tee -a "$LOGGING_FILE"
   else
     exec node server.js 2>&1
   fi
   ```

   ```dockerfile
   FROM node:20-alpine
   WORKDIR /app
   COPY server.js entrypoint.sh ./
   RUN npm install express && chmod +x entrypoint.sh
   EXPOSE 8080
   CMD ["./entrypoint.sh"]
   ```

3. Crear el servicio en segundo plano `ruby-worker` (`ruby-worker/worker.rb`, `ruby-worker/entrypoint.sh` y `ruby-worker/Dockerfile`):
   ```ruby
   loop do
     puts "[INFO] Ruby worker heartbeat at #{Time.now}"
     $stdout.flush
     sleep 5
   end
   ```

   ```bash
   #!/bin/sh
   set -e

   if [ -n "$LOGGING_FILE" ]; then
     mkdir -p "$(dirname "$LOGGING_FILE")"
     : > "$LOGGING_FILE"
     exec ruby worker.rb 2>&1 | tee -a "$LOGGING_FILE"
   else
     exec ruby worker.rb 2>&1
   fi
   ```

   ```dockerfile
   FROM ruby:3.3-alpine
   WORKDIR /app
   COPY worker.rb entrypoint.sh ./
   RUN chmod +x entrypoint.sh
   CMD ["./entrypoint.sh"]
   ```

4. Crear `kibana-lab/filebeat.yml`:
   ```yaml
   filebeat.inputs:
     - type: log
       enabled: true
       paths:
         - "/usr/src/app/logs/app.log"
       fields:
         service.name: "node-api"
       fields_under_root: true

     - type: log
       enabled: true
       paths:
         - "/usr/src/app/logs/worker.log"
       fields:
         service.name: "ruby-worker"
       fields_under_root: true

   output.elasticsearch:
     hosts: ["http://elasticsearch:9200"]
   ```

5. Crear `kibana-lab/compose.yml`:
   ```yaml
   services:
     elasticsearch:
       image: docker.elastic.co/elasticsearch/elasticsearch:8.15.2
       container_name: elasticsearch
       environment:
         - discovery.type=single-node
         - xpack.security.enabled=false
       ports:
         - "9200:9200"

     kibana:
       image: docker.elastic.co/kibana/kibana:8.15.2
       container_name: kibana
       environment:
         - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
       depends_on:
         - elasticsearch
       ports:
         - "5601:5601"

     filebeat:
       image: docker.elastic.co/beats/filebeat:8.15.2
       container_name: filebeat
       user: root
       depends_on: [elasticsearch]
       volumes:
         - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
         - app_logs:/usr/src/app/logs:ro

     node-api:
       build: ./node-api
       container_name: node-api
       environment:
         - LOGGING_FILE=/usr/src/app/logs/app.log
       ports:
         - "8080:8080"
       volumes:
         - app_logs:/usr/src/app/logs

     ruby-worker:
       build: ./ruby-worker
       container_name: ruby-worker
       environment:
         - LOGGING_FILE=/usr/src/app/logs/worker.log
       volumes:
         - app_logs:/usr/src/app/logs

   volumes:
     app_logs:
   ```

6. Iniciar la pila y generar logs:
   ```bash
   $ docker compose build node-api ruby-worker
   $ docker compose up -d
   $ curl http://localhost:8080/
   $ curl http://localhost:8080/error
   ```
   *Figura 12.9, Figura 12.10 y Figura 12.11 – Contenedores en ejecución y salida de logs*

7. Configurar Kibana:
   - Abrir `http://localhost:5601` -> **Management** -> **Stack Management** -> **Data Views**.
   - Crear un Data View con patrón de índice `filebeat-*` y campo de marca de tiempo `@timestamp` (*Figura 12.12*).
   - Ir a **Analytics** -> **Discover** (*Figura 12.13*).

8. Ejemplos de consultas en KQL:
   - Filtrar solo los logs de la API: `service.name : "node-api"`
   - Filtrar errores: `message : "ERROR"`
   - Combinación lógica: `service.name : "node-api" AND message : "ERROR"`

9. Limpieza:
   ```bash
   $ docker compose down -v
   ```

---

### Recolección de métricas con Prometheus (Go, Python y C#)

Prometheus utiliza un **modelo pull**: sondea periódicamente los endpoints HTTP de métricas (por defecto `/metrics`) expuestos por cada servicio.

#### Laboratorio multi-lenguaje (`prometheus-demo`):
```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-12
$ mkdir prometheus-demo && cd prometheus-demo
$ mkdir go-service py-service cs-service prometheus
```

#### 1. Servicio en Go (`go-service/main.go` y Dockerfile):
```go
package main

import (
	"fmt"
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var reqCounter = prometheus.NewCounter(
	prometheus.CounterOpts{
		Name: "go_requests_total",
		Help: "Total number of requests",
	},
)

func main() {
	prometheus.MustRegister(reqCounter)
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		reqCounter.Inc()
		fmt.Fprintf(w, "Hello from Go service!")
	})
	http.Handle("/metrics", promhttp.Handler())
	http.ListenAndServe(":8081", nil)
}
```

```dockerfile
FROM golang:1.23-alpine AS build
WORKDIR /src
COPY . .
RUN go mod init go-service && go mod tidy && \
    go build -o app

FROM alpine
WORKDIR /app
COPY --from=build /src/app .
EXPOSE 8081
CMD ["./app"]
```

#### 2. Servicio en Python (`py-service/app.py` y Dockerfile):
```python
from flask import Flask
from prometheus_client import Counter, generate_latest
from prometheus_client import CONTENT_TYPE_LATEST
import logging, sys

app = Flask(__name__)

# Enable unbuffered logging to stdout
logging.basicConfig(stream=sys.stdout, level=logging.INFO)

# Define a simple counter
counter = Counter('py_requests_total', 'Total HTTP requests handled')

@app.route('/')
def home():
    counter.inc()
    app.logger.info("Handled request to /")
    return 'Hello from Python!'

@app.route('/metrics')
def metrics():
    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}

if __name__ == '__main__':
    app.logger.info("Starting Flask app on port 8082")
    app.run(host='0.0.0.0', port=8082)
```

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
RUN pip install flask prometheus_client
EXPOSE 8082
CMD ["python", "app.py"]
```

#### 3. Servicio en C# .NET (`cs-service/Program.cs` y Dockerfile):
```csharp
using Prometheus;

var counter = Metrics.CreateCounter(
    "cs_requests_total", "Total number of requests");

var app = WebApplication.Create();

app.MapGet("/", () => {
    counter.Inc();
    return "Hello from C# service!";
});

app.MapMetrics();
app.Run("http://0.0.0.0:8083");
```

```dockerfile
# Stage 1: build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
# Create project
RUN dotnet new web -n cs-service
WORKDIR /src/cs-service
# Add prometheus package
RUN dotnet add package prometheus-net.AspNetCore
# Copy Program.cs from your local folder into project
COPY Program.cs .
# Publish the app
RUN dotnet publish -c Release -o /out

# Stage 2: runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /out .
EXPOSE 8083
ENTRYPOINT ["dotnet", "cs-service.dll"]
```

#### 4. Configuración de Prometheus (`prometheus/prometheus.yml`):
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: "go-service"
    static_configs:
      - targets: ["go-service:8081"]

  - job_name: "py-service"
    static_configs:
      - targets: ["py-service:8082"]

  - job_name: "cs-service"
    static_configs:
      - targets: ["cs-service:8083"]
```

#### 5. Manifiesto `prometheus-demo/compose.yml`:
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"

  go-service:
    build: ./go-service
    container_name: go-service
    ports:
      - "8081:8081"

  py-service:
    build: ./py-service
    container_name: py-service
    ports:
      - "8082:8082"

  cs-service:
    build: ./cs-service
    container_name: cs-service
    ports:
      - "8083:8083"
```

Inicia la pila (`docker compose up -d`) y visita `http://localhost:9090/targets` para verificar el estado `UP` de los cuatro servicios. Consulta las métricas `go_requests_total`, `py_requests_total` y `cs_requests_total` tras generar peticiones en los puertos 8081, 8082 y 8083.

---

### Monitorización completa con Prometheus, Node Exporter y Grafana

1. Preparar la estructura:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-12
   $ mkdir monitoring && cd monitoring
   $ mkdir prometheus grafana app
   ```

2. Crear la aplicación Node.js instrumentada (`app/server.js` y `app/Dockerfile`):
   ```javascript
   const express = require('express');
   const client = require('prom-client');
   const app = express();

   const register = new client.Registry();
   const counter = new client.Counter({
     name: 'http_requests_total',
     help: 'Total number of HTTP requests',
   });
   register.registerMetric(counter);
   client.collectDefaultMetrics({ register });

   app.get('/', (req, res) => {
     counter.inc();
     res.send('Hello from monitored app!');
   });

   app.get('/metrics', async (req, res) => {
     res.set('Content-Type', register.contentType);
     res.end(await register.metrics());
   });

   app.listen(8080, () => console.log('App running on port 8080'));
   ```

   ```dockerfile
   FROM node:20-alpine
   WORKDIR /usr/src/app
   COPY server.js .
   RUN npm install express prom-client
   EXPOSE 8080
   CMD ["node", "server.js"]
   ```

3. Crear `prometheus/prometheus.yml`:
   ```yaml
   global:
     scrape_interval: 15s

   scrape_configs:
     - job_name: 'prometheus'
       static_configs:
         - targets: ['prometheus:9090']

     - job_name: 'node'
       static_configs:
         - targets: ['node-exporter:9100']

     - job_name: 'app'
       metrics_path: /metrics
       static_configs:
         - targets: ['app:8080']
   ```

4. Crear `monitoring/compose.yml`:
   ```yaml
   services:
     prometheus:
       image: prom/prometheus:latest
       container_name: prometheus
       volumes:
         - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
       ports:
         - "9090:9090"

     node-exporter:
       image: prom/node-exporter:latest
       container_name: node-exporter
       ports:
         - "9100:9100"

     app:
       build: ./app
       container_name: monitored-app
       ports:
         - "8080:8080"

     grafana:
       image: grafana/grafana:latest
       container_name: grafana
       ports:
         - "3000:3000"
       volumes:
         - ./grafana:/var/lib/grafana
   ```

5. Iniciar la pila:
   ```bash
   $ docker compose up -d
   ```

6. Configurar Grafana:
   - Acceder a `http://localhost:3000` (credenciales por defecto: `admin` / `admin`).
   - Ir a **Connections** -> **Data Sources** -> **Add data source** -> Seleccionar **Prometheus** con URL `http://prometheus:9090` -> Clic en **Save & Test**.
   - Crear un nuevo panel visualizando la métrica `http_requests_total` o métricas de sistema de Node Exporter como:
     - `node_cpu_seconds_total`
     - `node_memory_MemAvailable_bytes`
     - `node_filesystem_avail_bytes`
   - Configurar reglas de alerta proactivas en la pestaña **Alerting**.

7. Limpieza:
   ```bash
   $ docker compose down -v
   ```

---

### Observabilidad integral y monitorización de seguridad

La observabilidad trasciende la monitorización clásica integrando los **tres pilares fundamentales**:

1. **Logs**: Registran eventos discretos detallados en puntos exactos en el tiempo (*qué sucedió*).
2. **Métricas**: Indicadores numéricos agregados que muestran patrones de rendimiento y tendencias (*cómo se comporta el sistema*).
3. **Trazas distribuidas (*Traces*)**: Rastreo del recorrido de una petición individual a través de la malla de microservicios (*dónde y por qué ocurrieron cuellos de botella*).

#### Monitorización de seguridad en tiempo de ejecución:
- **Escaneo de imágenes**: Herramientas como Docker Scout, Trivy o Snyk identifican vulnerabilidades (CVEs) en dependencias antes del despliegue.
- **Detección de intrusiones en tiempo de ejecución**: Motores como **Falco** o **Sysdig** auditan llamadas al sistema del kernel de Linux y alertan ante actividades anómalas (por ejemplo, ejecución de una shell interactiva dentro de un contenedor web o modificación de binarios protegidos).
- **Auditoría de eventos**: Envío de logs del daemon de Docker y de la API de Kubernetes a sistemas SIEM para trazabilidad y cumplimiento normativo.

---

### Resumen

En este capítulo aprendimos:
- La importancia del registro de eventos y la recolección de métricas funcionales y no funcionales.
- La configuración de drivers de log (`json-file`) y parámetros de rotación (`max-size`, `max-file`).
- La construcción de tuberías de logs centralizadas con Filebeat, Elasticsearch y Kibana.
- El patrón de volumen compartido para permitir recolección de logs uniforme en macOS, Windows y Linux.
- La instrumentación de aplicaciones en Go, Python, C# y Node.js para exponer métricas a Prometheus.
- La integración de Node Exporter y Grafana para visualización interactiva y alertas de rendimiento.
- La convergencia de logs, métricas y trazas en la observabilidad y su impacto directo en la seguridad de contenedores.

---

### Preguntas

1. **¿Por qué es importante el registro centralizado de logs en entornos de contenedores?**
2. **¿Qué función cumple Filebeat dentro del Elastic Stack?**
3. **¿Por qué Filebeat no puede leer los logs de Docker directamente en macOS y Windows?**
4. **¿Cómo funciona la solución alternativa basada en volúmenes compartidos?**
5. **¿Cuál es el beneficio de añadir campos personalizados como `service.name` en Filebeat?**
6. **¿En qué se diferencian los logs de las métricas?**
7. **¿Qué hace Prometheus y cómo recolecta los datos?**
8. **¿Cuál es el rol de Grafana en la pila de monitorización?**
9. **¿Por qué se utilizaron tres lenguajes distintos (Go, Python y C#) en el ejemplo de Prometheus?**
10. **¿Qué es la observabilidad y en qué se diferencia de la monitorización tradicional?**
11. **¿Cómo ayuda la observabilidad a la seguridad de los contenedores?**
12. **¿Cuál es la conclusión principal antes de abordar la seguridad de contenedores?**

---

### Respuestas

1. **Importancia del logging centralizado**:  
   Dado que los contenedores son efímeros y pueden destruirse o recrearse en diferentes nodos, sus logs locales se pierden al detenerse. Centralizar los logs asegura persistencia histórica y permite correlacionar eventos de múltiples microservicios en un único lugar.

2. **Rol de Filebeat**:  
   Es un agente ligero (*log shipper*) que monitoriza archivos de registro en los contenedores o hosts, enriquece los eventos con metadatos y los transmite de forma segura y eficiente hacia Elasticsearch.

3. **Limitación en macOS y Windows**:  
   Docker Desktop ejecuta el motor de contenedores dentro de una máquina virtual Linux ligera. La ruta interna `/var/lib/docker/containers` reside dentro de la VM y no es accesible directamente desde el sistema de archivos del host.

4. **Funcionamiento del volumen compartido**:  
   Los contenedores escriben sus salidas en archivos ubicados en un volumen Docker compartido (e.g. `/usr/src/app/logs`). Filebeat monta el mismo volumen en modo solo lectura y procesa las líneas añadidas, haciéndolo 100% portable entre Linux, macOS y Windows.

5. **Beneficio de `service.name`**:  
   Enriquece cada registro con una etiqueta identificativa del microservicio de origen, permitiendo filtrar y segmentar consultas en Kibana de forma clara.

6. **Diferencia entre logs y métricas**:  
   Los logs capturan eventos individuales discretos con mensajes contextuales (*qué pasó y cuándo*). Las métricas son valores numéricos medidos en intervalos continuos (*CPU, consumo de RAM, tasa de peticiones*) útiles para observar tendencias y alertas.

7. **Funcionamiento de Prometheus**:  
   Es un sistema de monitorización y alertas que sondea periódicamente (*pull model*) endpoints HTTP `/metrics` expuestos por aplicaciones y exportadores, almacenando los datos como series temporales indexadas por etiquetas.

8. **Rol de Grafana**:  
   Es una plataforma de visualización que se conecta a Prometheus para consultar métricas, renderizar gráficos interactivos en paneles de control y disparar notificaciones de alerta ante anomalías.

9. **Uso de múltiples lenguajes en el ejemplo**:  
   Para demostrar que Prometheus es agnóstico del lenguaje; cualquier tecnología que exponga un endpoint HTTP en formato de texto estándar de Prometheus puede integrarse sin inconvenientes.

10. **Monitorización frente a Observabilidad**:  
    La monitorización comprueba umbrales predefinidos sobre problemas conocidos (*si el sistema está caído*). La observabilidad correlaciona logs, métricas y trazas para deducir los estados internos del sistema y comprender fallos desconocidos o inesperados (*por qué está fallando*).

11. **Observabilidad y seguridad**:  
    Permite detectar anomalías de tráfico de red, picos inusuales de recursos o llamadas al sistema sospechosas en tiempo de ejecución, facilitando la detección temprana de intrusiones y el análisis forense.

12. **Conclusión principal**:  
    Obtener visibilidad completa a través de logs centralizados y métricas en tiempo real constituye el prerrequisito indispensable para asegurar, diagnosticar y operar contenedores en producción.

