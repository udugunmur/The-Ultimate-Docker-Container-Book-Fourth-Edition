# Parte 4: Docker, Kubernetes y la Nube
## Capítulo 20: Patrones de contenerización en el mundo real

Hemos llegado al capítulo final de este libro. Cuando comenzamos este viaje, los contenedores eran un concepto abstracto. Exploramos qué son, por qué son fundamentales y cómo eliminan la fricción en la cadena de suministro de software. A partir de ahí, aprendimos a construir y gestionar imágenes, a trabajar con datos y configuraciones, a depurar y probar aplicaciones dentro de contenedores, y a evolucionar progresivamente desde arquitecturas de un solo contenedor hasta sistemas distribuidos complejos.

Posteriormente nos adentramos en la orquestación con Kubernetes, aprendiendo a desplegar, escalar, asegurar, monitorizar y resolver incidencias en aplicaciones que se ejecutan en clústeres. A lo largo del camino, examinamos cómo la automatización y las herramientas inteligentes potencian las prácticas de DevOps y elevan la fiabilidad en todo el ciclo de entrega.

En este punto, los contenedores ya no deben sentirse como una herramienta experimental, sino como una parte natural de cómo se construye y opera el software moderno. Más importante aún, has comprobado que la contenerización no trata únicamente de empaquetar código: **trata de consistencia, automatización, observabilidad y responsabilidad compartida**. Las herramientas específicas seguirán evolucionando, pero los principios perdurarán: *construye una vez, ejecuta de manera consistente, automatiza todo lo que sea automatizable y mejora continuamente*.

En este capítulo final, nos alejamos de las técnicas individuales para analizar los **patrones arquitectónicos recurrentes** que surgen una y otra vez en organizaciones reales de diferentes industrias y sectores:
1. **Modernización de sistemas heredados (*Legacy*)**: Transformación de monolitos en despliegues contenerizados.
2. **Microservicios en acción**: Descomposición de sistemas complejos en servicios escalables y autónomos.
3. **CI/CD en la práctica**: Aceleración y estabilización de la entrega continua mediante contenedores.
4. **Patrones de migración a la nube**: Despliegue de cargas de trabajo en AWS, Azure y GCP con consistencia declarativa.
5. **Orquestación en producción**: Operación fiable de Kubernetes a gran escala.
6. **Seguridad integral del contenedor**: Aplicación de defensas prácticas en todo el ciclo de vida (*Build, Ship, Run*).

Cada sección sigue una estructura clara:
- **Contexto**: ¿Cuál es el desafío recurrente al que se enfrenta la organización?
- **Solución**: ¿Cómo se aplican Docker y Kubernetes para resolverlo?
- **Lecciones clave (*Key Learnings*)**: ¿Qué aprendizajes prácticos emergen de la experiencia real?

---

### Modernización de sistemas heredados: De monolitos a contenedores

#### Contexto
Casi todas las iniciativas de modernización comienzan con un sistema preexistente: un monolito grande y fuertemente acoplado que ha crecido durante años. Estas aplicaciones dan soporte a procesos de negocio críticos, pero son frágiles, difíciles de escalar y lentas de desplegar. Cada cambio conlleva un riesgo elevado de regresión en producción y los entornos de desarrollo, pruebas y producción sufren de divergencia de configuración (*configuration drift*).

La contenerización ofrece una oportunidad estratégica: empaquetar la aplicación existente para aislar dependencias y estandarizar el entorno de ejecución, permitiendo avanzar hacia una arquitectura modular sin necesidad de reescribir todo el sistema desde cero.

#### Solución
La modernización se estructura habitualmente en tres fases evolutivas:

```
[ Fase 1: Lift and Shift ]  --->  [ Fase 2: Refactorización ]  --->  [ Fase 3: Rediseño Arquitectónico ]
 (Empaquetar sin cambios)          (Separar tareas y config)            (Microservicios y Kubernetes)
```

##### Fase 1: *Lift and Shift* (Rehospedaje)
El objetivo es lograr la paridad de entornos empaquetando la aplicación existente (Java WAR/JAR, .NET, Python) tal como está dentro de un contenedor:
```dockerfile
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/legacy-app.jar .
CMD ["java", "-jar", "legacy-app.jar"]
```
Esta fase aporta beneficios inmediatos: entornos idénticos, compilaciones reproducibles y la capacidad de ejecutar la aplicación en cualquier infraestructura sin modificar el código fuente.

