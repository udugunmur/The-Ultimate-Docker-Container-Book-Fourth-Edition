# Parte 4: Docker, Kubernetes y la Nube
## Capítulo 18: Monitorización y resolución de problemas de una aplicación en producción

En el capítulo anterior aprendimos a desplegar aplicaciones contenerizadas en la nube utilizando servicios gestionados de Kubernetes (EKS, AKS y GKE) y plataformas serverless. Sin embargo, el despliegue no es el final del camino, sino el inicio de su ciclo de vida operativo. Una vez que el sistema está en vivo en producción, el objetivo pasa de "hacer que funcione" a **garantizar que funcione de manera fiable y predecible**.

Operar aplicaciones a gran escala requiere **observabilidad**: la capacidad de inferir el estado interno de un sistema a partir del análisis de sus señales y salidas externas (trazas, métricas y logs). Sin observabilidad, la resolución de incidentes se convierte en un juego de adivinanzas a ciegas.

En este capítulo construiremos una solución integral de observabilidad para microservicios. Comenzaremos instrumentando aplicaciones con el estándar abierto **OpenTelemetry (OTel)** para capturar trazas distribuidas y visualizarlas en **Jaeger**. A continuación, extenderemos la arquitectura incorporando la recolección y consulta de métricas de rendimiento con **Prometheus**, y diseñando cuadros de mando (*dashboards*) interactivos en **Grafana**. Finalmente, transformaremos la telemetría en acciones operativas concretas mediante la definición de **alertas inteligentes en Alertmanager**, el diseño de **guías de resolución de incidentes (*runbooks*)** y el análisis metódico de fallos en Kubernetes.

---

### Temas tratados en este capítulo:
- Instrumentación de microservicios con OpenTelemetry (Python/Flask y Node.js/Express)
- Trazado distribuido y visualización de latencias extremo a extremo con Jaeger
- Recolección y almacenamiento de métricas temporales con Prometheus
- Visualización de rendimiento y salud operativa mediante paneles en Grafana
- Definición de reglas de alerta en PromQL y enrutamiento con Alertmanager
- Creación de guías operativas (*runbooks*) para respuesta rápida ante incidentes
- Estrategias de depuración y resolución de problemas (*troubleshooting*) en Kubernetes

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Instrumentar servicios en Python y Node.js utilizando los SDKs de OpenTelemetry para propagar el contexto de trazas a través de llamadas HTTP.
- Desplegar y configurar el OpenTelemetry Collector para recibir, procesar y reenviar telemetría en formato OTLP.
- Desplegar y conectar Prometheus y Grafana para recopilar métricas y construir cuadros de mando profesionales.
- Escribir expresiones PromQL para calcular tasas de error y cuantiles de latencia (p95/p99) mediante métricas tipo Counter e Histogram.
- Configurar reglas de alerta predictivas, enrutarlas hacia canales de notificación (Slack/Email) con Alertmanager y documentar su mitigación en runbooks ejecutables.

---

### Requisitos técnicos

- Clúster de Kubernetes en ejecución (en la nube con EKS/AKS/GKE o localmente con minikube/kind).
- Herramienta CLI `kubectl` configurada:
  ```bash
  $ kubectl config get-contexts
  $ kubectl get nodes
  ```
- Docker Desktop o Docker Engine para compilar y publicar imágenes.
- Gestor de paquetes Helm 3 (opcional para despliegues simplificados).

Preparar el espacio de trabajo:
```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir -p chapter-18/otel-demo
$ cd chapter-18/otel-demo
```

---

### Instrumentación de servicios con OpenTelemetry

**OpenTelemetry (OTel)** es el estándar abierto de la Cloud Native Computing Foundation (CNCF) que unifica la recolección de trazas, métricas y logs.

Sus tres componentes principales son:
1. **Bibliotecas de instrumentación (SDKs)**: Incrustadas en el código fuente para capturar intervalos de tiempo (*spans*), errores y contexto.
2. **OpenTelemetry Collector**: Componente intermedio que recibe, procesa en lotes (*batch*) y exporta los datos hacia diversos motores de almacenamiento.
3. **Exportadores (*Exporters*)**: Adaptadores que envían la telemetría en protocolos abiertos (como OTLP gRPC/HTTP) a Jaeger, Prometheus, Datadog, etc.

#### ¿Qué es el trazado distribuido (*Distributed Tracing*)?
Una **traza (*trace*)** representa el ciclo de vida completo de una petición a través de todos los microservicios. Cada unidad atómica de trabajo se denomina **tramo (*span*)**. La traza forma un árbol de spans que refleja las dependencias, tiempos de ejecución y cuellos de botella exactos.

