# Parte 4: Docker, Kubernetes y la Nube
## Capítulo 19: IA y automatización en DevOps

En el capítulo anterior construimos una canalización de observabilidad completa mediante la instrumentación, monitorización y alerta de una aplicación contenerizada en producción. Esas prácticas nos aportaron visibilidad y control, pero la interpretación de las señales y las acciones correctivas seguían dependiendo de operadores humanos.

En este capítulo daremos el siguiente paso evolutivo: **utilizar la inteligencia artificial (IA) y el aprendizaje automático (*Machine Learning*) para crear sistemas auto-optimizables, predictivos y progresivamente autónomos**. Mientras que el enfoque tradicional de DevOps se centraba en automatizar tareas mediante reglas estáticas (CI/CD, scripts), la frontera moderna consiste en enseñar a la infraestructura a **aprender continuamente de los datos operativos y tomar decisiones por sí misma**.

Exploraremos cómo las técnicas de IA potencian las actividades de DevOps: desde la programación inteligente de compilaciones y el escalado predictivo proactivo hasta el análisis automatizado de causa raíz (*Root-Cause Analysis*). Además, aprenderemos a integrar modelos de machine learning en flujos de trabajo de contenedores y a desplegarlos de forma segura junto con las cargas de trabajo de producción.

---

### Temas tratados en este capítulo:
- ¿Por qué aplicar Inteligencia Artificial en DevOps? (AIOps y MLOps)
- Casos de uso y patrones de diseño de IA en operaciones
- Ecosistema de herramientas y marcos de trabajo: Kubeflow, KServe, Argo Workflows, Prometheus + ML, MLflow y Copilot CI
- Laboratorio práctico A: Construcción de un autoescalador predictivo basado en Machine Learning (scikit-learn y Kubernetes CronJob)
- Laboratorio práctico B: Canalización de reentrenamiento y actualización continua de modelos con Argo Workflows y PVCs

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Comprender la transición desde la automatización reactiva basada en reglas estáticas hacia las operaciones inteligentes basadas en datos (*AIOps*).
- Implementar el bucle de retroalimentación fundamental: **Observar → Aprender → Predecir → Actuar (*Observe → Learn → Predict → Act*)**.
- Entrenar modelos de regresión (*Random Forest*) para predecir la carga de CPU futura basándose en series temporales históricas.
- Desplegar una API de inferencia en contenedores que traduzca las predicciones en recomendaciones de réplicas de Kubernetes.
- Automatizar el ciclo de vida continuo de reentrenamiento (*Continuous Training / Model Refresh*) de modelos de IA utilizando **Argo Workflows** y **CronWorkflows**.

---

### Requisitos técnicos

- Clúster de Kubernetes en ejecución (kind, minikube o clúster en la nube).
- Herramientas instaladas: `kubectl`, `Docker`, `Helm 3`, `jq` y Python 3.10+.
- Dependencias de Python para el pipeline de entrenamiento:
  ```bash
  $ pip install numpy pandas scikit-learn joblib
  ```
- Repositorio del libro clonado localmente:
  [https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-19](https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-19)

---

### ¿Por qué IA para DevOps?

El DevOps tradicional automatiza tareas mecánicas (compilación, pruebas, despliegues) basadas en reglas fijas. La **Inteligencia Artificial introduce la automatización de decisiones operativas**, aprendiendo patrones a partir de los datos y adaptándose a entornos dinámicos.

Tres factores impulsan esta convergencia:
1. **Abundancia de datos**: Las métricas, trazas y registros acumulados constituyen conjuntos de datos ideales para entrenar modelos de IA.
2. **Elasticidad del cómputo**: Orquestadores como Kubernetes permiten programar tareas efímeras de entrenamiento e inferencia bajo demanda.
3. **Madurez de herramientas abiertas**: Marcos como Kubeflow, PyTorch y Argo facilitan la integración nativa de modelos en flujos de contenedores.

Esta evolución da lugar a dos disciplinas clave:
- **AIOps (*Artificial Intelligence for IT Operations*)**: Aplicación de IA para automatizar la monitorización, correlación de alertas y resolución de incidencias.
- **MLOps (*Machine Learning Operations*)**: Aplicación de las disciplinas de DevOps (control de versiones, CI/CD, monitorización) al ciclo de vida del aprendizaje automático.

---

### Casos de uso y patrones de IA en DevOps

