# Parte 4: Docker, Kubernetes y la Nube
## Capítulo 17: Ejecución de una aplicación contenerizada en la nube

En los capítulos anteriores aprendimos a desplegar y operar aplicaciones contenerizadas localmente en clústeres de Kubernetes mediante minikube, kind o Docker Desktop. Aunque estos entornos locales son ideales para el desarrollo y las pruebas, no reflejan la realidad operativa de producción. Los sistemas reales deben ejecutarse en infraestructuras resilientes, elásticas y seguras, con mecanismos profesionales de copia de seguridad, monitorización y cumplimiento normativo.

Montar y mantener un clúster de Kubernetes autogestionado en producción supone un desafío colosal: aprovisionamiento y parcheo de máquinas virtuales, configuración de alta disponibilidad (HA) para `etcd`, balanceadores de carga y cortafuegos de red. Por este motivo, la inmensa mayoría de las organizaciones recurren a los **servicios gestionados de Kubernetes en la nube**.

Los tres principales proveedores de nube pública ofrecen plataformas gestionadas: **Amazon Elastic Kubernetes Service (EKS)**, **Microsoft Azure Kubernetes Service (AKS)** y **Google Kubernetes Engine (GKE)**. En estos entornos, el proveedor asume la responsabilidad total de operar y actualizar el plano de control (*control plane*), permitiendo a los equipos centrarse exclusivamente en el código y sus cargas de trabajo.

En este capítulo, aprenderemos a preparar imágenes multiplataforma (*multi-arch*) con **Docker Buildx** y desplegaremos la aplicación **TaskBoard** en EKS, AKS y GKE utilizando los mismos manifiestos de Kubernetes desarrollados localmente. Finalmente, exploraremos la frontera de los **contenedores serverless** (**AWS Fargate**, **Google Cloud Run** y **Azure Container Apps**), donde incluso la gestión de nodos desaparece por completo.

---

### Temas tratados en este capítulo:
- Preparación y publicación de imágenes multiplataforma con Docker Buildx (ARM64 y AMD64)
- Ventajas y modelo de costes de los servicios gestionados de Kubernetes
- Despliegue de la aplicación TaskBoard en Amazon EKS (AWS)
- Despliegue de la aplicación TaskBoard en Microsoft Azure AKS
- Despliegue de la aplicación TaskBoard en Google GKE y modo Autopilot
- Plataformas de contenedores Serverless (AWS Fargate, Google Cloud Run y Azure Container Apps)

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Evaluar los beneficios y el modelo de responsabilidad compartida de los servicios gestionados de Kubernetes frente a clústeres autogestionados.
- Construir y publicar imágenes de contenedor multiarquitectura compatibles con entornos mixtos (Apple Silicon x86/ARM).
- Aprovisionar clústeres y desplegar aplicaciones completas en Amazon EKS, Azure AKS y Google GKE.
- Comprender el funcionamiento de los Services de tipo `LoadBalancer` que aprovisionan balanceadores de carga en la nube.
- Identificar los casos de uso donde los contenedores serverless ofrecen mayor eficiencia y reducción de costes operativos.

---

### Requisitos técnicos

- Docker Engine / Docker Desktop con soporte para **Docker Buildx**.
- Herramienta CLI `kubectl` instalada y verificada:
  ```bash
  $ kubectl version --client
  ```
- Cuenta en un registro de contenedores (Docker Hub, AWS ECR, Azure ACR o Google Artifact Registry).
- Herramientas CLI de los proveedores de nube:
  - **AWS CLI** y **eksctl** para Amazon EKS.
  - **Azure CLI (`az`)** para Azure AKS.
  - **Google Cloud SDK (`gcloud`)** para Google GKE.

Preparar el espacio de trabajo:
```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir chapter-17 && cd chapter-17
```

> [!WARNING]
> **Costes en la nube**:  
> La creación de clústeres gestionados consume recursos en la nube que pueden generar cargos. Recuerda ejecutar los comandos de limpieza al finalizar cada laboratorio.

