# Parte 4: Docker, Kubernetes y la Nube
## Capítulo 16: Despliegue, actualización y seguridad de una aplicación con Kubernetes

En el capítulo anterior exploramos los bloques de construcción fundamentales de Kubernetes: Pods, ReplicaSets, Deployments y Services.

Ahora es el momento de poner en práctica todo ese conocimiento. En este capítulo desplegaremos una **aplicación multi-servicio completa** llamada **TaskBoard** en un clúster de Kubernetes, aprenderemos a actualizarla sin interrupción de servicio (*zero-downtime*) y la blindaremos aplicando mecanismos de seguridad nativos. Descubrirás cómo Kubernetes habilita la entrega continua mediante **actualizaciones continuas (*Rolling Updates*)** y **despliegues Azul/Verde (*Blue/Green*)**, cómo supervisa la salud interna con **sondas de vida (*Liveness*), preparación (*Readiness*) e inicio (*Startup*)**, y cómo proteger credenciales sensibles mediante **Kubernetes Secrets**.

Por último, aplicaremos las mejores prácticas de seguridad en producción: ejecución como usuario sin privilegios (`non-root`), sistemas de archivos de solo lectura, principio de menor privilegio con **RBAC**, estándares de seguridad de Pods (*Pod Security Admission*) y aislamiento de red granular con **NetworkPolicies**.

---

### Temas tratados en este capítulo:
- Despliegue paso a paso de una aplicación multi-servicio (Frontend NGINX, API REST Python y Base de Datos PostgreSQL)
- Definición y uso de sondas de vida (*Liveness*), preparación (*Readiness*) e inicio (*Startup*)
- Despliegues y actualizaciones sin tiempo de inactividad (*Rolling Updates* y *Blue/Green Deployments*)
- Prácticas esenciales de seguridad en Kubernetes:
  - Ejecución como usuario `non-root` y sistemas de archivos de solo lectura
  - Identidades y control de acceso basado en roles con ServiceAccounts y RBAC
  - Restricción de perfiles con Pod Security Admission (`restricted`)
  - Aislamiento de tráfico con políticas de red (*NetworkPolicies*)
  - Gestión y rotación segura de secretos montados en volúmenes

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Desplegar aplicaciones multicapa interconectadas mediante ConfigMaps, Secrets, PVCs, Deployments y Services.
- Configurar sondas para garantizar alta disponibilidad y evitar que el tráfico llegue a instancias no preparadas.
- Ejecutar actualizaciones progresivas y conmutaciones Azul/Verde con *rollback* instantáneo en caso de fallos.
- Implementar defensas en profundidad: endurecimiento de Pods, microsegmentación de red y menor privilegio con RBAC.

---

### Requisitos técnicos

Necesitarás un entorno local de Kubernetes (preferiblemente **minikube**, Docker Desktop o kind) y la herramienta CLI `kubectl`.

Verificar versiones:
```bash
$ kubectl version --client
$ minikube version
```

Preparar el espacio de trabajo:
```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir chapter-16 && cd chapter-16
```

Los manifiestos y código fuente se encuentran en el repositorio del libro:  
[https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-16/solutions](https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-16/solutions)

---

### Despliegue de nuestra primera aplicación completa: TaskBoard

La aplicación **TaskBoard** consta de tres componentes:
1. `frontend`: Interfaz web estática servida por NGINX (puerto 80).
2. `api`: Backend REST en Python/Gunicorn (puerto 8080).
3. `db`: Base de datos PostgreSQL 16 con almacenamiento persistente (puerto 5432).

#### 1. Iniciar el clúster y construir las imágenes:
```bash
$ minikube start
$ kubectl cluster-info
$ kubectl get nodes
$ minikube image build -t taskboard-api:1.0.0 ./solutions/task-board/api
$ minikube image build -t taskboard-frontend:1.0.0 ./solutions/task-board/frontend
```