```
Petición de Usuario (POST /order)
 |
 +--> [ orders-service: 120ms ] (Span Raíz)
        |
        +--> [ payments-service: 55ms ] (Span Hijo)
```
*Figura 18.1: Ilustración conceptual de una traza distribuida a través de múltiples servicios*

---

### Laboratorio práctico: Instrumentación con OpenTelemetry

Desplegaremos dos microservicios comunicados entre sí: `orders-service` (Python/Flask) y `payments-service` (Node.js/Express).

#### Paso 1: Servicio de pedidos en Python y Flask (`python-orders`)
Crea `python-orders/app.py`:
```python
from flask import Flask, request, jsonify
import requests
import os

# --- OpenTelemetry imports
from opentelemetry import trace
from opentelemetry.instrumentation.flask import (
    FlaskInstrumentor,
)
from opentelemetry.instrumentation.requests import (
    RequestsInstrumentor,
)
from opentelemetry.sdk.resources import (
    SERVICE_NAME,
    Resource,
)
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor
)
from opentelemetry.exporter.otlp.proto.http.trace_exporter import (
    OTLPSpanExporter,
)
from trace_exporter import OTLPSpanExporter

SERVICE = os.getenv("SERVICE_NAME", "orders-service")
OTEL_ENDPOINT = os.getenv(
    "OTEL_EXPORTER_OTLP_ENDPOINT",
    "http://otel-collector:4318",
)

# --- Configure tracer provider
trace.set_tracer_provider(
    TracerProvider(
        resource=Resource.create({SERVICE_NAME: SERVICE})
    )
)
tracer_provider = trace.get_tracer_provider()
tracer_provider.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(
            endpoint=f"{OTEL_ENDPOINT}/v1/traces"
        )
    )
)

app = Flask(__name__)

# Auto-instrument Flask + outgoing requests
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

@app.route("/order", methods=["POST"])
def order():
    payload = request.get_json(
        force=True,
        silent=True,
    ) or {}
    # Simulate a downstream call to payments-service
    try:
        resp = requests.get(
            "http://payments-service:8080/pay",
            timeout=2,
        )
        pay_status = {
            "payments_status": resp.text,
            "payments_code": resp.status_code,
        }
    except Exception as ex:
        pay_status = {
            "payments_status": f"error: {ex}",
            "payments_code": 500,
        }
    return jsonify(
        {
            "service": SERVICE,
            "message": "Order received",
            "payload": payload,
            "downstream": pay_status,
        }
    )

@app.route("/healthz")
def health():
    return "ok"

if __name__ == "__main__":
    # Flask dev server; in container we use it directly
    app.run(host="0.0.0.0", port=8080)
```

Crea `python-orders/requirements.txt`:
```
flask==3.0.0
requests==2.32.3
setuptools<75
opentelemetry-api==1.26.0
opentelemetry-sdk==1.26.0
opentelemetry-exporter-otlp==1.26.0
opentelemetry-instrumentation==0.47b0
opentelemetry-instrumentation-flask==0.47b0
opentelemetry-instrumentation-requests==0.47b0
```

Crea `python-orders/Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
# opentelemetry config via env; can be overridden in Kubernetes
ENV SERVICE_NAME=orders-service \
    OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
EXPOSE 8080
CMD ["python", "app.py"]
```

#### Paso 2: Servicio de pagos en Node.js y Express (`node-payments`)
Crea `node-payments/package.json`:
```json
{
  "name": "payments-service",
  "version": "1.0.0",
  "description": "Node payments service with OpenTelemetry (OTLP/HTTP)",
  "main": "index.js",
  "license": "MIT",
  "dependencies": {
    "express": "^4.19.2",
    "@opentelemetry/sdk-node": "^0.53.0",
    "@opentelemetry/api": "^1.9.0",
    "@opentelemetry/exporter-trace-otlp-http": "^0.53.0",
    "@opentelemetry/auto-instrumentations-node": "^0.53.0",
    "@opentelemetry/resources": "^1.9.0",
    "@opentelemetry/semantic-conventions": "^1.9.0"
  }
}
```