---

### Preparación de imágenes multiplataforma con Docker Buildx

Al desarrollar en un ordenador portátil con procesador ARM64 (como Apple Silicon) y desplegar en instancias de nube x86_64 (como instancias `t3.small` en AWS), es necesario compilar imágenes que admitan ambas arquitecturas.

#### 1. Configurar un constructor multiarquitectura:
```bash
# Crear un builder que soporte multi-arch
$ docker buildx create --use --name mbuilder
$ docker buildx inspect --bootstrap
```

#### 2. Compilar y publicar imágenes para `linux/amd64` y `linux/arm64`:
```bash
# Compilar y enviar imágenes multi-arch
$ docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t YOUR_REGISTRY/taskboard-api:1.0.1 \
  -f ./api/Dockerfile \
  --push ./api

$ docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t YOUR_REGISTRY/taskboard-web:1.0.1 \
  -f ./frontend/Dockerfile \
  --push ./frontend
```
*(Sustituye `YOUR_REGISTRY` por tu usuario de Docker Hub o nombre de registro).*

---

### ¿Por qué elegir un servicio gestionado de Kubernetes?

Operar Kubernetes a escala empresarial exige configurar cortafuegos, certificados TLS, replicación de `etcd`, balanceadores de carga y actualizaciones continuas sin caídas.

#### Ventajas clave de un servicio gestionado:
- **Plano de control gestionado**: El proveedor aprovisiona, escala, repara y actualiza los componentes maestros (`kube-apiserver`, `etcd`, controladores).
- **Integración de seguridad e identidad nativa**: Vinculación directa con IAM (AWS IRSA, Azure AD Workload Identity, Google Workload Identity).
- **Escalabilidad elástica**: Autoescalado automático tanto de Pods (HPA) como de nodos del clúster (*Cluster Autoscaler*).
- **Alta disponibilidad multizona integrada**: El plano de control se distribuye entre múltiples zonas de disponibilidad (*Availability Zones*).

#### Modelo de responsabilidad compartida:
- **Responsabilidad del proveedor**: Plano de control, disponibilidad del API Server, base de datos `etcd` y parches del sistema de control.
- **Responsabilidad del usuario**: Nodos de trabajo (*worker nodes*), cargas de trabajo desplegadas, políticas de red, RBAC, gestión de secretos y vulnerabilidades en las imágenes.

---

### Ejecución de una aplicación en Amazon EKS (AWS)

#### Paso 1: Verificación de requisitos
Verificar credenciales de AWS y versiones de herramientas:
```bash
$ aws sts get-caller-identity --profile <your-profile>
$ eksctl version
$ kubectl version --client
```

#### Paso 2: Creación del clúster con `eksctl`
```bash
$ eksctl create cluster \
  --profile <your-profile> \
  --name taskboard-cluster \
  --region eu-central-1 \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```
*(El aprovisionamiento tarda entre 10 y 15 minutos. `eksctl` actualiza automáticamente el archivo `~/.kube/config`).*

Verificar nodos:
```bash
$ kubectl get nodes
```

#### Paso 3: Namespace, ConfigMap y Secret
```bash
$ kubectl create namespace taskboard
$ kubectl config set-context --current --namespace=taskboard
```

Crear `config.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: taskboard-config
data:
  POSTGRES_DB: taskboard
  POSTGRES_USER: taskuser
  POSTGRES_PASSWORD: taskpass
  API_HTTP_PORT: "8080"
  WEB_HTTP_PORT: "80"
```
Aplicar y crear el secreto:
```bash
$ kubectl apply -f config.yaml
$ kubectl create secret generic taskboard-db-secret \
  --from-literal=POSTGRES_USER=taskuser \
  --from-literal=POSTGRES_PASSWORD=taskpass
```