#### 2. Crear un espacio de nombres dedicado:
```bash
$ kubectl create namespace taskboard
$ kubectl config set-context --current --namespace=taskboard
$ kubectl get ns
```

#### 3. Configuración y credenciales (`k8s/config.yaml`):
Crea el archivo `k8s/config.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: taskboard-config
data:
  DB_HOST: "db"
  DB_PORT: "5432"
  DB_NAME: "taskboard"
---
apiVersion: v1
kind: Secret
metadata:
  name: taskboard-secrets
type: Opaque
stringData:
  DB_USER: "taskuser"
  DB_PASSWORD: "taskpassword"
```
*Figura 16.1: ConfigMap y Secret para TaskBoard*

Aplicar:
```bash
$ kubectl apply -f k8s/config.yaml
$ kubectl get configmap,secret
```

#### 4. Almacenamiento persistente para PostgreSQL (`k8s/storage.yaml`):
Crea `k8s/storage.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
*Figura 16.2: PersistentVolumeClaim para la base de datos*

Aplicar:
```bash
$ kubectl apply -f k8s/storage.yaml
```

#### 5. Despliegue y servicio de la base de datos (`k8s/db.yaml`):
Crea `k8s/db.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: taskboard
    tier: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: taskboard
      tier: db
  template:
    metadata:
      labels:
        app: taskboard
        tier: db
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: taskboard-config
                  key: DB_NAME
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: taskboard-secrets
                  key: DB_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: taskboard-secrets
                  key: DB_PASSWORD
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: db-data
---
apiVersion: v1
kind: Service
metadata:
  name: db
  labels:
    app: taskboard
    tier: db
spec:
  type: ClusterIP
  selector:
    app: taskboard
    tier: db
  ports:
    - port: 5432
      targetPort: 5432
```
*Figura 16.3: Despliegue y servicio de PostgreSQL*

Aplicar y verificar:
```bash
$ kubectl apply -f k8s/db.yaml
$ kubectl rollout status deploy/db
$ kubectl get pods -l tier=db -o wide
```

#### 6. Despliegue y servicio de la API (`k8s/api.yaml`):
Crea `k8s/api.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app: taskboard
    tier: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: taskboard
      tier: api
  template:
    metadata:
      labels:
        app: taskboard
        tier: api
    spec:
      containers:
        - name: api
          image: taskboard-api:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: taskboard-config
            - secretRef:
                name: taskboard-secrets
---
apiVersion: v1
kind: Service
metadata:
  name: api
  labels:
    app: taskboard
    tier: api
spec:
  type: ClusterIP
  selector:
    app: taskboard
    tier: api
  ports:
    - port: 8080
      targetPort: 8080
```
*Figura 16.4: Despliegue y servicio de la API con 2 réplicas*

Aplicar y verificar endpoint de salud:
```bash
$ kubectl apply -f k8s/api.yaml
$ kubectl rollout status deploy/api
$ kubectl get svc api
$ kubectl run busybox --image=busybox --rm -it --restart=Never -- wget -S -qO- http://api:8080/health || true
```
*Figura 16.5: Prueba de salud de la API en ejecución*

#### 7. Despliegue y servicio del Frontend (`k8s/frontend.yaml`):
Crea `k8s/frontend.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: taskboard
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: taskboard
      tier: frontend
  template:
    metadata:
      labels:
        app: taskboard
        tier: frontend
    spec:
      containers:
        - name: frontend
          image: taskboard-frontend:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: taskboard
    tier: frontend
spec:
  type: ClusterIP
  selector:
    app: taskboard
    tier: frontend
  ports:
    - port: 80
      targetPort: 80