Crea `node-payments/index.js`:
```javascript
const express = require("express");

// --- OpenTelemetry imports
const { NodeSDK } = require("@opentelemetry/sdk-node");
const { OTLPTraceExporter } = require("@opentelemetry/exporter-trace-otlp-http");
const { getNodeAutoInstrumentations } = require("@opentelemetry/auto-instrumentations-node");
const { Resource } = require("@opentelemetry/resources");
const { SemanticResourceAttributes } = require("@opentelemetry/semantic-conventions");

const SERVICE_NAME = process.env.SERVICE_NAME || "payments-service";
const OTEL_ENDPOINT = process.env.OTEL_EXPORTER_OTLP_ENDPOINT || "http://otel-collector:4318";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: `${OTEL_ENDPOINT}/v1/traces`,
  }),
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: SERVICE_NAME,
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

// Works whether start() returns void or a Promise
Promise.resolve(sdk.start())
  .then(() => console.log("OpenTelemetry SDK started"))
  .catch((err) => console.error("OpenTelemetry SDK error", err));

process.on("SIGTERM", () => {
  Promise.resolve(sdk.shutdown())
    .then(() => console.log("OpenTelemetry SDK shut down"))
    .finally(() => process.exit(0));
});

const app = express();

app.get("/pay", (_req, res) => {
  // simulate small processing delay
  setTimeout(() => res.send("Payment processed"), 50);
});

app.get("/healthz", (_req, res) => res.send("ok"));

app.listen(8080, () => console.log("payments-service listening on 8080"));
```

Crea `node-payments/Dockerfile`:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci --omit=dev
COPY index.js .
ENV SERVICE_NAME=payments-service \
    OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
EXPOSE 8080
CMD ["node", "index.js"]
```

Compilar y publicar imágenes:
```bash
$ docker build -t myregistry/orders-service:otel-1.0.0 python-orders
$ docker push myregistry/orders-service:otel-1.0.0
$ docker build -t myregistry/payments-service:otel-1.0.0 node-payments
$ docker push myregistry/payments-service:otel-1.0.0
```

#### Paso 3: Despliegue del Namespace y del OpenTelemetry Collector
Crea `k8s/namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: observability
```
Aplicar y establecer contexto:
```bash
$ kubectl apply -f k8s/namespace.yaml
$ kubectl config set-context --current --namespace=observability
```

Crea `k8s/otel-collector-cm.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: observability
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          http:
          grpc:
    processors:
      batch: {}
    exporters:
      # Send data to Jaeger using OTLP gRPC
      otlp:
        endpoint: jaeger-collector.observability.svc.cluster.local:4317
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlp]
```

Crea `k8s/otel-collector-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: observability
  labels:
    app: otel-collector
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:0.103.1
          args: ["--config=/etc/otel/config.yaml"]
          ports:
            - name: otlp-grpc
              containerPort: 4317
            - name: otlp-http
              containerPort: 4318
          volumeMounts:
            - name: otel-config
              mountPath: /etc/otel
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
      volumes:
        - name: otel-config
          configMap:
            name: otel-collector-config
```

Crea `k8s/otel-collector-svc.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: observability
  labels:
    app: otel-collector
spec:
  selector:
    app: otel-collector
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
```

Aplicar y verificar:
```bash
$ kubectl apply -f k8s/otel-collector-cm.yaml
$ kubectl apply -f k8s/otel-collector-deployment.yaml
$ kubectl rollout status deploy/otel-collector
$ kubectl apply -f k8s/otel-collector-svc.yaml
$ kubectl logs deploy/otel-collector
```

#### Paso 4: Despliegue de Jaeger (Visualización de trazas)
Crea `k8s/jaeger-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: observability
  labels:
    app: jaeger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.58
          args: ["--collector.otlp.enabled=true"]
          ports:
            - name: ui
              containerPort: 16686 # Web UI
            - name: jaeger-http
              containerPort: 14268 # HTTP ingest
            - name: jaeger-grpc
              containerPort: 14250 # gRPC ingest
            - name: otlp-grpc
              containerPort: 4317 # OTLP gRPC
            - name: otlp-http
              containerPort: 4318 # OTLP HTTP
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

Crea `k8s/jaeger-query-svc.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: observability
  labels:
    app: jaeger
spec:
  selector:
    app: jaeger
  ports:
    - name: ui
      port: 16686
      targetPort: 16686
  type: ClusterIP
```

Crea `k8s/jaeger-collector-svc.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
  namespace: observability
  labels:
    app: jaeger
spec:
  selector:
    app: jaeger
  ports:
    - name: jaeger-grpc
      port: 14250
      targetPort: 14250
    - name: jaeger-http
      port: 14268
      targetPort: 14268
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
  type: ClusterIP
```

Aplicar:
```bash
$ kubectl apply -f k8s/jaeger-deployment.yaml
$ kubectl rollout status deploy/jaeger
$ kubectl apply -f k8s/jaeger-collector-svc.yaml
$ kubectl apply -f k8s/jaeger-query-svc.yaml
$ kubectl logs deploy/jaeger
```