#### Paso 4: Despliegue de PostgreSQL con almacenamiento EBS (AWS CSI Driver)
1. Instalar el controlador CSI para volúmenes Amazon EBS:
   ```bash
   export AWS_PROFILE=eks-lab
   export AWS_REGION=eu-central-1
   CLUSTER=demo-cluster

   $ eksctl utils associate-iam-oidc-provider \
     --cluster $CLUSTER --approve

   $ eksctl create iamserviceaccount \
     --profile <your-profile> \
     --cluster $CLUSTER \
     --namespace kube-system \
     --name ebs-csi-controller-sa \
     --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
     --approve \
     --override-existing-serviceaccounts

   $ eksctl create addon \
     --profile <your-profile> \
     --cluster $CLUSTER \
     --name aws-ebs-csi-driver \
     --force
   ```

2. Crear `postgres-aws.yaml` (StatefulSet con clase de almacenamiento `gp2`):
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: postgres
   spec:
     ports:
       - port: 5432
         name: postgres
     clusterIP: None
     selector:
       app: postgres
   ---
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: postgres
   spec:
     serviceName: postgres
     replicas: 1
     selector:
       matchLabels:
         app: postgres
     template:
       metadata:
         labels:
           app: postgres
       spec:
         securityContext:
           fsGroup: 999 # pg group in the official image
         containers:
           - name: postgres
             image: postgres:15
             ports:
               - containerPort: 5432
                 name: postgres
             env:
               - name: PGDATA
                 value: /var/lib/postgresql/data/pgdata
               - name: POSTGRES_DB
                 valueFrom:
                   configMapKeyRef:
                     name: taskboard-config
                     key: POSTGRES_DB
               - name: POSTGRES_USER
                 valueFrom:
                   secretKeyRef:
                     name: taskboard-db-secret
                     key: POSTGRES_USER
               - name: POSTGRES_PASSWORD
                 valueFrom:
                   secretKeyRef:
                     name: taskboard-db-secret
                     key: POSTGRES_PASSWORD
             volumeMounts:
               - name: postgres-data
                 mountPath: /var/lib/postgresql/data
         volumes: []
     volumeClaimTemplates:
       - metadata:
           name: postgres-data
         spec:
           accessModes: ["ReadWriteOnce"]
           storageClassName: gp2 # AWS EBS gp2 storage class
           resources:
             requests:
               storage: 2Gi
   ```
   Aplicar:
   ```bash
   $ kubectl apply -f postgres-aws.yaml
   $ kubectl get pods -l app=postgres
   $ kubectl get pvc
   ```

#### Paso 5: Despliegue de la API (`api.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskboard-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: taskboard-api
  template:
    metadata:
      labels:
        app: taskboard-api
    spec:
      containers:
        - name: api
          image: YOUR_REGISTRY/taskboard-api:1.0.1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: postgres
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: taskboard-config
                  key: POSTGRES_DB
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: taskboard-db-secret
                  key: POSTGRES_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: taskboard-db-secret
                  key: POSTGRES_PASSWORD
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 15
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: taskboard-api
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: ClusterIP
```
Aplicar y verificar:
```bash
$ kubectl apply -f api.yaml
$ kubectl rollout status deployment/taskboard-api
$ kubectl run test --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://taskboard-api:8080/health
```

#### Paso 6: Despliegue del Frontend con Network Load Balancer (NLB)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskboard-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: taskboard-web
  template:
    metadata:
      labels:
        app: taskboard-web
    spec:
      containers:
        - name: web
          image: YOUR_REGISTRY/taskboard-web:1.0.1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          env:
            - name: API_BASE_URL
              value: "http://taskboard-api:8080"
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 15
---
apiVersion: v1
kind: Service
metadata:
  name: taskboard-web
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  selector:
    app: taskboard-web
  ports:
    - name: http
      port: 80
      targetPort: 80
  type: LoadBalancer
```
Aplicar y obtener la URL pública de AWS:
```bash
$ kubectl apply -f frontend.yaml
$ kubectl get svc taskboard-web
```
*Figura 17.1: Aplicación TaskBoard ejecutándose en Amazon EKS*