```
*Figura 16.6: Manifiesto del Frontend*

Aplicar y acceder a la aplicación:
```bash
$ kubectl apply -f k8s/frontend.yaml
$ kubectl rollout status deploy/frontend
$ minikube service frontend -n taskboard --url
# O alternativamente:
$ kubectl port-forward svc/frontend 8081:80
```
*Figura 16.7 – Interfaz web de la aplicación TaskBoard en el navegador*

Prueba funcional dentro del clúster:
```bash
$ kubectl run busybox --image=busybox --rm -it --restart=Never -- wget -S -qO- http://api:8080/tasks || true
```

---

### Definición de sondas de vida, preparación e inicio

Las sondas (*probes*) permiten al `kubelet` diagnosticar el estado real de la aplicación interna:
- **Sonda de vida (*Liveness Probe*)**: Detecta si la aplicación se ha bloqueado en un bucle infinito o interbloqueo (*deadlock*). Si falla, **reinicia el contenedor**.
- **Sonda de preparación (*Readiness Probe*)**: Detecta si la aplicación está lista para procesar tráfico. Si falla, **retira el Pod de los endpoints del Service** sin reiniciarlo.
- **Sonda de inicio (*Startup Probe*)**: Protege aplicaciones lentas en arrancar (migraciones de BD, carga de modelos), pausando las comprobaciones de vida y preparación hasta que se complete la inicialización.

#### Configuración de sondas en `k8s/api.yaml`:
```yaml
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 2
            periodSeconds: 5
          startupProbe:
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 30
            periodSeconds: 5
```
*Figura 16.8: Configuración de sondas para el servicio API*

Aplicar y verificar:
```bash
$ kubectl apply -f k8s/api.yaml
$ kubectl rollout status deploy/api
$ kubectl describe pod -l tier=api
```

#### Demostración de autorreparación y protección ante fallos:
Prepara tres terminales simultáneas:
- **Terminal A (Pods)**: `$ kubectl get pods -l app=taskboard,tier=api -w`
- **Terminal B (Endpoints)**: `$ kubectl get endpoints api -w`
- **Terminal C (Eventos)**: `$ kubectl events -watch`

1. **Simulación de caída de proceso (Liveness)**:
   ```bash
   $ POD=$(kubectl get pod -l app=taskboard,tier=api -o jsonpath='{.items[0].metadata.name}')
   $ kubectl exec "$POD" -- sh -c 'kill 1'
   ```
   *(El ReplicaSet detecta la terminación y lanza un nuevo Pod de reemplazo de inmediato).*

2. **Simulación de caída de dependencia (Readiness)**:
   ```bash
   $ kubectl scale deploy/db --replicas=0
   ```
   *(La base de datos se apaga -> `/health` devuelve 503 -> Los Pods pasan a estado `0/1 NotReady` -> El Service retira sus IPs de los endpoints y no envía tráfico).*
   ```bash
   $ kubectl scale deploy/db --replicas=1
   ```
   *(La base de datos se recupera -> `/health` devuelve 200 -> Los Pods vuelven a estar `1/1 Ready` -> El tráfico se reanuda automáticamente sin intervención manual).*

---

### Despliegues y actualizaciones sin tiempo de inactividad (*Zero-Downtime*)

#### 1. Actualización continua (*Rolling Update*)
Kubernetes crea nuevos Pods con la nueva versión, espera a que pasen la sonda de preparación (*Readiness Probe*), y luego termina gradualmente los Pods antiguos.

1. Modificar `task-board/api/server.py` para devolver la versión 1.1.0:
   ```python
   @app.get("/health")
   def health():
       try:
           with engine.connect() as conn:
               conn.execute(text("SELECT 1"))
           # Added version field
           return {"status": "ok", "version": "1.1.0"}, 200
       except OperationalError:
           return {"status": "degraded", "version": "1.1.0"}, 503
   ```

2. Construir la imagen `v1.1.0`:
   ```bash
   $ minikube image build -t taskboard-api:1.1.0 ./task-board/api
   ```

3. Desencadenar la actualización continua:
   ```bash
   $ kubectl set image deployment/api api=taskboard-api:1.1.0
   $ kubectl rollout status deployment/api
   $ kubectl get rs -l app=taskboard,tier=api
   ```

4. Comprobar la respuesta actualizada:
   ```bash
   $ kubectl port-forward svc/api 8080:8080
   $ curl -s http://localhost:8080/health
   # {"status":"ok","version":"1.1.0"}
   ```

5. Reversión instantánea (*Rollback*):
   ```bash
   $ kubectl rollout undo deployment/api
   $ kubectl rollout status deployment/api
   ```

6. Control de velocidad de despliegue:
   ```yaml
   strategy:
     type: RollingUpdate
     rollingUpdate:
       maxSurge: 1
       maxUnavailable: 0
   ```

---

#### 2. Despliegue Azul/Verde (*Blue-Green Deployment*)

Permite validar la versión nueva (verde) en el clúster real antes de conmutar el tráfico de producción desde la versión actual (azul).

| Paso | Entorno | Tráfico de producción | Estado |
| :--- | :--- | :--- | :--- |
| **1** | Azul (`v1.0.0`) activo | ✅ Producción estable | Versión actual |
| **2** | Verde (`v1.1.0`) desplegado | ❌ Pruebas internas | Esperando validación |
| **3** | Conmutar selector del Service | ✅ `v1.1.0` (Verde) | Cambio instantáneo |
| **4** | Eliminar versión Azul | ✅ Producción limpia | Finalizado |
| **5** | *(Opcional)* Reversión | ✅ `v1.0.0` (Azul) | *Rollback* inmediato |

*Tabla 16.1 – Flujo de trabajo en despliegues Blue-Green*

1. Etiquetar el despliegue actual como Azul:
   ```bash
   $ kubectl label deployment/api color=blue --overwrite
   ```

2. Desplegar la versión Verde en `k8s/api-green.yaml`:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: api-green
     labels:
       app: taskboard
       tier: api
       color: green
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: taskboard
         tier: api
         color: green
     template:
       metadata:
         labels:
           app: taskboard
           tier: api
           color: green
       spec:
         containers:
           - name: api
             image: taskboard-api:1.1.0
             imagePullPolicy: IfNotPresent
             ports:
               - containerPort: 8080
             envFrom:
               - configMapRef:
                   name: taskboard-config
               - secretRef:
                   name: taskboard-secrets
             readinessProbe:
               httpGet:
                 path: /health
                 port: 8080
               initialDelaySeconds: 2
               periodSeconds: 5
             livenessProbe:
               httpGet:
                 path: /health
                 port: 8080
               initialDelaySeconds: 5
               periodSeconds: 10
   ```
   *Figura 16.9: Manifiesto de la versión verde*