#### Paso 5: Despliegue de los servicios Orders y Payments
Crea `k8s/orders-deployment.yaml` y `k8s/orders-svc.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
  namespace: observability
  labels:
    app: orders-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: orders-service
  template:
    metadata:
      labels:
        app: orders-service
    spec:
      containers:
        - name: orders
          image: myregistry/orders-service:otel-1.0.0
          imagePullPolicy: IfNotPresent
          env:
            - name: SERVICE_NAME
              value: "orders-service"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.observability.svc.cluster.local:4318"
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 300m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: orders-service
  namespace: observability
  labels:
    app: orders-service
spec:
  selector:
    app: orders-service
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

Crea `k8s/payments-deployment.yaml` y `k8s/payments-svc.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-service
  namespace: observability
  labels:
    app: payments-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: payments-service
  template:
    metadata:
      labels:
        app: payments-service
    spec:
      containers:
        - name: payments
          image: myregistry/payments-service:otel-1.0.0
          imagePullPolicy: IfNotPresent
          env:
            - name: SERVICE_NAME
              value: "payments-service"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.observability.svc.cluster.local:4318"
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 300m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: payments-service
  namespace: observability
  labels:
    app: payments-service
spec:
  selector:
    app: payments-service
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

Aplicar y desplegar:
```bash
$ kubectl apply -f k8s/orders-deployment.yaml
$ kubectl apply -f k8s/orders-svc.yaml
$ kubectl apply -f k8s/payments-deployment.yaml
$ kubectl apply -f k8s/payments-svc.yaml
$ kubectl rollout status deploy/orders-service
$ kubectl rollout status deploy/payments-service
```

#### Paso 6: Generación de tráfico y consulta de trazas en Jaeger
1. Abrir port-forward hacia `orders-service` y hacia Jaeger:
   ```bash
   $ kubectl port-forward svc/orders-service 18080:8080
   $ kubectl port-forward svc/jaeger-query 16686:16686
   ```

2. Generar peticiones HTTP de prueba:
   ```bash
   for i in $(seq 1 20); do
     curl -s -X POST http://localhost:18080/order \
       -H 'Content-Type: application/json' \
       -d "{\"orderId\":\"M-$i\"}" >/dev/null
     sleep 0.2
   done
   ```

3. Abrir `http://localhost:16686` en el navegador, seleccionar `Service: orders-service`, operación `POST /order` y pulsar **Find Traces** para examinar la descomposición temporal de cada llamada.

---

### Recolección y visualización de métricas con Prometheus y Grafana

Mientras que las trazas explican el recorrido de una petición individual, las **métricas** revelan el estado agregado y el comportamiento del sistema a lo largo del tiempo.

- **Prometheus**: Base de datos de series temporales que extrae (*scrapes*) métricas periódicamente de los endpoints HTTP `/metrics` de cada servicio.
- **Grafana**: Plataforma de visualización y análisis de datos que se conecta a Prometheus como origen de datos (*data source*).

#### Instrumentación de métricas en Python y Node.js
Copia el directorio a `prometheus-grafana-demo`:
```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-18
$ cp -rf ./otel-demo ./prometheus-grafana-demo
$ cd prometheus-grafana-demo
```

