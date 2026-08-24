# Parte 4: Docker, Kubernetes y la Nube
## Capítulo 15: Introducción a Kubernetes

En este capítulo aprenderás cómo **Kubernetes**—el orquestador de contenedores estándar de la industria—automatiza el despliegue, escalado y gestión operativa de aplicaciones contenerizadas. Exploraremos su arquitectura interna y componentes fundamentales, configuraremos un clúster local con **Docker Desktop, minikube o kind**, y aprenderemos a desplegar y observar cargas de trabajo en acción. Finalmente, descubriremos herramientas clave del ecosistema como **GitOps, Helm 3 y Kustomize**, que aportan automatización, consistencia y reproducibilidad a las operaciones modernas en Kubernetes.

---

### Temas tratados en este capítulo:
- Comprensión de la arquitectura interna de Kubernetes (Plano de control y Nodos de trabajo)
- Introducción a clústeres locales de Kubernetes (Docker Desktop, minikube y kind)
- El bloque fundamental: Pods, ciclo de vida y contenedores `pause`
- Pods y volúmenes persistentes (`PersistentVolumeClaim`)
- Gestión de réplicas y autorreparación con **ReplicaSets**
- Introducción a **Deployments** de Kubernetes
- Descubrimiento de servicios y balanceo de carga con **Services** (ClusterIP, NodePort, Headless)
- Enrutamiento basado en contexto con **Ingress Controllers**
- Herramientas del ecosistema: **GitOps, Helm 3 y Kustomize**

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Explicar la arquitectura de alto nivel de Kubernetes y las funciones del plano de control (*API Server, Controller Manager, Scheduler, etcd*) y los nodos de trabajo (*kubelet, kube-proxy, container runtime*).
- Describir qué es un Pod de Kubernetes y cómo comparte los *namespaces* del kernel de Linux entre múltiples contenedores.
- Definir y desplegar Pods, PersistentVolumeClaims, ReplicaSets, Deployments y Services en formato declarativo YAML.
- Configurar y operar un clúster local de Kubernetes mediante Docker Desktop, minikube o kind.
- Implementar enrutamiento HTTP por rutas de URL utilizando un controlador Ingress (NGINX Ingress).
- Entender cómo Helm 3, Kustomize y GitOps simplifican la entrega continua y la gestión de configuraciones sin duplicación.

---

### Requisitos técnicos

Para seguir los ejemplos prácticos de este capítulo necesitarás un entorno Docker funcional y la herramienta CLI `kubectl`.

Verificar la instalación de `kubectl`:
```bash
$ kubectl version --client
```

Preparar el espacio de trabajo local:
```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir chapter-15 && cd chapter-15
```

Los ejemplos completos se encuentran disponibles en:  
[https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-15/solution](https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-15/solution)

---

### Comprensión de la arquitectura de Kubernetes

Un clúster de Kubernetes se compone de un conjunto de servidores (máquinas virtuales o servidores físicos *bare metal*) divididos en dos funciones esenciales:
1. **Nodos maestros (*Control Plane / Master Nodes*)**: Encargados de administrar el clúster, planificar cargas y coordinar el estado deseado.
2. **Nodos de trabajo (*Worker Nodes*)**: Los ejecutores que hospedan los Pods con las aplicaciones de los usuarios.