| Dominio | Caso de uso de ejemplo | Beneficio operativo |
| :--- | :--- | :--- |
| **Monitorización y Alertas** | Detección de anomalías en latencia o errores con ML en lugar de umbrales estáticos | Reducción drástica de falsos positivos y fatiga de alertas |
| **Respuesta a Incidentes** | Clasificación y agrupación de alertas por similitud con incidencias pasadas | Diagnóstico y triaje acelerados (*MTTR* reducido) |
| **Optimización de CI/CD** | Predicción de duración de builds o detección de pruebas inestables (*flaky tests*) | Priorización inteligente y pipelines más rápidos |
| **Gestión de Capacidad** | Previsión de demanda de recursos y escalado proactivo (*Predictive Autoscaling*) | Ahorro significativo de costes de infraestructura |
| **Estrategia de Release** | Automatización de promociones en despliegues Canary/Blue-Green según métricas en vivo | Despliegues continuos más seguros y sin riesgo |

*Tabla 19.1 – Casos de uso de IA en el ciclo de vida de DevOps*

#### El bucle de retroalimentación de AIOps:
```
Observar (Métricas/Trazas) ---> Aprender (Entrenar Modelo) ---> Predecir (Estimar Demanda) ---> Actuar (Autoescalar/Sanar) ---> (Repetir)
```

---

### Ecosistema de herramientas y marcos de trabajo

```
+-----------------------------------------------------------------------+
|                    PILA TECNOLÓGICA DE AIOps / MLOps                  |
|                                                                       |
|  [ Experimentación y Gobierno ]  PyCaret / MLflow                     |
|  [ Orquestación de Pipelines  ]  Kubeflow / Argo Workflows / Events   |
|  [ Inferencia y Despliegue   ]  KServe / Knative                      |
|  [ Telemetría y Predicción   ]  Prometheus / scikit-learn / Prophet  |
|  [ Asistencia al Ingeniero   ]  Ansible Lightspeed / GitHub Copilot   |
+-----------------------------------------------------------------------+
```

- **Kubeflow**: Plataforma nativa de Kubernetes para orquestar flujos completos de ML (preprocesamiento, entrenamiento, validación y despliegue distribuido).
- **KServe**: Servidor declarativo de modelos de IA con soporte para autoescalado a cero (*scale-to-zero*) mediante Knative.
- **Argo Workflows y Argo Events**: Motor de automatización y canalización basada en eventos nativo de Kubernetes.
- **Prometheus + scikit-learn / Prophet**: Combinación de series temporales con algoritmos predictivos para modelar estacionalidades horarias y semanales.
- **PyCaret y MLflow**: Seguimiento de experimentos, registro versionado de artefactos (*Model Registry*) y gobernanza de modelos.
- **Ansible Lightspeed y GitHub Copilot CI**: Asistentes generativos que facilitan la autoría y validación de manifiestos y pipelines.

---

### Laboratorio A: Construcción de un autoescalador predictivo con Machine Learning

En este laboratorio construiremos un autoescalador que anticipa la carga de CPU y ajusta la capacidad de la aplicación **antes** de que lleguen los picos de tráfico, superando las limitaciones reactivas del HPA tradicional.

```
+-------------------------------------------------------------------------+
|                  ARQUITECTURA DEL AUTOESCALADOR PREDICTIVO              |
|                                                                         |
|  +--------------------+         HTTP GET          +------------------+  |
|  | Autoscaler         | ------------------------> | Predictor API    |  |
|  | (Kubernetes        |   /predict?hour=14        | (Flask + Model)  |  |
|  |  CronJob c/2 min)  | <------------------------ |                  |  |
|  +--------------------+   {"recommended": 9}     +------------------+  |
|           |                                                             |
|           | kubectl scale deployment sample-app --replicas=9            |
|           v                                                             |
|  +-------------------------------------------------------------------+  |
|  | Target App (NGINX Deployment): [Pod] [Pod] [Pod] [Pod] [Pod]...  |  |
|  +-------------------------------------------------------------------+  |
+-------------------------------------------------------------------------+
```
*Figura 19.1: Arquitectura del sistema del Laboratorio A*

#### Paso 1: Generación de datos sintéticos de series temporales
Ejecutar el generador:
```bash
$ python predictor/generate_data.py
```