1. **Añadir cliente Prometheus a `python-orders`**:
   En `python-orders/requirements.txt`, añade:
   ```
   prometheus_client==0.20.0
   ```
   En `python-orders/app.py`:
   ```python
   from flask import Flask, request, jsonify
   import requests
   import os

   # --- OpenTelemetry imports
   from opentelemetry import trace
   from opentelemetry.instrumentation.flask import (
       FlaskInstrumentor,
   )
   from opentelemetry.instrumentation.requests import (
       RequestsInstrumentor,
   )
   from opentelemetry.sdk.resources import (
       SERVICE_NAME,
       Resource,
   )
   from opentelemetry.sdk.trace import TracerProvider
   from opentelemetry.sdk.trace.export import (
       BatchSpanProcessor,
   )
   from opentelemetry.exporter.otlp.proto.http.trace_exporter import (
       OTLPSpanExporter,
   )

   # --- Metrics
   from prometheus_client import (
       Counter,
       Histogram,
       generate_latest,
       CONTENT_TYPE_LATEST,
   )

   SERVICE = os.getenv("SERVICE_NAME", "orders-service")
   OTEL_ENDPOINT = os.getenv(
       "OTEL_EXPORTER_OTLP_ENDPOINT",
       "http://otel-collector:4318",
   )

   # --- Prometheus metric objects
   REQS = Counter(
       "orders_requests_total",
       "Total /order requests",
   )
   ERRS = Counter(
       "orders_errors_total",
       "Total /order errors",
   )
   LAT = Histogram(
       "orders_request_seconds",
       "Latency of /order",
   )

   # --- OTel tracer
   trace.set_tracer_provider(
       TracerProvider(
           resource=Resource.create({SERVICE_NAME: SERVICE})
       )
   )
   tracer_provider = trace.get_tracer_provider()
   tracer_provider.add_span_processor(
       BatchSpanProcessor(
           OTLPSpanExporter(
               endpoint=f"{OTEL_ENDPOINT}/v1/traces"
           )
       )
   )

   app = Flask(__name__)
   FlaskInstrumentor().instrument_app(app)
   RequestsInstrumentor().instrument()

   @app.route("/order", methods=["POST"])
   @LAT.time() # measure latency of handler
   def order():
       REQS.inc()
       payload = request.get_json(
           force=True,
           silent=True,
       ) or {}
       try:
           resp = requests.get(
               "http://payments-service:8080/pay",
               timeout=2,
           )
           pay_status = {
               "payments_status": resp.text,
               "payments_code": resp.status_code,
           }
       except Exception as ex:
           ERRS.inc()
           pay_status = {
               "payments_status": f"error: {ex}",
               "payments_code": 500,
           }
       return jsonify(
           {
               "service": SERVICE,
               "message": "Order received",
               "payload": payload,
               "downstream": pay_status,
           }
       )

   @app.route("/metrics")
   def metrics():
       data = generate_latest()
       return data, 200, {
           "Content-Type": CONTENT_TYPE_LATEST
       }

   @app.route("/healthz")
   def health():
       return "ok"

   if __name__ == "__main__":
       app.run(host="0.0.0.0", port=8080)
   ```

2. **Añadir cliente Prometheus a `node-payments`**:
   En `node-payments/package.json`, añade `"prom-client": "^15.1.3"`.
   En `node-payments/index.js`:
   ```javascript
   const express = require("express");

   // --- OpenTelemetry imports
   const { NodeSDK } = require("@opentelemetry/sdk-node");
   const { OTLPTraceExporter } = require("@opentelemetry/exporter-trace-otlp-http");
   const { getNodeAutoInstrumentations } = require("@opentelemetry/auto-instrumentations-node");
   const { Resource } = require("@opentelemetry/resources");
   const { SemanticResourceAttributes } = require("@opentelemetry/semantic-conventions");

   // --- Metrics
   const client = require("prom-client");
   const REGISTRY = new client.Registry();
   client.collectDefaultMetrics({ register: REGISTRY });

   const REQS = new client.Counter({
     name: "payments_requests_total",
     help: "Total /pay requests",
     registers: [REGISTRY],
   });

   const LAT = new client.Histogram({
     name: "payments_request_seconds",
     help: "Latency of /pay",
     buckets: [0.025, 0.05, 0.1, 0.25, 0.5, 1, 2],
     registers: [REGISTRY],
   });

   const SERVICE_NAME = process.env.SERVICE_NAME || "payments-service";
   const OTEL_ENDPOINT = process.env.OTEL_EXPORTER_OTLP_ENDPOINT || "http://otel-collector:4318";

   const sdk = new NodeSDK({
     traceExporter: new OTLPTraceExporter({
       url: `${OTEL_ENDPOINT}/v1/traces`,
     }),
     resource: new Resource({
       [SemanticResourceAttributes.SERVICE_NAME]: SERVICE_NAME,
     }),
     instrumentations: [getNodeAutoInstrumentations()],
   });

   Promise.resolve(sdk.start())
     .then(() => console.log("OpenTelemetry SDK started"))
     .catch((err) => console.error("OpenTelemetry SDK error", err));

   process.on("SIGTERM", () => {
     Promise.resolve(sdk.shutdown())
       .then(() => console.log("OpenTelemetry SDK shut down"))
       .finally(() => process.exit(0));
   });

   const app = express();

   app.get("/pay", (_req, res) => {
     const end = LAT.startTimer();
     REQS.inc();
     setTimeout(() => {
       end();
       res.send("Payment processed");
     }, 50);
   });

   // Expose Prometheus metrics
   app.get("/metrics", async (_req, res) => {
     try {
       res.set("Content-Type", REGISTRY.contentType);
       res.end(await REGISTRY.metrics());
     } catch (e) {
       res.status(500).end(e.toString());
     }
   });

   app.get("/healthz", (_req, res) => res.send("ok"));

   app.listen(8080, () => console.log("payments-service listening on 8080"));
   ```