```
       +-----------------------------------------------------------+
       |                       API SERVER                          |
       |  (Gateway REST de comunicación para kubectl y componentes) |
       +--------+------------------+-------------------+-----------+
                |                  |                   |
       +--------v-------+  +-------v--------+  +-------v--------+
       |   Controller   |  |   Scheduler    |  |     etcd       |
       |    Manager     |  | (Planificación |  | (Almacén de    |
       | (Reconciliación|  |  de Pods en    |  |  estado HA     |
       |  de estado)    |  |    nodos)      |  |  Raft Consensus|
       +----------------+  +----------------+  +----------------+
                                   |
         +-------------------------+-------------------------+
         |                                                   |
+--------v-------------------------+       +-----------------v-------------------------+
|           WORKER NODE 1          |       |           WORKER NODE 2                   |
| +------------+ +---------------+ |       | +------------+ +---------------+          |
| |  Kubelet   | |  kube-proxy   | |       | |  Kubelet   | |  kube-proxy   |          |
| +------------+ +---------------+ |       | +------------+ +---------------+          |
| +------------------------------+ |       | +------------------------------+          |
| | Container Runtime (containerd)| |       | | Container Runtime (containerd)|          |
| +------------------------------+ |       | +------------------------------+          |
|   [ Pod 1 ]      [ Pod 2 ]       |       |   [ Pod 3 ]      [ Pod 4 ]                |
+----------------------------------+       +-------------------------------------------+
```
*Figura 15.1 – Arquitectura global de un clúster de Kubernetes*

#### Componentes del Plano de Control (*Master Nodes*):
- **API Server (`kube-apiserver`)**: La puerta de enlace central de Kubernetes. Expone una API REST consumida por `kubectl`, los controladores y los agentes de los nodos.
- **Cluster Store (`etcd`)**: Base de datos clave-valor distribuida y consistente (basada en el algoritmo de consenso Raft). Almacena todo el estado, topología, secretos y configuraciones del clúster. Requiere un número impar de nodos (1, 3, 5) para quórum y alta disponibilidad.
- **Controller Manager (`kube-controller-manager`)**: Bucle de control continuo que detecta discrepancias entre el estado actual y el deseado, ejecutando acciones correctivas (e.g., recrear Pods caídos).
- **Scheduler (`kube-scheduler`)**: Asigna los Pods recién creados a los nodos más adecuados evaluando requerimientos de CPU/memoria, afinidad, políticas de red y calidad de servicio.

#### Componentes de los Nodos de Trabajo (*Worker Nodes*):
- **Kubelet**: Agente principal del nodo. Recibe las especificaciones de los Pods (`PodSpecs`) desde el API Server y se asegura de que sus contenedores se ejecuten y permanezcan saludables.
- **Container Runtime**: Motor que ejecuta los contenedores (predeterminadamente `containerd` o `CRI-O`).
- **kube-proxy**: Demonio de red que mantiene las reglas de cortafuegos (*iptables* / IPVS) en el host para enrutar el tráfico de los Services hacia los Pods correspondientes.

---

### Clústeres locales de Kubernetes

Para practicar y desarrollar localmente disponemos de tres herramientas estándar:

| Herramienta | Ideal para | Características |
| :--- | :--- | :--- |
| **Docker Desktop** | Desarrollo diario | Activación con un solo clic en Ajustes, integración total con Docker CLI |
| **minikube** | Aprendizaje y pruebas | Clúster mononodo en máquina virtual/contenedor, amplio soporte de *addons* (Ingress, Dashboard) |
| **kind (Kubernetes in Docker)** | CI/CD y automatización | Rápido, ligero, ejecuta nodos completos de K8s como contenedores Docker |

*Tabla 15.1 – Comparativa de entornos locales de Kubernetes*

#### 1. Docker Desktop con Kubernetes:
- Abre **Docker Desktop** -> **Settings** -> **Kubernetes** -> Marca **Enable Kubernetes** -> Haz clic en **Apply & restart** (*Figura 15.4*).
- Verifica la instalación:
  ```bash
  $ kubectl get nodes
  ```
  Salida:
  ```text
  NAME             STATUS   ROLES           AGE   VERSION
  docker-desktop   Ready    control-plane   2m    v1.29.4
  ```

#### 2. minikube:
```bash
$ minikube start
$ kubectl get nodes
$ minikube addons enable ingress
$ minikube addons enable metrics-server
$ minikube dashboard
$ minikube stop
$ minikube delete
```
*Figura 15.5 – Inicio y verificación de minikube*