##### Fase 2: Refactorización (*Refactor*)
Una vez que el monolito se ejecuta de forma fiable en contenedores, se desacoplan componentes secundarios: tareas en segundo plano (*background workers*), servicios de exportación o endpoints de API independientes. La configuración y los logs se externalizan mediante variables de entorno y volúmenes, y las bases de datos se conectan a través de redes de Docker.

##### Fase 3: Rediseño (*Rearchitect*)
El sistema evoluciona hacia una arquitectura orientada a servicios orquestada por Kubernetes. Los scripts manuales dan paso a manifiestos declarativos y Helm charts, habilitando autoescalado horizontal y actualizaciones continuas sin tiempo de inactividad.

#### Lecciones clave
- **Comenzar conteniendo, no transformando**: El primer hito es lograr una imagen que funcione de forma idéntica al sistema heredado. Evita la tentación de rediseñar todo desde el primer día.
- **Externalizar la configuración desde el inicio**: Mantén credenciales y parámetros fuera de las imágenes mediante ConfigMaps y Secrets.
- **Integrar observabilidad tempranamente**: Añade logs estructurados y métricas básicas desde las primeras fases para simplificar la depuración cuando se divida el monolito.
- **La arquitectura sigue a la cultura**: La transición técnica hacia contenedores debe ir acompañada de una evolución organizativa hacia la responsabilidad compartida (*DevOps*).

---

### Microservicios en acción: Descomposición y escalado de aplicaciones complejas

#### Contexto
Cuando un monolito exitoso crece, se convierte en un cuello de botella organizativo: los despliegues tardan horas, coordinar lanzamientos entre múltiples equipos resulta caótico y escalar exige replicar la aplicación completa incluso si solo una función específica experimenta alta demanda.

#### Solución
La transición a microservicios divide el sistema en capacidades de negocio independientes (*Bounded Contexts*) guiadas por principios de *Domain-Driven Design* (DDD) o *EventModeling*:

```
                           +------------------------+
                           |   API Gateway / Ingress|
                           +------------------------+
                                  |          |
                 +----------------+          +----------------+
                 v                                            v
     +-----------------------+                    +-----------------------+
     | orders-service (8081) |                    |payments-service (8082)|
     +-----------------------+                    +-----------------------+
                 |                                            |
         [( Base de Datos )]                          [( Base de Datos )]
         (  Exclusiva de  )                           (  Exclusiva de  )
         (    Órdenes     )                           (     Pagos      )
```

1. **Desarrollo y prueba local con Docker Compose**:
   ```yaml
   services:
     orders:
       build: ./orders
       ports:
         - "8081:8080"
     payments:
       build: ./payments
       ports:
         - "8082:8080"
     frontend:
       build: ./frontend
       ports:
         - "80:80"
   ```

2. **Descubrimiento de servicios y balanceo en Kubernetes**:  
   Cada microservicio se gestiona mediante un Deployment y un Service de Kubernetes, aprovechando el DNS interno para la resolución automática de nombres y el balanceo de carga.

3. **Propiedad estricta de los datos**:  
   Cada servicio debe poseer su propio esquema de datos. Compartir bases de datos relacionales entre microservicios recrea el acoplamiento del monolito a nivel de almacenamiento.

4. **Comunicación y observabilidad**:  
   Comenzar con llamadas síncronas REST/gRPC y evolucionar hacia mensajería asíncrona desacoplada (Kafka, RabbitMQ, NATS) respaldada por trazas distribuidas con OpenTelemetry.

#### Lecciones clave
- **La descomposición debe ser gradual**: Aplica la *Primera Ley del Diseño de Aplicaciones Distribuidas* de Martin Fowler: *"¡No distribuyas tus objetos a menos que sea estrictamente necesario!"*. Los sistemas distribuidos introducen latencia de red y nuevos modos de fallo.
- **La automatización no es negociable**: Multiplicar el número de artefactos desplegables exige canalizaciones de CI/CD robustas para evitar que la sobrecarga operativa sea inmanejable.
- **Observabilidad es control**: Logs centralizados, métricas de rendimiento y trazas de extremo a extremo son indispensables para diagnosticar fallos en sistemas distribuidos.

---

### CI/CD en la práctica: Aceleración de la entrega continua con contenedores

#### Contexto
Construir imágenes manualmente y desplegarlas de forma artesanal genera inconsistencias y largos ciclos de retroalimentación. Un pipeline de CI/CD moderno garantiza que cada cambio en el código fuente sea validado, probado y desplegado utilizando **exactamente la misma imagen inmutable** que llegará a producción.

#### Solución