3. **Recompilar imágenes `otel-1.1.0`**:
   ```bash
   $ docker build -t myregistry/orders-service:otel-1.1.0 ./python-orders
   $ docker build -t myregistry/payments-service:otel-1.1.0 ./node-payments
   $ docker push myregistry/orders-service:otel-1.1.0
   $ docker push myregistry/payments-service:otel-1.1.0
   ```

#### Configuración y despliegue de Prometheus (`k8s/prometheus-*.yaml`)
Crea `k8s/prometheus-cm.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: observability
data:
  prometheus.yml: |
    global:
      scrape_interval: 5s
    scrape_configs:
      - job_name: "prometheus"
        static_configs:
          - targets: ["localhost:9090"]
      - job_name: "otel-collector"
        static_configs:
          - targets: ["otel-collector.observability.svc.cluster.local:8888"]
      - job_name: "orders-service"
        kubernetes_sd_configs:
          - role: endpoints
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_name]
            regex: orders-service
            action: keep
      - job_name: "payments-service"
        kubernetes_sd_configs:
          - role: endpoints
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_name]
            regex: payments-service
            action: keep
```

Crea `k8s/prometheus-deployment.yaml` y `k8s/prometheus-svc.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:v2.54.1
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config
              mountPath: /etc/prometheus
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
      volumes:
        - name: prometheus-config
          configMap:
            name: prometheus-config
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: observability
spec:
  selector:
    app: prometheus
  ports:
    - port: 9090
      targetPort: 9090
```

#### Despliegue de Grafana con Datasource aprovisionado
Crea `k8s/grafana-datasources.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: observability
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
      - uid: prometheus
        name: Prometheus
        type: prometheus
        access: proxy
        url: http://prometheus:9090
        isDefault: true
        editable: true
```

Crea `k8s/grafana-deployment.yaml` y `k8s/grafana-svc.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:11.2.0
          ports:
            - containerPort: 3000
          env:
            - name: GF_SECURITY_ADMIN_USER
              value: "admin"
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: "admin"
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: observability
spec:
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
```

Aplicar todos los recursos:
```bash
$ kubectl apply -f k8s/prometheus-cm.yaml
$ kubectl apply -f k8s/prometheus-deployment.yaml
$ kubectl apply -f k8s/prometheus-svc.yaml
$ kubectl apply -f k8s/grafana-datasources.yaml
$ kubectl apply -f k8s/grafana-dashboard-cm.yaml
$ kubectl apply -f k8s/grafana-deployment.yaml
$ kubectl apply -f k8s/grafana-svc.yaml
```

Generar carga concurrente:
```bash
$ kubectl port-forward svc/orders-service 18080:8080

seq 1 500 | xargs -I{} -P 10 bash -c \
  'curl -s -X POST http://localhost:18080/order \
  -H "Content-Type: application/json" \
  -d "{\"orderId\":\"burst-{}\",\"amount\":$((RANDOM%250+10))}" >/dev/null'
```
*Figura 18.6: Panel de control de Grafana con métricas de órdenes, latencia p95 y tasas de error*

---

### Definición de alertas y guías de respuesta (*Runbooks*)

Una alerta transforma la telemetría pasiva en una **respuesta humana o automatizada**.

#### Principios para alertas operativas eficaces:
1. **Alertar sobre síntomas de usuario, no sobre causas internas**: A los usuarios les impacta una tasa alta de error HTTP 500 o una latencia superior a 1 segundo, no qué contenedor individual tiene un 85% de CPU.
2. **Evitar la fatiga de alertas (*Alert Fatigue*)**: Demasiadas alertas irrelevantes provocan que el equipo ignore las alarmas críticas reales.
3. **Vincular cada alerta a un Runbook ejecutable**.

#### Reglas de alerta en Prometheus (`alerts.yml`):
```yaml
groups:
  - name: service_alerts
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: rate(orders_errors_total[2m]) / rate(orders_requests_total[2m]) > 0.05
        for: 1m
        labels:
          severity: warning
          service: orders-service
        annotations:
          summary: "High error rate in orders-service"
          description: "Error rate above 5% for more than 1 minute."
      - alert: HighLatency
        expr: histogram_quantile(0.95, sum(rate(orders_request_seconds_bucket[2m])) by (le)) > 1
        for: 1m
        labels:
          severity: critical
          service: orders-service
        annotations:
          summary: "High latency in orders-service"
          description: "p95 latency exceeds 1 s for over a minute."
      - alert: PaymentsLatency
        expr: histogram_quantile(0.95, sum(rate(payments_request_seconds_bucket[2m])) by (le)) > 0.8
        for: 1m
        labels:
          severity: warning
          service: payments-service
        annotations:
          summary: "High latency in payments-service"
          description: "p95 latency exceeds 800 ms for more than 1 minute."
```
*Figura 18.7: Reglas de alerta cargadas y evaluadas en Prometheus*