#### 3. kind (Kubernetes in Docker):
```bash
$ brew install kind       # macOS
$ choco install kind      # Windows
$ kind create cluster --name demo
$ kubectl cluster-info --context kind-demo
$ kubectl get nodes
$ kind delete cluster --name demo
```
*Figura 15.6 – Creación de un clúster con kind*

> [!NOTE]
> **Gestión de contextos con `kubectl`**:  
> `kubectl` utiliza el archivo `~/.kube/config` para almacenar clústeres y credenciales. Puedes listar contextos con `kubectl config get-contexts` y cambiar al clúster de Docker Desktop con `kubectl config use-context docker-desktop`.

---

### Introducción a los Pods

En Kubernetes **no se ejecutan contenedores individuales directamente**; la unidad atómica mínima de ejecución es el **Pod**.

Un Pod es una abstracción que envuelve uno o más contenedores estrechamente vinculados que comparten:
- El mismo espacio de nombres de red (*network namespace*), lo que significa que **todos los contenedores del Pod comparten la misma dirección IP** y pueden comunicarse entre sí a través de `localhost`.
- Los mismos volúmenes de almacenamiento compartidos.
- El mismo espacio de nombres de IPC y UTS.

```
+-------------------------------------------------------------+
| POD (IP: 10.0.12.3)                                         |
|                                                             |
|   +-----------------------+     +-------------------------+ |
|   | Contenedor Principal  |     | Contenedor Secundario   | |
|   | (NGINX - Puerto 80)   |     | (App - Puerto 3000)     | |
|   +-----------+-----------+     +------------+------------+ |
|               |                              |              |
|               +-------- localhost:3000 ------+              |
|                                                             |
|   [ Volumen compartido montado en /data ]                   |
+-------------------------------------------------------------+
```
*Figura 15.7 – Estructura de un Pod con múltiples contenedores compartiendo red y almacenamiento*

#### ¿Cómo funciona el contenedor `pause`?
Para crear el espacio de red compartido, Kubernetes inicia primero un contenedor auxiliar ultraligero llamado **pause container**. Todos los demás contenedores del Pod reutilizan el espacio de red del contenedor `pause` mediante `--net container:pause` (*Figura 15.8 y Figura 15.9*).

#### Simulación interactiva con Docker:
```bash
$ docker container run --detach --name pause nginx:alpine
$ docker container run --name main -d -it --net container:pause alpine:latest ash
$ docker exec -it main /bin/sh
/ # wget -qO - localhost
/ # ip a show eth0
/ # exit
$ docker network inspect bridge
$ docker container rm -f pause main
```
*Figura 15.10 y Figura 15.11 – Demostración de comunicación por localhost compartiendo red*

#### Ciclo de vida del Pod (*Pod Lifecycle*):
- **Pending**: Aceptado por el clúster, descargando imágenes o esperando asignación de nodo.
- **Running**: Todos los contenedores han sido creados y al menos uno está en ejecución o iniciándose.
- **Succeeded**: Todos los contenedores terminaron con éxito (código de salida `0`).
- **Failed**: Al menos un contenedor terminó con error (código de salida distinto de cero) (*Figura 15.12*).

---

### Despliegue declarativo de un Pod

1. Crear el archivo `pod.yaml`:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: web-pod
   spec:
     containers:
     - name: web
       image: nginx:alpine
       ports:
       - containerPort: 80
   ```

2. Aplicar el manifiesto y verificar:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-15
   $ kubectl config use-context docker-desktop
   $ kubectl apply -f pod.yaml
   $ kubectl get pods
   $ kubectl describe pod/web-pod
   ```
   *Figura 15.13 – Salida de kubectl describe pod/web-pod*

---

### Pods y volúmenes de datos (`PersistentVolumeClaim`)