```
[ Código en Git ] ---> [ Build Imagen ] ---> [ Tests en Contenedor ] ---> [ Push a Registro (Digest) ] ---> [ Deploy Kubernetes ]
```

1. **Definición del pipeline como código (GitHub Actions)**:
   ```yaml
   name: build-and-deploy
   on:
     push:
       branches: [ main ]
   jobs:
     build:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - name: Build Docker image
           run: docker build -t myapp:${{ github.sha }} .
         - name: Push to registry
           run: |
             docker tag myapp:${{ github.sha }} \
               ghcr.io/myorg/myapp:${{ github.sha }}
             docker push ghcr.io/myorg/myapp:${{ github.sha }}
     deploy:
       needs: build
       runs-on: ubuntu-latest
       steps:
         - name: Deploy to Kubernetes
           run: |
             kubectl set image deployment/myapp \
               myapp=ghcr.io/myorg/myapp:${{ github.sha }}
   ```

2. **Ejecución de pruebas dentro del contenedor**:
   ```yaml
   - name: Run tests
     run: docker run --rm myapp:${{ github.sha }} pytest
   ```

3. **Promoción inmutable mediante *Image Digest***:  
   Las etiquetas (*tags*) como `latest` o `v1.2.3` son mutables. El uso del hash criptográfico SHA-256 (*digest*) garantiza que el artefacto desplegado en producción sea idéntico al validado en pruebas:
   ```bash
   $ kubectl set image deployment/myapp myapp=\
       ghcr.io/myorg/myapp@sha256:<digest>
   ```

   Obtención del digest:
   ```bash
   $ docker push ghcr.io/myorg/myapp:${GITHUB_SHA}
   # Output:
   # digest: sha256:8a4b9c4e89a3c2...

   $ docker inspect --format='{{index .RepoDigests 0}}' myapp

   $ DIGEST=$(docker inspect \
     --format='{{index .RepoDigests 0}}' \
     ghcr.io/myorg/myapp:${{ github.sha }})
   $ kubectl set image deployment/myapp myapp=$DIGEST
   ```

#### Lecciones clave
- **Las canalizaciones son infraestructura**: Versiona, prueba y revisa tus definiciones de CI/CD con la misma rigurosidad que el código de la aplicación.
- **Desplazar la seguridad a la izquierda (*Shift-Left*)**: Integra escaneo de vulnerabilidades y firma criptográfica de imágenes directamente en el pipeline.
- **Inmutabilidad absoluta**: Promociona las versiones entre entornos haciendo referencia al digest SHA-256, nunca reconstruyendo la imagen para producción.

---

### Patrones de migración a la nube: AWS, Azure y Google Cloud

#### Contexto
Las organizaciones migran a la nube en busca de elasticidad, presencia global o resiliencia multicloud. La contenerización proporciona una capa de abstracción universal que desacopla la aplicación de la infraestructura subyacente.

#### Solución: Despliegue multiplataforma con Helm y manifiestos estándar

##### 1. Amazon Web Services (AWS EKS con `eksctl`)
```bash
$ eksctl create cluster \
  --name app-prod \
  --region eu-central-1 \
  --nodes 3 \
  --node-type m5.large

$ aws eks update-kubeconfig \
  --region eu-central-1 \
  --name app-prod

$ kubectl create secret docker-registry ghcr-creds \
  --docker-server=ghcr.io \
  --docker-username=$GITHUB_ACTOR \
  --docker-password=$GH_TOKEN \
  --namespace app

$ helm upgrade --install myapp charts/myapp \
  --namespace app --create-namespace \
  --set image.repo=ghcr.io/myorg/myapp \
  --set image.tag=${GITHUB_SHA}
```

##### 2. Microsoft Azure (Azure AKS con Azure CLI)
```bash
$ az group create -n rg-app -l westeurope
$ az aks create -g rg-app -n aks-app \
  --node-count 3 --generate-ssh-keys

$ az aks get-credentials -g rg-app -n aks-app

$ kubectl create secret docker-registry ghcr-creds \
  --docker-server=ghcr.io \
  --docker-username=$GITHUB_ACTOR \
  --docker-password=$GH_TOKEN \
  --namespace app

$ helm upgrade --install myapp charts/myapp \
  --namespace app --create-namespace \
  --set image.repo=ghcr.io/myorg/myapp \
  --set image.tag=${GITHUB_SHA}
```