#### Enrutamiento de alertas con Alertmanager (`alertmanager.yml`):
```yaml
global:
  resolve_timeout: 5m

route:
  receiver: default
  group_by: ['alertname', 'service']
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 2h

receivers:
  - name: default
    email_configs:
      - to: "ops@example.com"
        from: "alertmanager@example.com"
        smarthost: "smtp.example.com:587"
        auth_username: "alertmanager@example.com"
        auth_identity: "alertmanager@example.com"
        auth_password: "your-password"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/your/webhook/url"
        channel: "#alerts"
        title: "{{ .CommonAnnotations.summary }}"
        text: "{{ .CommonAnnotations.description }}"
```

---

### Ejemplo de Runbooks operativos para producción

#### Runbook 1: Alta tasa de errores en `orders-service` (`HighErrorRate`)
- **Resumen**: La tasa de error en `orders-service` superó el 5% durante más de 1 minuto.
- **Impacto en el negocio**: Clientes incapaces de completar pagos o finalizar órdenes de compra.
- **Comprobaciones inmediatas**:
  1. Validar el estado de los Pods: `kubectl get pods -n observability -l app=orders-service`.
  2. Consultar el panel de Grafana para verificar si `payments-service` también reporta caídas o errores.
- **Causas probables**:
  - Despliegue reciente con regresiones de código.
  - Interrupción de red entre `orders-service` y `payments-service`.
  - Caída de base de datos relacional aguas abajo.
- **Pasos de mitigación**:
  1. Revisar los últimos logs: `kubectl logs deploy/orders-service --tail=100`.
  2. Si el problema comenzó tras un despliegue: `kubectl rollout undo deploy/orders-service`.
  3. Si es un fallo transitorio de memoria: `kubectl rollout restart deploy/orders-service`.

#### Runbook 2: Latencia elevada en `payments-service` (`PaymentsLatency`)
- **Resumen**: El percentil 95 (p95) de respuesta en `payments-service` supera los 800 ms.
- **Impacto en el negocio**: Tiempos de espera excesivos en la pasarela y posibles cancelaciones por timeout en el backend de órdenes.
- **Mitigación**:
  1. Escalar horizontalmente el servicio: `kubectl scale deploy/payments-service --replicas=3`.
  2. Verificar tiempos de respuesta de la pasarela de pagos externa.

---

### Comparativa: Prometheus vs Grafana Unified Alerting

| Característica | Prometheus + Alertmanager | Grafana Unified Alerting |
| :--- | :--- | :--- |
| **Definición** | Manifiestos YAML versionados en Git | Visualmente desde los paneles o JSON declarativo |
| **Ámbito** | Global para toda la infraestructura | Orientado a paneles y equipos específicos |
| **Enrutamiento** | Centralizado y desacoplado mediante Alertmanager | Puntos de contacto configurados en Grafana |
| **Caso de uso** | Alertas críticas de producción y guardia (*on-call*) | Alarmas visuales en paneles y pruebas exploratorias |

*Tabla 18.1 – Comparación entre alertas en Prometheus y Grafana*

---

### Procedimientos de resolución de problemas (*Troubleshooting*) en Kubernetes

Cuando un Pod o servicio experimenta anomalías, sigue este flujo metódico de diagnóstico:

```
                  [ INCIDENCIA REPORTADA ]
                              |
                              v
             1. kubectl get deploy,pods -o wide
              (¿Estado CrashLoopBackOff/Pending?)
                              |
              +---------------+---------------+
              |                               |
              v                               v
    2. kubectl describe pod         3. kubectl logs <pod> --previous
    (Eventos de Kubernetes,        (Excepciones y trazas de la app)
     fallo de sondas, OOMKilled)              |
              |                               |
              +---------------+---------------+
                              |
                              v
                4. kubectl exec -it <pod> -- sh
               (Inspección interactiva de red/DNS)
```

Comandos esenciales:
```bash
# Diagnóstico de eventos del Pod
$ kubectl describe pod <pod-name>

# Lectura de logs de la instancia anterior tras reinicio
$ kubectl logs <pod-name> --previous

# Comprobación de conectividad y DNS desde dentro del Pod
$ kubectl exec -it <pod-name> -- sh
```

---

### Resumen