1. Crear `volume-claim.yaml` solicitando 2 GiB de almacenamiento persistente:
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: my-data-claim
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 2Gi
   ```

2. Aplicar y listar el PVC:
   ```bash
   $ kubectl apply -f volume-claim.yaml
   $ kubectl get pvc
   ```
   *Figura 15.14 – PersistentVolumeClaim creado en el clúster*

3. Eliminar el Pod anterior:
   ```bash
   $ kubectl delete pod/web-pod
   ```

4. Crear `pod-with-vol.yaml` montando el volumen en `/data`:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: web-pod
   spec:
     containers:
     - name: web
       image: nginx:alpine
       ports:
       - containerPort: 80
       volumeMounts:
       - name: my-data
         mountPath: /data
     volumes:
     - name: my-data
       persistentVolumeClaim:
         claimName: my-data-claim
   ```

5. Probar la persistencia de datos:
   ```bash
   $ kubectl apply -f pod-with-vol.yaml
   $ kubectl exec -it web-pod -- /bin/sh
   / # cd /data
   /data # echo "Hello world!" > sample.txt
   /data # exit
   $ kubectl delete pod/web-pod
   $ kubectl apply -f pod-with-vol.yaml
   $ kubectl exec -it web-pod -- ash
   / # cat /data/sample.txt
   ```
   *(Comprueba que devuelve `Hello world!`).*

6. Limpiar recursos:
   ```bash
   $ kubectl delete pod/web-pod
   $ kubectl delete pvc/my-data-claim
   ```

---

### Kubernetes ReplicaSet: Escalabilidad y Autorreparación

Un **ReplicaSet** garantiza que un número específico de réplicas de un Pod idéntico se encuentren siempre en ejecución activa. Si un Pod falla o el nodo se cae, el ReplicaSet programa uno nuevo inmediatamente (*Figura 15.15*).

1. Crear `replicaset.yaml`:
   ```yaml
   apiVersion: apps/v1
   kind: ReplicaSet
   metadata:
     name: rs-web
   spec:
     selector:
       matchLabels:
         app: web
     replicas: 3
     template:
       metadata:
         labels:
           app: web
       spec:
         containers:
         - name: nginx
           image: nginx:alpine
           ports:
           - containerPort: 80
   ```

2. Aplicar y verificar:
   ```bash
   $ kubectl apply -f replicaset.yaml
   $ kubectl get rs
   $ kubectl get pods
   ```

3. Demostración de autorreparación (*Self-healing*):
   ```bash
   # Elimina un Pod aleatorio del ReplicaSet
   $ kubectl delete po/$(kubectl get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
   $ kubectl get pods
   $ kubectl describe rs/rs-web
   ```
   *(Observa en Events cómo el ReplicaSet crea un nuevo Pod de inmediato para mantener las 3 réplicas deseadas - Figura 15.16).*

4. Limpiar:
   ```bash
   $ kubectl delete rs/rs-web
   ```

---

### Kubernetes Deployments

Un **Deployment** es un objeto de nivel superior que gestiona y envuelve a los ReplicaSets, aportando capacidades indispensables de **actualizaciones continuas sin tiempo de inactividad (*Rolling Updates*)** y **reversión instantánea de versiones (*Rollbacks*)** (*Figura 15.17*).

---

### Kubernetes Services: Descubrimiento de servicios y balanceo

Los Pods son efímeros y sus IPs cambian con cada recreación. Un **Service** proporciona una IP virtual estable (*ClusterIP*) y un nombre DNS unificado dentro del clúster, balanceando la carga entre todos los Pods seleccionados por sus etiquetas (*labels*) (*Figura 15.18 y Figura 15.19*).

#### Ejercicio práctico con Services (`service-demo`):

1. Preparar la carpeta:
   ```bash
   $ mkdir service-demo && cd service-demo
   ```

2. Crear `hello-rs.yaml`:
   ```yaml
   apiVersion: apps/v1
   kind: ReplicaSet
   metadata:
     name: hello-rs
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
   ```
   *Figura 15.20 – ReplicaSet con imagen hello*