##### 3. Google Cloud Platform (Google GKE con Google Cloud SDK)
```bash
$ gcloud container clusters create gke-app \
  --region europe-west6 \
  --num-nodes 3

$ gcloud container clusters get-credentials gke-app \
  --region europe-west6

$ kubectl create secret docker-registry ghcr-creds \
  --docker-server=ghcr.io \
  --docker-username=$GITHUB_ACTOR \
  --docker-password=$GH_TOKEN \
  --namespace app

$ helm upgrade --install myapp charts/myapp \
  --namespace app --create-namespace \
  --set image.repo=ghcr.io/myorg/myapp \
  --set image.tag=${GITHUB_SHA}
```

##### Despliegue universal por Digest
```bash
$ IMG=ghcr.io/myorg/myapp:${GITHUB_SHA}
$ docker push $IMG
$ DIGEST=$(docker inspect \
  --format='{{index .RepoDigests 0}}' $IMG)
$ helm upgrade --install myapp charts/myapp \
  --namespace app --create-namespace \
  --set image.ref="$DIGEST"
```

#### Lecciones clave
- **Construir una vez, ejecutar en cualquier nube**: Utiliza el mismo Helm chart y la misma imagen de contenedor, parametrizando únicamente los valores específicos del entorno.
- **Identidad gestionada frente a credenciales estáticas**: Reemplaza contraseñas fijas por roles IAM federados (AWS IRSA, Azure Workload Identity, Google Workload Identity).

---

### Orquestación en producción: Lecciones operativas de Kubernetes

#### Contexto
A medida que Kubernetes pasa de ser un entorno experimental al núcleo de la infraestructura corporativa, el desafío principal se traslada a mantener la estabilidad, la gobernanza y la resiliencia operativa a gran escala.

#### Solución

##### 1. Tratar el clúster como una plataforma interna (*Platform Engineering*)
El equipo de plataforma establece las directrices globales (red, seguridad, cuotas y monitorización) y los equipos de desarrollo despliegan sus cargas mediante flujos automatizados de GitOps sin acceso manual directo a los clústeres.

##### 2. Operaciones declarativas con GitOps (Argo CD)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/platform
    path: charts/myapp
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

##### 3. Monitorización automatizada con Prometheus Operator (`PodMonitor`)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  podMetricsEndpoints:
  - port: http
    path: /metrics
```

##### 4. Diseñar para el fallo (*Design for Failure*)
Definir sondas de vida y preparación en todos los despliegues para garantizar la autorreparación y evitar enviar tráfico a instancias no preparadas:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 30

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

#### Lecciones clave
- **Kubernetes es una plataforma, no un producto final**: Proporciona los bloques fundamentales para construir servicios de autoservicio para los desarrolladores.
- **La automatización sin observabilidad genera fragilidad**: La capacidad de autorreparación de Kubernetes solo es efectiva si se acompaña de telemetría completa.
- **GitOps como única fuente de verdad**: Lo que está declarado en el repositorio de Git es exactamente lo que debe ejecutarse en el clúster.

---

### Seguridad integral del contenedor: Estrategias en el ciclo de vida

#### Contexto
Los contenedores no son intrínsecamente seguros por el mero hecho de aislar procesos. Imágenes base obsoletas, ejecución como usuario root, secretos incrustados en capas de imagen o permisos excesivos de RBAC generan vulnerabilidades graves en producción.

#### Solución: Defensas en profundidad (*Build, Ship, Run*)

```
[ BUILD ] ---------------------> [ SHIP ] ---------------------> [ RUN ]
- Base mínima (distroless/slim)  - Firma con Cosign              - SecurityContext (Non-root, Drop ALL)
- Usuario no root                - Escaneo en CI (Trivy/Grype)   - Políticas de admisión (Kyverno/PSS)
- Sin secretos en capas          - Registro inmutable            - Monitorización en tiempo real (Falco)
```

##### 1. Construcción segura (Dockerfile con usuario sin privilegios)
```dockerfile
FROM python:3.12-slim
RUN adduser --disabled-password appuser
WORKDIR /app
COPY . .
USER appuser
CMD ["python", "main.py"]
```

##### 2. Escaneo automatizado de vulnerabilidades en CI (Trivy)
```bash
$ trivy image --exit-code 1 --severity HIGH,CRITICAL \
  ghcr.io/myorg/myapp:${GITHUB_SHA}
```

##### 3. Firma y verificación criptográfica de imágenes (Cosign)
```bash
# Firmar imagen tras superar pruebas
$ cosign sign --key cosign.key \
  ghcr.io/myorg/myapp:${GITHUB_SHA}

# Verificar antes del despliegue
$ cosign verify --key cosign.pub \
  ghcr.io/myorg/myapp:${GITHUB_SHA}