3. Aplicar y probar internamente la versión Verde:
   ```bash
   $ kubectl apply -f k8s/api-green.yaml
   $ kubectl rollout status deployment/api-green
   $ POD=$(kubectl get pod -l color=green -o jsonpath='{.items[0].metadata.name}')
   $ kubectl port-forward "$POD" 8085:8080
   $ curl -s http://localhost:8085/health
   ```

4. Conmutar el Service a los Pods Verdes (`k8s/api-service.yaml`):
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: api
   spec:
     type: ClusterIP
     selector:
       app: taskboard
       tier: api
       color: green
     ports:
       - port: 8080
         targetPort: 8080
   ```
   Aplicar la conmutación:
   ```bash
   $ kubectl apply -f k8s/api-service.yaml
   $ kubectl get endpoints api -w
   ```

5. Desmantelar la versión Azul antigua:
   ```bash
   $ kubectl delete deployment api
   ```

---

### Mejores prácticas de seguridad en Kubernetes

#### 1. Ejecución como usuario no root y sistema de archivos de solo lectura
Modificar los Dockerfiles para crear usuarios no privilegiados:
- En `task-board/api/Dockerfile`:
  ```dockerfile
  RUN useradd -m -u 1000 -s /usr/sbin/nologin appuser \
      && chown -R 1000:1000 /app
  USER 1000:1000
  ```
- En `task-board/frontend/Dockerfile`:
  ```dockerfile
  RUN adduser -D -u 10100 web && chown -R web:web \
      /var/cache/nginx /var/run /var/log/nginx /usr/share/nginx/html
  USER 10100
  ```

Configurar el `securityContext` en `k8s/api.yaml`:
```yaml
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            runAsUser: 1000
            capabilities:
              drop:
                - ALL
          env:
            - name: GUNICORN_CMD_ARGS
              value: "--worker-tmp-dir /tmp"
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```
*Figura 16.10 y Figura 16.11: securityContext y volumen temporal para la API*

Configurar el `securityContext` en `k8s/frontend.yaml`:
```yaml
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            runAsUser: 10100
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: var-cache
              mountPath: /var/cache/nginx
            - name: var-run
              mountPath: /var/run
      volumes:
        - name: tmp
          emptyDir: {}
        - name: var-cache
          emptyDir: {}
        - name: var-run
          emptyDir: {}