En este capítulo aprendimos:
- A instrumentar microservicios heterogéneos con **OpenTelemetry** en Python y Node.js para habilitar el rastreo distribuido (*tracing*).
- A procesar y exportar trazas hacia **Jaeger** mediante el **OpenTelemetry Collector**.
- A implementar métricas tipo Counter e Histogram y exponerlas a través de endpoints `/metrics` compatibles con **Prometheus**.
- A construir paneles de monitorización visuales e interactivos en **Grafana**.
- A definir reglas de alerta en PromQL orientadas al impacto de usuario y enrutarlas con **Alertmanager**.
- A estructurar **runbooks ejecutables** para mitigar incidentes de manera consistente y profesional.

---

### Preguntas

1. **¿Cuál es el propósito principal de instrumentar servicios con OpenTelemetry?**
2. **¿Cuáles son los tres componentes fundamentales de OpenTelemetry y qué función cumple cada uno?**
3. **¿Cómo contribuye el OpenTelemetry Collector al pipeline de observabilidad?**
4. **¿Por qué exponemos un endpoint `/metrics` en nuestras aplicaciones y cuál es el rol de Prometheus respecto a él?**
5. **¿Qué función cumple una métrica de tipo Histogram y por qué es superior para medir latencias frente a una media simple?**
6. **¿Cómo complementa Grafana a Prometheus en una pila de observabilidad?**
7. **¿Cuáles son las principales diferencias entre las alertas de Prometheus y las de Grafana?**
8. **¿Por qué son indispensables los Runbooks y qué información clave deben contener?**
9. **¿Cuál es la función específica de Alertmanager en el ecosistema de monitorización?**
10. **¿Por qué la revisión periódica y la prueba simulada de alertas son vitales para la preparación operativa?**

---

### Respuestas

1. **Propósito de OpenTelemetry**:  
   Permite a las aplicaciones emitir telemetría estandarizada y neutral respecto al proveedor (trazas, métricas y logs), proporcionando visibilidad detallada sobre cómo fluyen las peticiones entre microservicios.

2. **Tres componentes centrales**:  
   - *Bibliotecas de instrumentación*: Capturan datos dentro del código.  
   - *SDK de OpenTelemetry*: Procesa y serializa la telemetría.  
   - *Collector*: Recibe, transforma y exporta los datos hacia backends de análisis (Jaeger, Prometheus).

3. **Rol del OTel Collector**:  
   Actúa como canalización intermedia centralizada. Desacopla la instrumentación de las aplicaciones respecto a los motores de almacenamiento final, permitiendo filtrar, transformar y distribuir datos en formato OTLP a múltiples destinos sin cambiar el código de los microservicios.

4. **Endpoint `/metrics` y Prometheus**:  
   Expone métricas numéricas agregadas en texto plano estructurado. Prometheus consulta periódicamente (*pull scraping*) este endpoint, almacenando las series temporales para su análisis, graficado y evaluación de alertas.

5. **Métricas tipo Histogram**:  
   Agrupan las observaciones en rangos o cubos (*buckets*). Permiten calcular percentiles y cuantiles exactos (p50, p95, p99), revelando la experiencia real de los usuarios más lentos, lo cual queda enmascarado cuando se usa un promedio aritmético.

6. **Rol de Grafana**:  
   Es la capa de visualización e inteligencia operativa. Consulta los datos de Prometheus y los representa en paneles interactivos en tiempo real para evaluar la salud global del sistema.

7. **Prometheus Alerts frente a Grafana Alerts**:  
   Las reglas de Prometheus se declaran en YAML, se evalúan centralizadamente y se enrutan mediante Alertmanager. Las alertas de Grafana se crean visualmente asociadas a paneles individuales del cuadro de mando.

8. **Importancia y contenido de los Runbooks**:  
   Reducen el tiempo medio de recuperación (*MTTR*) proporcionando instrucciones claras. Deben contener: resumen del problema, impacto en el usuario, comprobaciones inmediatas, causas probables, pasos de mitigación y enlaces a dashboards y trazas.

9. **Función de Alertmanager**:  
   Recibe las alertas generadas por Prometheus, elimina duplicados, agrupa eventos relacionados, gestiona períodos de inhibición/silencio y las enruta a canales externos (Slack, Email, PagerDuty).

10. **Preparación operativa continua**:  
    Los sistemas evolucionan; realizar pruebas de fallos y revisiones periódicas post-incidente garantiza que los umbrales de alerta sigan siendo relevantes, evitando la fatiga de alertas y asegurando que las notificaciones lleguen al personal de guardia adecuado con guías actualizadas.