#### Paso 7: Limpieza en AWS
```bash
$ kubectl delete ns taskboard
$ eksctl delete cluster --profile <your-profile> --name taskboard-cluster --region eu-central-1
```

---

### Ejecución de una aplicación en Microsoft Azure AKS

#### Paso 1: Autenticación y creación del clúster con Azure CLI
```bash
$ az login
$ az account set --subscription "<YOUR_SUBSCRIPTION_NAME_OR_ID>"

# Crear Resource Group
$ az group create \
  --name rg-taskboard-aks \
  --location westeurope

# Crear clúster AKS
$ az aks create \
  --resource-group rg-taskboard-aks \
  --name taskboard-aks \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --enable-managed-identity \
  --generate-ssh-keys

# Obtener credenciales
$ az aks get-credentials \
  --resource-group rg-taskboard-aks \
  --name taskboard-aks
$ kubectl get nodes
```

#### Paso 2: Despliegue de la aplicación en AKS
1. Crear el namespace, ConfigMap y Secret:
   ```bash
   $ kubectl create namespace taskboard
   $ kubectl config set-context --current --namespace=taskboard
   $ kubectl apply -f config.yaml
   $ kubectl create secret generic taskboard-db-secret \
     --from-literal=POSTGRES_USER=taskuser \
     --from-literal=POSTGRES_PASSWORD=taskpass
   ```

2. Crear `postgres-azure.yaml` (utiliza el CSI predeterminado de Azure y `subPath` en el montaje):
   ```yaml
             volumeMounts:
               - name: postgres-data
                 mountPath: /var/lib/postgresql/data
                 subPath: postgres
   ```
   *Figura 17.2: Configuración de subPath en Azure*

3. Desplegar componentes:
   ```bash
   $ kubectl apply -f postgres-azure.yaml
   $ kubectl apply -f api.yaml
   $ kubectl apply -f frontend.yaml
   $ kubectl get svc taskboard-web
   ```
   *Figura 17.3: TaskBoard en ejecución sobre Microsoft Azure AKS*

#### Paso 3: Limpieza en Azure
```bash
$ kubectl delete namespace taskboard
$ az group delete --name rg-taskboard-aks --yes --no-wait
```

---

### Ejecución de una aplicación en Google Kubernetes Engine (GKE)

#### Paso 1: Autenticación y creación del clúster en Google Cloud
```bash
$ gcloud auth login
$ gcloud projects create taskboard-demo-project --name="TaskBoard Demo"
$ gcloud config set project taskboard-demo-project
$ gcloud services enable container.googleapis.com compute.googleapis.com
$ gcloud config set compute/region europe-west3
$ gcloud config set compute/zone europe-west3-a

# Crear clúster estándar
$ gcloud container clusters create taskboard-gke \
  --zone europe-west3-a \
  --num-nodes 2 \
  --machine-type e2-small

# Obtener credenciales
$ gcloud container clusters get-credentials taskboard-gke --zone europe-west3-a
$ kubectl get nodes
```

#### Paso 2: Despliegue en GKE
1. Crear namespace y configuraciones:
   ```bash
   $ kubectl create namespace taskboard
   $ kubectl config set-context --current --namespace=taskboard
   $ kubectl apply -f config.yaml
   $ kubectl create secret generic taskboard-db-secret \
     --from-literal=POSTGRES_USER=taskuser \
     --from-literal=POSTGRES_PASSWORD=taskpass
   ```

2. Desplegar base de datos, API y Frontend:
   ```bash
   $ kubectl apply -f postgres-gke.yaml
   $ kubectl apply -f api.yaml
   $ kubectl apply -f frontend.yaml
   $ kubectl get svc taskboard-web
   ```
   *Figura 17.4: TaskBoard en ejecución sobre Google GKE*