```
*Figura 16.12 y Figura 16.13: securityContext y volúmenes temporales para NGINX*

#### 2. Menor privilegio con ServiceAccount y RBAC (`k8s/rbac-api.yaml`)
Crea `k8s/rbac-api.yaml` para limitar los permisos de la API en el clúster:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-sa
  namespace: taskboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: api-read-config
  namespace: taskboard
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-read-config-binding
  namespace: taskboard
subjects:
  - kind: ServiceAccount
    name: api-sa
    namespace: taskboard
roleRef:
  kind: Role
  name: api-read-config
  apiGroup: rbac.authorization.k8s.io
```
*Figura 16.14: Definición de ServiceAccount, Role y RoleBinding*

Vincular en `k8s/api.yaml` bajo `spec`:
```yaml
    spec:
      serviceAccountName: api-sa
      containers:
        - name: api
```

#### 3. Restricción a nivel de Namespace (*Pod Security Admission*)
```bash
$ kubectl label namespace taskboard \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted \
  --overwrite
```

#### 4. Aislamiento de tráfico con NetworkPolicies
1. **Denegación predeterminada total (`k8s/net-default-deny.yaml`)**:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: default-deny-all
     namespace: taskboard
   spec:
     podSelector: {}
     policyTypes:
       - Ingress
       - Egress
   ```
   *Figura 16.15: Política Default Deny total*

2. **Permitir Frontend -> API (`k8s/net-allow-frontend-to-api.yaml`)**:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-frontend-to-api
     namespace: taskboard
   spec:
     podSelector:
       matchLabels:
         app: taskboard
         tier: api
     policyTypes:
       - Ingress
     ingress:
       - from:
           - podSelector:
               matchLabels:
                 app: taskboard
                 tier: frontend
         ports:
           - protocol: TCP
             port: 8080
   ```
   *Figura 16.16: Permiso de entrada hacia la API solo desde el Frontend*

3. **Permitir API -> Base de datos (`k8s/net-allow-api-to-db.yaml`)**:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-api-to-db
     namespace: taskboard
   spec:
     podSelector:
       matchLabels:
         app: taskboard
         tier: db
     policyTypes:
       - Ingress
     ingress:
       - from:
           - podSelector:
               matchLabels:
                 app: taskboard
                 tier: api
         ports:
           - protocol: TCP
             port: 5432
   ---
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-api-egress-to-db
     namespace: taskboard
   spec:
     podSelector:
       matchLabels:
         app: taskboard
         tier: api
     policyTypes:
       - Egress
     egress:
       - to:
           - podSelector:
               matchLabels:
                 app: taskboard
                 tier: db
         ports:
           - protocol: TCP
             port: 5432
   ```
   *Figura 16.17: Permisos de comunicación exclusiva entre API y DB*

4. **Permitir resolución DNS (`k8s/net-allow-dns.yaml`)**:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-dns-egress
     namespace: taskboard
   spec:
     podSelector: {}
     policyTypes:
       - Egress
     egress:
       - to:
           - namespaceSelector: {}
         ports:
           - protocol: UDP
             port: 53
           - protocol: TCP
             port: 53
   ```
   *Figura 16.18: Permitir consultas DNS en el clúster*