```

##### 4. Integración con gestores externos de secretos (External Secrets Operator)
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-store
    kind: SecretStore
  target:
    name: db-credentials
  data:
  - secretKey: username
    remoteRef:
      key: prod/db
      property: username
  - secretKey: password
    remoteRef:
      key: prod/db
      property: password
```

##### 5. Aplicación del principio de menor privilegio en Pods (`securityContext`)
```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

##### 6. Control de admisión declarativo con Kyverno
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: enforce
  rules:
  - name: no-privileged
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Privileged containers are not allowed"
      pattern:
        spec:
          containers:
          - securityContext:
              privileged: false
```

#### Lecciones clave
- **La seguridad comienza en el Dockerfile**: Imágenes base mínimas, versiones fijas y ejecución estricta como usuario `non-root`.
- **Eliminar la dispersión de secretos**: Prohíbe almacenar credenciales en Git o en capas de imagen; sincronízalas desde almacenes seguros de secretos.
- **Control de admisión continuo**: Políticas declarativas con Kyverno o PSS impiden que se desplieguen contenedores privilegiados en el clúster.

---

### Resumen del libro

La seguridad y la fiabilidad son el hilo conductor que une cada etapa del ciclo de vida de los contenedores. Los contenedores no son simplemente una tecnología para empaquetar software: **son la base de la predictibilidad, la reproducibilidad y la resiliencia en la ingeniería moderna de software**.

El camino recorrido refleja la evolución de la propia industria: desde la creación de contenedores individuales y redes locales, pasando por la orquestación y la automatización inteligente, hasta la operación a escala global y la seguridad integral en la nube.

---

### Epílogo: Reflexiones finales

Hemos recorrido un largo camino juntos. Desde comprender qué son los contenedores y por qué son fundamentales, hasta construir, probar y operar aplicaciones distribuidas en Kubernetes, cada capítulo ha representado un paso hacia el dominio de la entrega moderna de software.

El verdadero valor de los contenedores no es solo técnico, sino también **cultural**: impulsan a los equipos a diseñar en unidades desacopladas y manejables, a automatizar incansablemente, a construir confianza a través de la reproducibilidad y a asumir la propiedad de extremo a extremo sobre sus aplicaciones.

El ecosistema cloud-native seguirá evolucionando. Surgirán nuevas abstracciones y herramientas, pero los principios practicados a lo largo de este libro —inmutabilidad, automatización, observabilidad y seguridad— permanecerán inmutables como los cimientos sobre los que se construye cualquier sistema resiliente.

---

### Próximos pasos

Tu viaje de aprendizaje no termina aquí. Las siguientes direcciones te permitirán profundizar tu maestría técnica:
- **Contribuir a proyectos Open Source**: Explora repositorios de Kubernetes, Prometheus, Docker o Argo CD.
- **Dominar GitOps y Platform Engineering**: Implementa plataformas de autoservicio para desarrolladores con Argo CD o Flux.
- **Profundizar en la seguridad de la cadena de suministro**: Explora marcos como SLSA, generación de SBOMs (*Software Bill of Materials*) y atestaciones con Cosign.
- **Avanzar en extensiones de Kubernetes**: Desarrolla operadores personalizados con Custom Resource Definitions (CRDs) y mallas de servicios (*Service Meshes* como Istio o Linkerd).
- **Compartir el conocimiento**: Escribe artículos técnicos, imparte talleres y guía a otros ingenieros en la adopción de las mejores prácticas.

---

### Nota del autor

> Escribir este libro ha sido un viaje tanto profesional como personal. A lo largo de los años, he visto a los contenedores evolucionar desde una conveniente herramienta para desarrolladores hasta convertirse en una de las tecnologías más transformadoras de la ingeniería de software moderna.
>
> Durante mi carrera, he tenido el privilegio de trabajar con ingenieros extraordinarios que me enseñaron que el éxito en tecnología no depende de las herramientas en sí, sino de la colaboración, la curiosidad y el rigor artesanal en la ingeniería. Espero que este libro te haya transmitido ese mismo espíritu.
>
> Si hay un mensaje final que me gustaría dejarte, es este: **nunca dejes de aprender y nunca dejes de compartir lo que aprendes**. Construye con cuidado. Automatiza con propósito. Y diseña sistemas que hagan que el cambio sea seguro y predecible.
>
> Gracias por leer, por experimentar y por recorrer este camino conmigo.
>
> *¡Que tus clústeres se mantengan saludables, tus compilaciones sean siempre reproducibles y tus despliegues transcurran sin incidentes!*
>
> — **Gabriel N. Schenker**