3. Aplicar:
   ```bash
   $ kubectl apply -f hello-rs.yaml
   $ kubectl get pods -l app=hello -o wide
   ```

4. Crear `hello-svc.yaml`:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: hello-svc
   spec:
     type: ClusterIP
     selector:
       app: hello
     ports:
     - port: 80
       targetPort: 80
   ```
   *Figura 15.21 – Definición de Service ClusterIP*

5. Aplicar e inspeccionar:
   ```bash
   $ kubectl apply -f hello-svc.yaml
   $ kubectl get svc hello-svc -o wide
   $ kubectl get endpoints hello-svc
   $ kubectl describe svc hello-svc
   ```

6. Probar el balanceo de carga y resolución DNS interna:
   ```bash
   $ kubectl run curl --rm -it --restart=Never --image=curlimages/curl:8.15.0 -- sh
   # Dentro del contenedor:
   for i in $(seq 1 10); do curl -s http://hello-svc; echo; sleep 1; done
   exit
   ```

7. Probar acceso local con reenvío de puertos (*Port-forward*):
   ```bash
   $ kubectl port-forward service/hello-svc 8080:80
   # En otra terminal:
   $ curl -s http://localhost:8080
   ```

8. (Opcional) Servicio sin IP (*Headless Service*) para descubrimiento directo:
   Crear `hello-headless.yaml`:
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: hello-headless
   spec:
     clusterIP: None
     selector:
       app: hello
     ports:
     - port: 80
       targetPort: 80
   ```
   Verificar registros DNS múltiples:
   ```bash
   $ kubectl apply -f hello-headless.yaml
   $ kubectl run dns --rm -it --restart=Never --image=busybox:1.36 -- nslookup hello-headless
   ```
   *Figura 15.24 – Registros A devueltos por el servicio Headless*

9. Limpieza:
   ```bash
   $ kubectl delete svc hello-svc hello-headless 2>/dev/null || true
   $ kubectl delete -f hello-rs.yaml
   ```

---

### Enrutamiento basado en contexto con Ingress

Un **Ingress Controller** (como NGINX Ingress o Traefik) actúa como proxy inverso en la capa 7 (HTTP/HTTPS), enrutando peticiones hacia distintos servicios según el nombre de host o la ruta de URL (*Figura 15.25*).

#### Ejercicio práctico con Ingress en minikube:

1. Iniciar minikube y habilitar el complemento Ingress:
   ```bash
   $ minikube start
   $ minikube addons enable ingress
   $ kubectl get pods -n kube-system | grep ingress
   ```
   *Figura 15.26 – Habilitación del addon Ingress en minikube*

2. Crear la carpeta del ejercicio:
   ```bash
   $ mkdir ingress-demo && cd ingress-demo
   ```

3. Crear `backend-web.yaml`:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: web
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: web
     template:
       metadata:
         labels:
           app: web
       spec:
         containers:
         - name: nginx
           image: nginxdemos/hello:plain-text
           ports:
           - containerPort: 80
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: web-service
   spec:
     selector:
       app: web
     ports:
     - port: 80
       targetPort: 80
   ```
   *Figura 15.27 – Despliegue y servicio web*

4. Crear `backend-api.yaml`:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: api
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: api
     template:
       metadata:
         labels:
           app: api
       spec:
         containers:
         - name: api
           image: hashicorp/http-echo:latest
           args:
           - "-text=Hello from the API service"
           ports:
           - containerPort: 5678
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: api-service
   spec:
     selector:
       app: api
     ports:
     - port: 5678
       targetPort: 5678
   ```
   *Figura 15.28 – Despliegue y servicio API*

5. Aplicar ambos backends:
   ```bash
   $ kubectl apply -f backend-web.yaml
   $ kubectl apply -f backend-api.yaml
   ```