Lógica de generación sinusoidal con ruido gaussiano:
```python
# Sinusoidal base: trough at ~04:00, peak at ~14:00
phase_shift = 2 * np.pi * (240 / NUM_MINUTES)
base = np.sin(2 * np.pi * minutes / NUM_MINUTES - phase_shift)

# Scale from [-1, 1] to [BASE_LOW, BASE_HIGH]
amplitude = (BASE_HIGH - BASE_LOW) / 2
midpoint = (BASE_HIGH + BASE_LOW) / 2
cpu = midpoint + amplitude * base
```
Salida esperada:
```
Generated 1440 data points -> predictor/data/cpu_load.csv
cpu_percent min=13.89 max=100.00 mean=49.73
```

#### Paso 2: Entrenamiento del modelo Random Forest
Ejecutar el script de entrenamiento:
```bash
$ python predictor/train_model.py
```

Ingeniería de características con media móvil (*rolling average*):
```python
ROLLING_WINDOW = 10  # minutes
df["rolling_avg"] = (
    df["cpu_percent"]
    .rolling(window=ROLLING_WINDOW, min_periods=1)
    .mean()
)
```
Salida esperada:
```
Loaded 1440 rows from predictor/data/cpu_load.csv
Evaluation on test set (288 samples):
  MAE = 3.75
  R²  = 0.9500
Feature importances:
  hour        0.0072
  minute      0.0030
  day_of_week 0.0000
  rolling_avg 0.9898
Model saved -> predictor/model.pkl
```

#### Paso 3: Construcción de la API de inferencia (Predictor)
La API mapea el porcentaje de CPU predicho a un número recomendado de réplicas:
```python
def _cpu_to_replicas(cpu: float) -> int:
    """Linearly map a predicted CPU% to a replica count."""
    if cpu <= CPU_LOW:
        return MIN_REPLICAS
    if cpu >= CPU_HIGH:
        return MAX_REPLICAS
    ratio = (cpu - CPU_LOW) / (CPU_HIGH - CPU_LOW)
    return max(MIN_REPLICAS, min(MAX_REPLICAS, round(
        MIN_REPLICAS + ratio * (MAX_REPLICAS - MIN_REPLICAS)
    )))
```

`predictor/Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
COPY model.pkl .
EXPOSE 8080
CMD ["gunicorn", "app:app", "--bind", "0.0.0.0:8080"]
```

Compilar imagen del predictor:
```bash
$ docker build -t predictor:1.0 ./predictor
```

#### Paso 4: Construcción del Autoscaler en contenedor
Lógica en Bash (`autoscaler/scale.sh`):
```bash
# 1. Query the predictor
RESPONSE=$(curl -s --fail --max-time "${CURL_TIMEOUT}" "${PREDICTOR_URL}")

# 2. Parse the recommendation
RECOMMENDED=$(echo "${RESPONSE}" | jq -r '.recommended_replicas')

# 3. Scale if the current count differs
CURRENT=$(kubectl get deployment "${DEPLOYMENT}" -n "${NAMESPACE}" \
  -o jsonpath='{.spec.replicas}')

if [ "${CURRENT}" -ne "${RECOMMENDED}" ]; then
  kubectl scale deployment "${DEPLOYMENT}" \
    -n "${NAMESPACE}" --replicas="${RECOMMENDED}"
fi
```

`autoscaler/Dockerfile`:
```dockerfile
FROM bitnami/kubectl:latest
USER root
COPY scale.sh /scripts/scale.sh
RUN chmod +x /scripts/scale.sh
USER 1001
ENTRYPOINT ["bash", "/scripts/scale.sh"]
```

Compilar imagen del autoscaler:
```bash
$ docker build -t autoscaler:1.0 ./autoscaler
```

#### Paso 5: Creación del clúster kind y despliegue
```bash
# Crear clúster kind y precargar imágenes
$ kind create cluster --config kind-config.yaml --name lab-a
$ kind load docker-image predictor:1.0 --name lab-a
$ kind load docker-image autoscaler:1.0 --name lab-a
$ kubectl cluster-info --context kind-lab-a

# Desplegar aplicación objetivo
$ kubectl apply -f sample-app/deployment.yaml
$ kubectl rollout status deployment/sample-app --timeout=60s

# Desplegar Predictor API
$ kubectl apply -f predictor/deployment.yaml
$ kubectl rollout status deployment/predictor --timeout=120s

# Desplegar RBAC y CronJob del Autoscaler
$ kubectl apply -f autoscaler/rbac.yaml
$ kubectl apply -f autoscaler/cronjob.yaml
```

Reglas RBAC requeridas (`autoscaler/rbac.yaml`):
```yaml
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "deployments/scale"]
    verbs: ["get", "list", "patch", "update"]
```