#### Paso 3: Limpieza en Google Cloud
```bash
$ kubectl delete namespace taskboard
$ gcloud container clusters delete taskboard-gke --zone europe-west3-a --quiet
$ gcloud projects delete taskboard-demo-project
```

---

### Plataformas de contenedores Serverless

Los **contenedores serverless** representan el siguiente escalón evolutivo en la nube: abstraen no solo el plano de control, sino también los nodos de trabajo físicos y virtuales.

```
+-------------------------------------------------------------------------+
|                  MODELO DE CONTENEDORES SERVERLESS                     |
|                                                                         |
|  Desarrollador / CI-CD ----> Imagen OCI (Docker Hub / ECR / Artifact)   |
|                                   |                                     |
|                                   v                                     |
|           +-----------------------------------------------+             |
|           |        Motor Serverless (Totalmente Gestionado)|            |
|           |       (AWS Fargate / Cloud Run / Container Apps)|            |
|           +-----------------------------------------------+             |
|                                   |                                     |
|               +-------------------+-------------------+                 |
|               | (Carga baja)                          | (Pico de tráfico)
|               v                                       v                 |
|      [ 0 Instancias ] (Sin coste)           [ 100+ Instancias ]         |
+-------------------------------------------------------------------------+
```

| Plataforma | Proveedor | Base tecnológica | Características principales |
| :--- | :--- | :--- | :--- |
| **AWS Fargate** | Amazon Web Services | Integrado con EKS y ECS | Ejecución de Pods aislados por hardware; sin gestión de EC2; cobro por vCPU/RAM por segundo |
| **Google Cloud Run** | Google Cloud Platform | Knative / OCI estándar | Escalado ultrarrápido a cero instancias; facturación por petición y milisegundo; endpoints HTTPS automáticos |
| **Azure Container Apps (ACA)** | Microsoft Azure | Kubernetes + KEDA + Dapr | Escalado reactivo basado en colas o eventos; microservicios distribuidos sin administrar AKS |

#### ¿Cuándo elegir cada modelo?
- **Clústeres Kubernetes Gestionados (EKS, AKS, GKE)**: Para sistemas con estado (*stateful*), bases de datos en clúster, arquitecturas complejas de microservicios con dependencias de red avanzadas y control granular de planificación.
- **Contenedores Serverless (Cloud Run, Fargate, ACA)**: Para APIs REST sin estado (*stateless*), tareas por lotes (*batch jobs*), procesamiento de colas o arquitecturas orientadas a eventos donde se desea pagar solo por el cómputo exacto consumido.

---

### Resumen

En este capítulo aprendimos:
- A compilar imágenes multi-arquitectura con **Docker Buildx** para garantizar compatibilidad entre ARM64 y AMD64.
- Las ventajas del modelo de responsabilidad compartida de los servicios gestionados frente a clústeres propios.
- A aprovisionar clústeres y desplegar aplicaciones reales en **Amazon EKS**, **Microsoft Azure AKS** y **Google GKE** demostrando la portabilidad total de los manifiestos de Kubernetes.
- La función de los Services de tipo `LoadBalancer` para aprovisionar balanceadores de carga públicos automáticamente.
- El paradigma de los **contenedores serverless** (AWS Fargate, Google Cloud Run y Azure Container Apps) y su impacto en la reducción de la sobrecarga operativa.

---

### Preguntas