5. Aplicar políticas:
   ```bash
   $ kubectl apply -f k8s/net-default-deny.yaml \
     -f k8s/net-allow-frontend-to-api.yaml \
     -f k8s/net-allow-api-to-db.yaml \
     -f k8s/net-allow-dns.yaml
   ```

#### 5. Montaje de secretos como archivos y rotación segura
Montar los secretos como archivos en memoria `/run/secrets/` en lugar de variables `ENV`:
```yaml
          volumeMounts:
            - name: secrets
              mountPath: /run/secrets
              readOnly: true
            - name: tmp
              mountPath: /tmp
          env:
            - name: DB_USER_FILE
              value: /run/secrets/DB_USER
            - name: DB_PASSWORD_FILE
              value: /run/secrets/DB_PASSWORD
      volumes:
        - name: secrets
          secret:
            secretName: taskboard-secrets
        - name: tmp
          emptyDir: {}
```
*Figura 16.20: Montaje seguro de Secrets como archivos*

Actualización del código en `task-board/api/server.py`:
```python
def read_secret(var_name, file_var_name):
    file_path = os.getenv(file_var_name)
    if file_path and os.path.exists(file_path):
        with open(file_path, "r") as f:
            return f.read().strip()
    return os.getenv(var_name)

db_user = read_secret("DB_USER", "DB_USER_FILE")
db_password = read_secret("DB_PASSWORD", "DB_PASSWORD_FILE")
```
*Figura 16.21: Lógica de lectura de credenciales desde archivo en memoria*

Reinicio controlado tras rotación de credenciales:
```bash
$ kubectl create secret generic taskboard-secrets \
  --from-literal=DB_USER=taskuser \
  --from-literal=DB_PASSWORD=newpass \
  -n taskboard --dry-run=client -o yaml | kubectl apply -f -

$ kubectl rollout restart deploy/api -n taskboard
$ kubectl rollout status deploy/api -n taskboard
```

---

### Resumen

En este capítulo aprendimos:
- A desplegar aplicaciones multi-servicio declarativas integrando ConfigMaps, Secrets, PVCs, Deployments y Services.
- A proteger la disponibilidad del servicio mediante **sondas Liveness, Readiness y Startup**.
- A ejecutar actualizaciones continuas (*Rolling Updates*) y conmutaciones *Blue-Green* sin tiempo de inactividad.
- A aplicar defensas en profundidad: usuarios `non-root`, sistemas de archivos de solo lectura, políticas de red (*NetworkPolicies*) y el estándar *Restricted Pod Security*.
- A gestionar secretos montándolos como volúmenes en memoria y orquestar su rotación controlada.

---

### Lecturas recomendadas