#### Paso 6: Observación y validación del autoescalado
Comprobar predicciones para diferentes horas:
```bash
$ kubectl port-forward svc/predictor 9091:8080 &
$ curl "http://localhost:9091/predict?hour=14&minute=30"
# {"predicted_cpu": 74.52, "recommended_replicas": 9, "timestamp": "2025-06-01T12:00:00Z"}

$ curl "http://localhost:9091/predict?hour=4&minute=0"
# {"predicted_cpu": 20.0, "recommended_replicas": 1, ...}
$ kill %1
```

Ejecutar script de demostración de 24 horas:
```bash
$ bash scripts/demo.sh
```
Salida esperada:
```
Predicted CPU load and recommended replicas for each hour:
  Hour    Predicted CPU   Replicas   Visual
  ------- --------------- ---------- --------------------
  00:00        28.5%          2      #####
  01:00        23.1%          1      ####
  02:00        19.8%          1      ###
  03:00        17.2%          1      ###
  04:00        20.0%          1      ####
  ...
  12:00        68.3%          8      #############
  13:00        73.8%          9      ##############
  14:00        75.2%          9      ###############
  15:00        72.1%          8      ##############
  ...
  22:00        35.5%          3      #######
  23:00        31.2%          2      ######
```

Limpieza del laboratorio A:
```bash
$ kind delete cluster --name lab-a
```

---

### Comparativa: Del laboratorio a un entorno de producción

| Componente del Lab A | Equivalente en Producción |
| :--- | :--- |
| **Datos sintéticos en CSV** | Canalización de métricas reales (Prometheus, Datadog, CloudWatch) |
| **Entrenamiento offline local** | Pipelines automatizados de entrenamiento (Kubeflow, MLflow, AWS SageMaker) |
| **Modelo `.pkl` horneado en imagen** | Registro central de modelos versionados (MLflow Model Registry, S3, GCS) |
| **API REST en Flask** | Endpoints gestionados de inferencia o KServe sobre Kubernetes |
| **CronJob cada 2 minutos** | Controlador personalizado de Kubernetes u operador reactivo por eventos |
| **Comando `kubectl scale` en Bash** | Integración nativa con Horizontal Pod Autoscaler (HPA) vía *Custom Metrics API* |

*Tabla 19.2 – Mapeo de componentes del prototipo a producción real*

---

### Laboratorio B: Reentrenamiento continuo con Argo Workflows

En este laboratorio automatizaremos el ciclo de actualización continua del modelo (*Continuous Model Refresh*) para combatir el desfase del modelo (*model drift*) mediante **Argo Workflows**.

Ciclo del flujo:
```
Entrenar Modelo (Job) ---> Publicar en PVC Compartido ---> Reiniciar Despliegue API ---> Servir Nueva Versión
```

#### Paso 1: Creación de Namespaces (`k8s/namespaces.yaml`)
```yaml
apiVersion: v1
kind: Namespace
metadata: { name: aiops }
---
apiVersion: v1
kind: Namespace
metadata: { name: argo }
```
Aplicar:
```bash
$ kubectl apply -f k8s/namespaces.yaml
```

#### Paso 2: Script de reentrenamiento y construcción de imagen
`trainer/app/train.py`:
```python
import math, random, joblib, pandas as pd
from datetime import datetime, timezone, timedelta
from sklearn.ensemble import RandomForestRegressor

now = datetime.now(timezone.utc).replace(second=0, microsecond=0)
start = now - timedelta(hours=24)

ts, cpu = [], []
t = start
while t <= now:
    m = t.hour*60 + t.minute
    base = 0.6 + 0.4*math.sin(2*math.pi*m/(24*60))
    val = max(0.05, base*(0.9 + 0.2*random.random()))
    ts.append(t)
    cpu.append(val)
    t += timedelta(minutes=5)

df = pd.DataFrame({"ts": ts, "cpu": cpu})
df["minute_of_day"] = df["ts"].dt.hour*60 + df["ts"].dt.minute
df["cpu_next"] = df["cpu"].shift(-6)
train = df.dropna()

X = train[["minute_of_day","cpu"]]
y = train["cpu_next"]
m = RandomForestRegressor(n_estimators=80, random_state=7).fit(X, y)
joblib.dump(m, "/tmp/cpu_forecast.joblib")
print("Saved /tmp/cpu_forecast.joblib")
```

`trainer/Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app/train.py /app/train.py
RUN pip install --no-cache-dir pandas scikit-learn joblib
CMD ["python", "/app/train.py"]
```