6. Crear `context-routing.yaml`:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: context-routing
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /
   spec:
     ingressClassName: nginx
     rules:
     - host: demo.local
       http:
         paths:
         - path: /web
           pathType: Prefix
           backend:
             service:
               name: web-service
               port:
                 number: 80
         - path: /api
           pathType: Prefix
           backend:
             service:
               name: api-service
               port:
                 number: 5678
   ```
   *Figura 15.29 – Recurso Ingress para enrutamiento por rutas /web y /api*

7. Aplicar y probar:
   ```bash
   $ kubectl apply -f context-routing.yaml
   $ INGRESS_IP=$(minikube ip)
   $ curl -s http://$INGRESS_IP/web --header "Host: demo.local" | head -n 2
   $ curl -s http://$INGRESS_IP/api --header "Host: demo.local"
   ```

8. Limpieza:
   ```bash
   $ kubectl delete ingress context-routing
   $ kubectl delete -f backend-web.yaml
   $ kubectl delete -f backend-api.yaml
   $ minikube stop
   $ minikube delete
   ```

---

### Herramientas del ecosistema: GitOps, Helm 3 y Kustomize

#### 1. GitOps: Entrega continua basada en repositorios Git
- **Concepto**: Git actúa como la **única fuente de verdad** (*Single Source of Truth*). Todo manifiesto de Kubernetes reside en repositorios de control de versiones.
- **Herramientas**: **Argo CD** y **Flux** sincronizan continuamente el clúster con el repositorio Git, aplicando cambios automáticamente y evitando desviaciones de configuración (*configuration drift*).

#### 2. Helm 3: El gestor de paquetes de Kubernetes
- Empaqueta manifiestos en plantillas parametrizadas llamadas **Charts**.
- Permite instalar aplicaciones complejas con un solo comando:
  ```bash
  $ helm install my-app bitnami/nginx
  ```
- No requiere componentes en el servidor (sin Tiller), integrándose nativamente con RBAC y facilitando *rollbacks*.

#### 3. Kustomize: Personalización declarativa nativa
- Permite adaptar manifiestos YAML para múltiples entornos (desarrollo, staging, producción) mediante capas (*overlays*) sin introducir lenguajes de plantillas complejos.
- Estructura típica:
  ```text
  my-app/
  ├── base/
  │   ├── deployment.yaml
  │   ├── service.yaml
  │   └── kustomization.yaml
  └── overlays/
      ├── staging/
      │   └── kustomization.yaml
      └── production/
          └── kustomization.yaml
  ```
- Despliegue con `kubectl`:
  ```bash
  $ kubectl apply -k overlays/staging/
  ```

---

### Resumen

En este capítulo aprendimos:
- La arquitectura interna de Kubernetes dividida entre el plano de control (*API Server, etcd, Controller Manager, Scheduler*) y los nodos de trabajo (*Kubelet, Container Runtime, kube-proxy*).
- Cómo levantar clústeres locales para desarrollo con Docker Desktop, minikube y kind.
- Los conceptos fundamentales: **Pods** (unidad atómica con contenedor `pause`), **ReplicaSets** (autorreparación y escalado), **Deployments** (actualizaciones continuas) y **Services** (IP virtual y balanceo DNS).
- El enrutamiento HTTP en capa 7 mediante **Ingress Controllers**.
- La gestión profesional de configuraciones y despliegues con **GitOps, Helm 3 y Kustomize**.

---

### Lecturas recomendadas

- **Documentación oficial de Kubernetes**: [https://kubernetes.io/docs/](https://kubernetes.io/docs/)
- **Tutorial interactivo de fundamentos**: [https://kubernetes.io/docs/tutorials/kubernetes-basics/](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
- **Documentación de Helm 3**: [https://helm.sh/docs/](https://helm.sh/docs/)
- **Guía de Kustomize**: [https://kustomize.io/](https://kustomize.io/)
- **Argo CD para GitOps**: [https://argo-cd.readthedocs.io/](https://argo-cd.readthedocs.io/)
- **Flux CD**: [https://fluxcd.io/](https://fluxcd.io/)
- **Kubernetes Gateway API**: [https://gateway-api.sigs.k8s.io/](https://gateway-api.sigs.k8s.io/)

---

### Preguntas

1. **¿Qué problema resuelve Kubernetes y en qué se diferencia de los despliegues manuales tradicionales?**
2. **¿Cuáles son los componentes principales del plano de control y de los nodos de trabajo en Kubernetes?**
3. **¿Qué es un Pod y en qué se diferencia de un contenedor Docker independiente?**
4. **¿Cuáles son las opciones de Kubernetes local y qué diferencia a Docker Desktop, minikube y kind?**
5. **¿Qué significa configuración declarativa en Kubernetes y por qué es importante?**
6. **¿Cómo extiende GitOps el modelo declarativo de Kubernetes para automatizar la entrega?**
7. **¿Qué es un Chart de Helm y cómo simplifica el despliegue de aplicaciones complejas?**
8. **¿Cómo ayuda Kustomize a gestionar configuraciones específicas por entorno sin duplicar código YAML?**
9. **¿Por qué se utilizan conjuntamente Helm, Kustomize y GitOps en entornos modernos?**
10. **¿Qué pasos prácticos seguirías para desplegar una aplicación básica en tu clúster local?**

---

### Respuestas

1. **Problema que resuelve Kubernetes**:  
   Automatiza el despliegue, escalado, autorreparación y balanceo de red de aplicaciones contenerizadas en clústeres masivos de servidores mediante un bucle continuo de reconciliación declarativa.

2. **Componentes clave**:  
   - *Plano de control*: API Server (puerta de enlace), `etcd` (almacén de estado), Controller Manager (reconciliación) y Scheduler (planificación).  
   - *Nodos de trabajo*: Kubelet (agente del nodo), Container Runtime (ejecutor de contenedores) y `kube-proxy` (enrutamiento de red).

3. **Definición de Pod**:  
   Es la unidad mínima desplegable. Agrupa uno o varios contenedores que comparten la misma dirección IP, *namespaces* de red e IPC y volúmenes de almacenamiento mediante un contenedor auxiliar `pause`.

4. **Comparativa de clústeres locales**:  
   Docker Desktop ofrece la integración más rápida para desarrollo diario; minikube proporciona una máquina virtual con soporte para extensiones; y kind ejecuta clústeres multinodo ultrarrápidos dentro de contenedores Docker ideales para pruebas CI/CD.

5. **Configuración declarativa**:  
   Define el estado final deseado (número de réplicas, versión de imagen) en lugar de comandos paso a paso. Kubernetes compara constantemente el estado real con el declarado y ejecuta correcciones automáticas.

6. **Impacto de GitOps**:  
   Almacena los manifiestos en repositorios Git como única fuente de verdad; controladores como Argo CD sincronizan el clúster automáticamente con cada commit, asegurando trazabilidad y auditoría.

7. **Helm Charts**:  
   Son paquetes reutilizables que agrupan manifiestos YAML y plantillas parametrizadas, permitiendo instalar, actualizar o revertir aplicaciones complejas con un solo comando.

8. **Función de Kustomize**:  
   Permite aplicar parches y sobreescrituras (*overlays*) sobre una base común de manifiestos YAML puros sin necesidad de aprender lenguajes de plantillas adicionales.

9. **Uso conjunto de herramientas**:  
   Helm gestiona y versiona paquetes; Kustomize ajusta parámetros específicos por entorno (staging, prod); y GitOps automatiza la sincronización continua hacia el clúster.

10. **Pasos prácticos de despliegue**:  
    Iniciar Docker Desktop o minikube, verificar la conexión con `kubectl get nodes`, redactar un manifiesto YAML para un Deployment y Service, aplicarlo con `kubectl apply -f <archivo>.yaml` y verificar los Pods con `kubectl get pods` y `kubectl port-forward`.