1. **¿Cuáles son las principales ventajas de usar un servicio gestionado de Kubernetes en lugar de administrar un clúster propio?**
2. **¿Qué componentes de Kubernetes son gestionados por el proveedor de nube y cuáles permanecen bajo tu responsabilidad?**
3. **¿Cuáles son los tres servicios gestionados de Kubernetes más populares y qué compañías los ofrecen?**
4. **¿Cómo garantizan los manifiestos de Kubernetes la portabilidad de una aplicación entre diferentes proveedores de nube?**
5. **¿Cuál es la diferencia entre un Service de tipo `ClusterIP` y uno de tipo `LoadBalancer`?**
6. **¿Por qué desplegamos PostgreSQL como un `StatefulSet` en lugar de un `Deployment`?**
7. **¿En qué se diferencia AWS Fargate de ejecutar un clúster EKS sobre nodos EC2 estándar?**
8. **¿Qué problema resuelven las plataformas de contenedores serverless como Google Cloud Run o Azure Container Apps?**
9. **¿En qué escenarios preferirías contenedores serverless frente a un clúster tradicional de Kubernetes?**
10. **¿Cuál es el concepto central que unifica a todos los servicios de Kubernetes gestionado y plataformas serverless?**
11. **¿Qué cambio fundamental de enfoque ocurre una vez que la aplicación se encuentra desplegada en producción en la nube?**

---

### Respuestas

1. **Ventajas de Kubernetes gestionado**:  
   El proveedor se encarga del aprovisionamiento, disponibilidad, copias de seguridad de `etcd`, parches y actualizaciones automáticas del plano de control, reduciendo drásticamente la sobrecarga de mantenimiento operativo y aumentando la resiliencia.

2. **Modelo de responsabilidad compartida**:  
   El proveedor gestiona el plano de control (`kube-apiserver`, `scheduler`, `controller-manager`, `etcd`). El usuario es responsable de los nodos de trabajo, la asignación de recursos, las políticas de seguridad (RBAC, NetworkPolicies), las imágenes y la configuración de sus aplicaciones.

3. **Los tres servicios principales**:  
   Amazon EKS (Amazon Web Services), Azure AKS (Microsoft Azure) y Google GKE (Google Cloud Platform).

4. **Portabilidad de manifiestos**:  
   Dado que todos los proveedores cumplen con el estándar unificado de la API de Kubernetes, los mismos archivos YAML de Deployments, Services y ConfigMaps se ejecutan idénticamente en cualquier nube sin requerir modificaciones en la lógica de la aplicación.

5. **ClusterIP frente a LoadBalancer**:  
   `ClusterIP` proporciona una IP virtual accesible únicamente dentro de la red interna del clúster (ideal para bases de datos o APIs internas). `LoadBalancer` aprovisiona automáticamente un balanceador de carga público externo provisto por la nube (AWS NLB, Azure LB, Google Cloud LB) con una IP pública enrutable.

6. **Uso de StatefulSet para bases de datos**:  
   Un StatefulSet garantiza identificadores de red estables y vinculación determinista a volúmenes persistentes dedicados que no se pierden al reiniciar o reprogramar el Pod.

7. **AWS Fargate frente a nodos EC2**:  
   En Fargate no existen instancias virtuales EC2 visibles que gestionar ni parchear. Cada Pod se ejecuta en un microentorno de cómputo aislado aprovisionado bajo demanda y facturado exactamente por los recursos consumidos.

8. **Solución que aporta el contenedor Serverless**:  
   Elimina la necesidad de dimensionar, administrar o pagar por servidores en espera cuando no hay tráfico, ofreciendo autoescalado automático (incluso a cero instancias).

9. **Casos ideales para Serverless**:  
   APIs HTTP sin estado, microservicios orientados a eventos, procesamiento por lotes o aplicaciones con tráfico esporádico o impredecible. Los clústeres tradicionales se prefieren para sistemas con estado persistente o arquitecturas complejas con requisitos estrictos de red.

10. **Concepto unificador**:  
    La **imagen de contenedor estandarizada OCI**. Es el artefacto universal de empaquetado y entrega que se ejecuta de forma idéntica en cualquier plataforma local, clúster gestionado o servicio serverless.

11. **Cambio de enfoque post-despliegue**:  
    La prioridad se traslada desde la entrega hacia la **observabilidad y fiabilidad operativa**: monitorización proactiva de métricas, agregación de logs, alertas tempranas y resolución de incidentes en tiempo real.