Compilar y publicar imagen:
```bash
$ docker image build -t YOUR_REGISTRY/trainer:1.0 trainer
$ docker image push YOUR_REGISTRY/trainer:1.0
```

#### Paso 3: Almacenamiento compartido para modelos (`k8s/pvc-model.yaml`)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-artifacts
  namespace: aiops
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
```
Aplicar e inicializar con un Job temporal:
```bash
$ kubectl apply -f k8s/pvc-model.yaml
$ kubectl apply -f k8s/init-model-job.yaml
$ kubectl -n aiops wait --for=condition=complete job/init-model
$ kubectl -n aiops delete job/init-model
```

#### Paso 4: Despliegue de la API que consume el modelo del PVC (`k8s/ai-autoscaler-api.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-autoscaler-api
  namespace: aiops
  labels:
    app: ai-autoscaler-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ai-autoscaler-api
  template:
    metadata:
      labels:
        app: ai-autoscaler-api
    spec:
      volumes:
        - name: model-vol
          persistentVolumeClaim:
            claimName: model-artifacts
      containers:
        - name: api
          image: YOUR_REPOSITORY/ai-autoscaler-api:latest
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: MODEL_PATH
              value: /shared/cpu_forecast.joblib
          volumeMounts:
            - name: model-vol
              mountPath: /shared
              readOnly: true
---
apiVersion: v1
kind: Service
metadata:
  name: ai-autoscaler-api
  namespace: aiops
spec:
  selector:
    app: ai-autoscaler-api
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```
Aplicar:
```bash
$ kubectl apply -f k8s/ai-autoscaler-api.yaml
$ kubectl -n aiops rollout status deploy/ai-autoscaler-api
```

#### Paso 5: Instalación de Argo Workflows y definición de plantillas
1. Instalar el controlador de Argo:
   ```bash
   $ kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.8/install.yaml
   $ kubectl -n argo rollout status deploy/workflow-controller
   ```

2. Plantilla de reentrenamiento (`k8s/wft-retrain-img.yaml`):
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: WorkflowTemplate
   metadata:
     name: retrain-model-img
     namespace: aiops
   spec:
     templates:
       - name: run
         container:
           image: YOUR_REGISTRY/trainer:1.0
           command: ["/bin/sh","-c"]
           args:
             - |
               echo "Training model..."
               python /app/train.py
               echo "Copying model to shared volume..."
               cp /tmp/cpu_forecast.joblib \
                  /shared/cpu_forecast.joblib
               echo "Done."
           volumeMounts:
             - name: model-vol
               mountPath: /shared
         volumes:
           - name: model-vol
             persistentVolumeClaim:
               claimName: model-artifacts
   ```

3. Plantilla de reinicio controlado (`k8s/wft-restart.yaml`):
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: WorkflowTemplate
   metadata:
     name: restart-serving
     namespace: aiops
   spec:
     templates:
       - name: run
         container:
           image: bitnami/kubectl:latest
           command:
             - kubectl
             - -n
             - aiops
             - rollout
             - restart
             - deploy/ai-autoscaler-api
   ```
   Aplicar plantillas:
   ```bash
   $ kubectl apply -f k8s/wft-retrain-img.yaml
   $ kubectl apply -f k8s/wft-restart.yaml
   ```

#### Paso 6: Automatización periódica con CronWorkflow
Definir la ejecución programada cada hora (`k8s/cwf-model-refresh-hourly.yaml`):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata:
  name: model-refresh-hourly
  namespace: aiops
spec:
  schedule: "0 * * * *" # every hour at :00
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 600
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 2
  workflowSpec:
    entrypoint: pipeline
    templates:
      - name: pipeline
        steps:
          - - name: retrain
              templateRef:
                name: retrain-model-img
                template: run
          - - name: restart-serving
              templateRef:
                name: restart-serving
                template: run
```
Aplicar y verificar el pipeline autónomo:
```bash
$ kubectl apply -f k8s/cwf-model-refresh-hourly.yaml
$ kubectl -n aiops get cronworkflow
$ kubectl -n aiops get wf --sort-by=.metadata.creationTimestamp
```

---

### Resumen