- **Documentación de Kubernetes – Despliegues y sondas**: [https://kubernetes.io/docs/concepts/workloads/controllers/deployment/](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- **Guía de seguridad en Kubernetes**: [https://kubernetes.io/docs/concepts/security/overview/](https://kubernetes.io/docs/concepts/security/overview/)
- **Recetas de NetworkPolicies con Cilium y Calico**: [https://docs.cilium.io/en/stable/policy/language/](https://docs.cilium.io/en/stable/policy/language/) y [https://docs.projectcalico.org/security/calico-network-policy](https://docs.projectcalico.org/security/calico-network-policy)
- **Metodología The Twelve-Factor App**: [https://12factor.net/](https://12factor.net/)
- **OWASP Kubernetes Security Cheat Sheet**: [https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html)
- **Configuración de Pod Security Admission (PSA)**: [https://kubernetes.io/docs/concepts/security/pod-security-admission/](https://kubernetes.io/docs/concepts/security/pod-security-admission/)

---

### Preguntas

1. **¿Cuál es el propósito principal de definir sondas de preparación (*Readiness*) y de vida (*Liveness*) en Kubernetes?**
2. **¿Cuál es la diferencia fundamental entre una actualización continua (*Rolling Update*) y un despliegue Azul/Verde (*Blue-Green*)?**
3. **¿Qué recurso de Kubernetes garantiza que un número específico de réplicas de un Pod se encuentren siempre en ejecución?**
4. **¿Cómo puedes observar el comportamiento de autorreparación de Kubernetes en la práctica?**
5. **¿Por qué los contenedores en Kubernetes deben ejecutarse como usuarios no root?**
6. **¿Cuál es la función de una ServiceAccount en Kubernetes y cómo se relaciona con RBAC?**
7. **¿Cómo mejora una NetworkPolicy la seguridad de un clúster de Kubernetes?**
8. **¿Por qué montar Secrets como archivos es más seguro que exponerlos como variables de entorno?**
9. **¿Cómo decide Kubernetes cuándo reemplazar Pods antiguos durante una actualización continua?**
10. **¿Qué ocurre si cambia el valor de un Secret o ConfigMap utilizado por un Pod en ejecución?**

---

### Respuestas

1. **Propósito de las sondas**:  
   La sonda de preparación comprueba si la aplicación puede atender tráfico; si falla, retira el Pod del balanceador sin reiniciarlo. La sonda de vida comprueba si el proceso sigue funcionando; si falla o se bloquea, Kubernetes reinicia el contenedor automáticamente.

2. **Rolling Update frente a Blue-Green**:  
   *Rolling Update* reemplaza réplicas gradualmente una a una manteniendo disponibilidad durante el proceso. *Blue-Green* ejecuta dos entornos completos en paralelo y conmuta el 100% del tráfico instantáneamente mediante el selector del Service tras completar las pruebas.

3. **Garantía de réplicas**:  
   El recurso **ReplicaSet** (gestionado por el Deployment) mantiene y reconcilia continuamente el número de réplicas deseadas.

4. **Observación de autorreparación**:  
   Terminando forzosamente el proceso principal de un Pod (`kill 1`). El ReplicaSet detecta la discrepancia inmediatamente y programa un nuevo Pod de reemplazo en cuestión de segundos.

5. **Importancia de usuarios non-root**:  
   Minimiza el radio de explosión ante un escape del contenedor, impidiendo que un atacante obtenga privilegios de superusuario en el nodo host o acceda a recursos administrativos del clúster.

6. **ServiceAccounts y RBAC**:  
   Una ServiceAccount otorga una identidad a los procesos dentro del Pod. Mediante Roles y RoleBindings de RBAC, se asignan permisos mínimos y específicos (e.g., solo lectura de ConfigMaps) aplicando el principio de menor privilegio.

7. **Rol de las NetworkPolicies**:  
   Establecen un cortafuegos interno basado en etiquetas. Al aplicar una política de denegación por defecto (*default deny*) y autorizaciones explícitas, se impide el movimiento lateral de atacantes entre microservicios no autorizados.

8. **Seguridad de Secrets montados como archivos**:  
   Los archivos montados residen en volúmenes en memoria (`tmpfs`) y no aparecen en listas de procesos (`ps`), volcados de memoria, logs de depuración ni inspecciones estáticas de contenedores.

9. **Mecanismo de Rolling Update**:  
   Utiliza los parámetros `maxSurge` y `maxUnavailable` junto con la sonda de preparación (*Readiness Probe*). Kubernetes espera a que el nuevo Pod reporte estado `Ready` antes de enviar la señal de terminación al Pod de la versión anterior.

10. **Actualización de Secrets en caliente**:  
    Kubernetes no reinicia automáticamente los Pods cuando cambia un Secret o ConfigMap. Se debe desencadenar un reinicio controlado con `kubectl rollout restart deploy/<nombre>` o utilizar anotaciones con *hash/checksum* en la plantilla del Pod.