En este capítulo aprendimos:
- Cómo la **Inteligencia Artificial y el Machine Learning transforman el DevOps tradicional** en operaciones predictivas y auto-optimizables (*AIOps*).
- A construir un **autoescalador predictivo** que anticipa la demanda de CPU basándose en patrones históricos entrenados con algoritmos de Random Forest.
- A orquestar un pipeline de **reentrenamiento y actualización continua de modelos** utilizando **Argo Workflows**, PersistentVolumeClaims y CronWorkflows en Kubernetes.
- La importancia del bucle continuo: **Recolectar → Entrenar → Publicar → Desplegar → Observar → Repetir**.

---

### Preguntas

1. **¿Qué problema fundamental busca resolver AIOps en las operaciones de TI modernas?**
2. **En el Laboratorio A, ¿por qué utilizamos datos sintéticos en lugar de conectar directamente una instancia de Prometheus en vivo?**
3. **¿Cuáles son los cuatro pasos esenciales que componen el bucle de retroalimentación de operaciones autónomas?**
4. **¿Cómo calcula el autoescalador del Laboratorio A el número recomendado de réplicas de Pods?**
5. **¿Cuál fue el propósito principal de introducir Argo Workflows en el Laboratorio B?**
6. **¿De qué manera simplificó el PersistentVolumeClaim (PVC) la gestión de modelos entre el entrenamiento y la inferencia?**
7. **¿Qué ventaja clave ofrece un `CronWorkflow` de Argo frente a un `CronJob` nativo de Kubernetes?**
8. **¿Cuáles son las diferencias más relevantes entre nuestro laboratorio local y un entorno de producción a gran escala?**
9. **¿Qué riesgos se deben evaluar al implementar automatizaciones auto-adaptativas basadas en IA en producción?**
10. **¿Cómo prepara este capítulo el terreno para los casos de estudio reales del Capítulo 20?**

---

### Respuestas

1. **Objetivo de AIOps**:  
   Aborda la complejidad inmanejable de los sistemas distribuidos a gran escala, utilizando analítica avanzada y ML para correlacionar alertas, predecir necesidades de capacidad y automatizar decisiones operativas complejas.

2. **Uso de datos sintéticos**:  
   Permitió simular patrones de tráfico realistas de 24 horas (ciclos día/noche y picos aleatorios) en un entorno local ligero y reproducible, sin depender de semanas de recopilación de telemetría previa.

3. **Cuatro pasos del bucle**:  
   - *Observar*: Recolectar métricas y señales del sistema.  
   - *Aprender*: Analizar datos y entrenar modelos con patrones históricos.  
   - *Predecir*: Estimar la demanda o detectar anomalías futuras.  
   - *Actuar*: Ejecutar cambios de infraestructura automáticamente (escalado, reinicios, mitigación).

4. **Cálculo de réplicas**:  
   La API recibe la hora actual, consulta el modelo de regresión para estimar el porcentaje de CPU y aplica una función de interpolación lineal acotada entre `MIN_REPLICAS` y `MAX_REPLICAS`.

5. **Función de Argo Workflows**:  
   Orquestar pipelines de múltiples pasos con dependencias estrictas: asegurar que el reentrenamiento finalice con éxito antes de publicar el artefacto y reiniciar el servicio de inferencia.

6. **Rol del PVC como almacén de artefactos**:  
   Actuó como punto de intercambio desacoplado: el contenedor de entrenamiento escribe el archivo `.joblib` en el volumen y el contenedor de la API lo carga en memoria al reiniciar, sin requerir servicios externos de almacenamiento de objetos.

7. **CronWorkflow frente a CronJob**:  
   Un `CronJob` solo ejecuta un único contenedor aislado, mientras que un `CronWorkflow` orquesta flujos complejos de múltiples pasos secuenciales o paralelos con paso de artefactos y control de errores como una unidad atómica.

8. **Diferencias con producción**:  
   En producción se utilizan métricas reales en tiempo real (Prometheus), almacenamiento de modelos versionado (MLflow/S3), servidores de inferencia especializados (KServe) y puertas de validación automática (*canary testing*) antes de promocionar un nuevo modelo.

9. **Riesgos de la IA en operaciones**:  
   Desviación del modelo (*model drift*), oscilaciones erráticas en la asignación de recursos (*flapping*), falta de visibilidad humana y posibles bucles de retroalimentación destructivos. Se mitigan mediante límites estrictos, validación continua y supervisión humana (*human-in-the-loop*).

10. **Conexión con el Capítulo 20**:  
    Proporciona la comprensión técnica de cómo la contenerización, la observabilidad y la automatización inteligente se integran para resolver los retos reales de escalabilidad y fiabilidad analizados en los casos de estudio finales.

