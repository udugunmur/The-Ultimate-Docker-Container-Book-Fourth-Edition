# Parte 3: Fundamentos de la Orquestación
## Capítulo 13: Seguridad en contenedores

En capítulos anteriores aprendimos a construir, probar, desplegar y monitorizar aplicaciones contenerizadas. Sin embargo, a medida que la adopción de contenedores se expande en las organizaciones, también lo hace la superficie de ataque. Un solo error de configuración o una imagen base desactualizada y vulnerable pueden comprometer toda la infraestructura.

En este capítulo, aprenderemos a **proteger nuestras aplicaciones contenerizadas a lo largo de todo su ciclo de vida**, desde la creación de imágenes hasta su ejecución en tiempo de producción. Abordaremos la seguridad de la cadena de suministro (*supply chain security*), la generación de inventarios **SBOM**, el escaneo de vulnerabilidades con **Trivy**, la firma digital y verificación criptográfica con **Cosign**, las prácticas de endurecimiento (*hardening*) del contenedor, la gestión segura de secretos y la detección de anomalías en tiempo de ejecución con **Falco**.

---

### Temas tratados en este capítulo:
- Seguridad en la cadena de suministro de software (*Supply Chain Security*)
- Escaneo de vulnerabilidades en imágenes y confianza de contenido (*Content Trust*)
- Prácticas de endurecimiento de contenedores (*Container Hardening*)
- Gestión segura de secretos (*Secrets Management*)
- Herramientas de seguridad en tiempo de ejecución (*Runtime Security*) con Falco

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Generar y mantener un Software Bill of Materials (SBOM) con **Syft** para auditar los componentes de tus imágenes.
- Analizar imágenes en busca de CVEs con **Trivy** e implementar controles de bloqueo en CI/CD.
- Firmar digitalmente imágenes y verificar su autenticidad e inmutabilidad con **Cosign**.
- Aplicar técnicas de endurecimiento: usuarios sin privilegios (`non-root`), sistemas de archivos de solo lectura (`--read-only`), eliminación de capacidades del kernel (`--cap-drop ALL`) y límites de recursos.
- Inyectar credenciales y certificados de forma segura sin exponerlos en imágenes ni variables de entorno.
- Desplegar **Falco** para monitorizar llamadas al sistema del kernel de Linux y alertar sobre comportamientos sospechosos en tiempo real.

---

### Requisitos técnicos

Necesitarás Docker Desktop o Docker Engine, junto con las herramientas **Syft**, **Trivy** y **Cosign** (y opcionalmente **Falco**).

Prepara la carpeta de trabajo:
```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir chapter-13 && cd chapter-13
```

#### Instalación de Ruby
- **En macOS (vía Homebrew)**:
  ```bash
  $ xcode-select --install
  $ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
  $ brew update && brew install ruby
  $ echo 'export PATH="/usr/local/opt/ruby/bin:$PATH"' >> ~/.zshrc
  $ source ~/.zshrc
  $ ruby -v && gem -v
  ```
- **En Windows (vía WSL2 / Ubuntu)**:
  ```bash
  $ sudo apt update
  $ sudo apt install -y build-essential libssl-dev libyaml-dev zlib1g-dev libgmp-dev ruby-full
  $ ruby -v && gem -v
  ```

#### Instalación de Trivy y Cosign
- **En macOS**:
  ```bash
  $ brew install trivy cosign
  $ trivy --version && cosign version
  ```
- **En Windows (PowerShell / WSL2)**:
  ```powershell
  PS> winget install --id=Sigstore.Cosign -e
  PS> choco install cosign
  PS> cosign version
  ```
  *(Para Trivy en Windows, se puede ejecutar como contenedor Docker)*:
  ```bash
  $ docker run --rm \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v ~/.cache/trivy:/root/.cache/trivy \
    aquasec/trivy image hello-ruby:latest
  ```

---

### Seguridad en la cadena de suministro de software (*Supply Chain Security*)

Un contenedor seguro comienza en la fase de compilación (*build time*). Cada imagen se compone de capas superpuestas que heredan dependencias de imágenes base públicas y repositorios de terceros, representando posibles vectores de ataque si contienen código malicioso o librerías desactualizadas.

#### Generación del Software Bill of Materials (SBOM)
Un **SBOM** es un manifiesto estructurado que cataloga todos los paquetes, librerías, dependencias y metadatos de licencias que componen una imagen de software. Es un requisito indispensable para auditorías de seguridad y marcos normativos como NIST SP 800-218 y la Orden Ejecutiva 14028 de EE.UU.

#### Instalación y uso de Syft:
- **macOS**: `brew install anchore/syft/syft`
- **Windows**: `winget install Anchore.Syft`
- **Linux**:
  ```bash
  $ curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh
  $ sudo mv syft /usr/local/bin/
  ```

1. Generar un inventario SBOM para una imagen:
   ```bash
   $ syft nginx:latest
   ```
   *Figura 13.1: Lista de componentes identificados en la imagen Nginx*

2. Exportar el SBOM en formato estandarizado CycloneDX JSON:
   ```bash
   $ syft nginx:latest -o cyclonedx-json > nginx-sbom.json
   ```

---

### Construcción, escaneo y firma de una aplicación de ejemplo (`hello-ruby`)

1. Crear el directorio del proyecto:
   ```bash
   $ mkdir hello-ruby && cd hello-ruby
   ```

2. Crear `Gemfile`:
   ```ruby
   source "https://rubygems.org"
   gem "sinatra"
   gem "puma"
   ```

3. Generar el archivo de bloqueo:
   ```bash
   $ bundle lock
   ```

4. Crear `config.ru`:
   ```ruby
   require_relative "app"
   run Sinatra::Application
   ```

5. Crear `app.rb`:
   ```ruby
   require "sinatra"

   set :bind, "0.0.0.0"
   set :port, 4567

   get "/" do
     "Hello, World! (from Ruby and Sinatra)"
   end
   ```
   *Figura 13.3: Código de la aplicación Ruby*

6. Crear `Dockerfile`:
   ```dockerfile
   FROM ruby:3.3-slim
   WORKDIR /app
   COPY Gemfile Gemfile.lock ./
   RUN gem install bundler && bundle install
   COPY . .
   EXPOSE 4567
   CMD ["ruby", "app.rb"]
   ```
   *Figura 13.4: Dockerfile de la aplicación Ruby*

7. Construir y ejecutar la imagen:
   ```bash
   $ docker build -t hello-ruby:latest .
   $ docker run --rm -p 4567:4567 hello-ruby:latest
   $ curl http://localhost:4567
   ```

8. Generar SBOM y escanear vulnerabilidades:
   ```bash
   $ syft hello-ruby:latest -o cyclonedx-json > hello-ruby-sbom.json
   $ trivy image hello-ruby:latest
   ```

---

### Firma y verificación criptográfica con Cosign

Para garantizar que una imagen proviene de una fuente autorizada y no ha sido alterada, la firmamos digitalmente con **Cosign**:

1. Etiquetar y publicar la imagen en un registro:
   ```bash
   $ docker image tag hello-ruby:latest <username>/hello-ruby:v1.0.0
   $ docker login -u <username>
   $ docker image push <username>/hello-ruby:v1.0.0
   ```

2. Generar el par de claves criptográficas:
   ```bash
   $ cosign generate-key-pair
   ```
   *(Genera `cosign.key` privada y `cosign.pub` pública).*

3. Firmar la imagen:
   ```bash
   $ cosign sign --key cosign.key <username>/hello-ruby:v1.0.0
   ```

4. Verificar la firma:
   ```bash
   $ cosign verify --key cosign.pub <username>/hello-ruby:v1.0.0
   ```
   Salida:
   ```text
   Verified OK : <username>/hello-ruby:v1.0.0
   ```

5. Añadir anotaciones de metadatos de compilación a la firma:
   ```bash
   $ cosign sign -a git-commit=abc123 --key cosign.key <username>/hello-ruby:v1.0.0
   ```

---

### Escaneo de vulnerabilidades con Trivy en pipelines CI/CD

**Trivy** analiza paquetes del sistema operativo y dependencias de lenguajes contra bases de datos de vulnerabilidades conocidas (CVEs).

1. Escaneo interactivo:
   ```bash
   $ trivy image hello-ruby:latest
   ```
   *Figura 13.5: Informe de vulnerabilidades generado por Trivy*

2. Bloqueo en CI/CD con código de salida ante fallos críticos:
   ```bash
   $ trivy image hello-ruby:latest \
     --severity CRITICAL,HIGH \
     --exit-code 1
   ```
   *(Si se detectan vulnerabilidades críticas o altas, el comando devuelve `exit 1` y cancela el pipeline).*

3. Exportación en JSON/SARIF para auditoría:
   ```bash
   $ trivy image hello-ruby:latest -f json -o trivy-result.json
   ```

---

### Prácticas de endurecimiento de contenedores (*Container Hardening*)

#### 1. Principio de menor privilegio: Usuario sin privilegios (`non-root`)
Por defecto, los contenedores se ejecutan como `root` (UID 0). Si un atacante escapa del contenedor, obtendría acceso de superusuario en el host.

```dockerfile
FROM ruby:3.3-slim
RUN groupadd -r appuser && useradd -r -g appuser appuser
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN gem install bundler && bundle install
COPY . .
RUN chown -R appuser:appuser /app
USER appuser
EXPOSE 4567
CMD ["ruby", "app.rb"]
```
*Figura 13.6: Configuración de usuario no root en Dockerfile*

#### 2. Eliminación de capacidades del kernel de Linux (`capabilities`)
Despoja al contenedor de todos los privilegios de bajo nivel del kernel y reasigna solo los estrictamente necesarios:
```bash
$ docker run --rm \
  -p 4567:4567 \
  --cap-drop ALL \
  --cap-add CHOWN \
  --cap-add NET_BIND_SERVICE \
  hello-ruby:latest
```

#### 3. Impedir escalada de privilegios (`--no-new-privileges`):
```bash
$ docker run --rm -p 4567:4567 --no-new-privileges hello-ruby:latest
```

#### 4. Sistema de archivos de solo lectura (`--read-only`):
Impide que un atacante descargue malware o modifique binarios dentro del contenedor:
```bash
$ docker run --rm \
  -p 4567:4567 \
  --read-only \
  -v /usr/src/app/logs \
  -v /usr/src/app/tmp \
  hello-ruby:latest
```

#### 5. Compilaciones multi-etapa (*Multi-stage builds*) e imágenes mínimas:
Garantiza que compiladores, código fuente y dependencias de compilación no lleguen a la imagen final de producción:

```dockerfile
# Stage 1: Build
FROM ruby:3.3-slim AS builder
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN apt-get update && apt-get install -y build-essential \
    && gem install bundler \
    && bundle install --without development test

# Stage 2: Runtime
FROM ruby:3.3-slim
RUN groupadd -r appuser && useradd -r -g appuser appuser
WORKDIR /app
COPY --from=builder /usr/local/bundle /usr/local/bundle
COPY . .
RUN chown -R appuser:appuser /app
USER appuser
EXPOSE 4567
CMD ["ruby", "app.rb"]
```
*Figura 13.7: Dockerfile multi-etapa*

#### 6. Seguridad a nivel de kernel (Seccomp y AppArmor):
```bash
$ docker run --rm \
  -p 4567:4567 \
  --security-opt seccomp=default.json \
  --security-opt apparmor=my-profile \
  hello-ruby:latest
```

#### 7. Comprobaciones de salud (*Health Checks*):
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:4567/ || exit 1
```

#### Ejemplo consolidado de Dockerfile endurecido:
```dockerfile
FROM ruby:3.3-slim AS builder
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
    && gem install bundler \
    && bundle install --without development test \
    && rm -rf /var/lib/apt/lists/*

FROM ruby:3.3-slim
RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser \
    && apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /usr/local/bundle /usr/local/bundle
COPY . .
RUN chown -R appuser:appuser /app
USER appuser
EXPOSE 4567
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:4567/ || exit 1
CMD ["ruby", "app.rb"]
```
*Figura 13.8: Dockerfile endurecido para hello-ruby*

Ejecución con flags de endurecimiento:
```bash
$ docker run --rm \
  -p 4567:4567 \
  --read-only \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --no-new-privileges \
  --memory 256m \
  --cpus 0.5 \
  hello-ruby:latest
```

---

### Compromisos y consideraciones en el endurecimiento (*Trade-offs*)

| Medida de endurecimiento | Beneficio de seguridad | Inconvenientes potenciales | Mitigación recomendada |
| :--- | :--- | :--- | :--- |
| **Ejecutar como no root / eliminar capabilities** | Reduce el impacto de un escape o vulnerabilidad | Ciertas tareas (enlazar puertos bajos `<1024`, chown) fallan | Identificar capabilities exactas (`--cap-drop ALL` y `--cap-add`) |
| **`--no-new-privileges`** | Evita escalada de privilegios vía `setuid` | Rompe procesos que legítimamente requieren elevar permisos | Usar por defecto; documentar excepciones |
| **Filesystem de solo lectura (`--read-only`)** | Evita inyección de malware y modificación de binarios | Falla en apps que escriben logs, temporales o cachés | Montar volúmenes específicos solo en `/tmp` o `/logs` |
| **Imágenes distroless / minimalistas** | Reduce drásticamente la superficie de ataque | Dificulta la depuración interactiva (sin shell ni package manager) | Mantener imágenes completas en desarrollo o usar contenedores sidecar efímeros |
| **Perfiles Seccomp / AppArmor** | Restringe llamadas al sistema y acceso a archivos | Perfiles muy estrictos causan bloqueos de ejecución | Empezar con el perfil por defecto y auditar llamadas antes de restringir |
| **Límites de recursos (RAM, CPU, PIDs)** | Previene denegación de servicio (DoS) y bombas fork | Si son muy ajustados, la app se detiene bajo picos de carga | Medir consumo base y añadir margen de seguridad |
| **Aislamiento de comunicación inter-contenedor (`--icc=false`)** | Frena el movimiento lateral de atacantes entre contenedores | Añade complejidad al requerir definición explícita de enlaces | Utilizar redes definidas por software (SDNs) dedicadas |

*Tabla 13.2: Compromisos y trade-offs en el endurecimiento de contenedores*

---

### Gestión segura de secretos (*Secrets Management*)

> [!CAUTION]
> **Nunca incluyas secretos en imágenes ni en variables `ENV`**:  
> Las imágenes son inmutables y públicas; las variables de entorno quedan expuestas en `docker inspect`, volcados de memoria y logs de depuración.

#### 1. Secretos en Docker Swarm (cifrados en reposo y en tránsito):
```bash
$ docker swarm init
$ echo "SuperSecretDbPwd" > db_password.txt
$ docker secret create db_password db_password.txt
$ docker service create \
  --name myapp \
  --secret db_password \
  -e DB_PASSWORD_FILE=/run/secrets/db_password \
  myapp:latest
```

#### 2. Secretos en Docker Compose:
1. Crear `docker-compose.yml`:
   ```yaml
   services:
     web:
       build: .
       ports:
         - "4567:4567"
       environment:
         DB_PASSWORD_FILE: /run/secrets/db_password
       secrets:
         - db_password

   secrets:
     db_password:
       file: ./db_password.txt
   ```
   *Figura 13.9: docker-compose.yml con inyección de secreto*

2. Leer el secreto desde el archivo en `app.rb`:
   ```ruby
   require "sinatra"

   set :bind, "0.0.0.0"
   set :port, 4567

   secret_path = ENV["DB_PASSWORD_FILE"]
   db_password = secret_path && File.exist?(secret_path) ? File.read(secret_path).strip : "NOT_SET"

   puts "Database Password Loaded: #{db_password}"

   get "/" do
     "Hello, World! (from Ruby and Sinatra)"
   end
   ```
   *Figura 13.10: Lectura del secreto montado en /run/secrets/*

3. Ejecutar y verificar en logs:
   ```bash
   $ docker compose up --build
   $ docker compose logs web
   ```
   *Figura 13.11: Comprobación de carga del secreto*

#### 3. Secretos en tiempo de compilación (`RUN --mount=type=secret`):
Permite descargar paquetes de repositorios privados durante el `docker build` sin almacenar credenciales en las capas resultantes de la imagen.

---

### Herramientas de seguridad en tiempo de ejecución (*Runtime Security con Falco*)

El escaneo estático solo detecta vulnerabilidades conocidas. La seguridad en tiempo de ejecución actúa como un sistema de detección de intrusiones analizando llamadas al sistema (*syscalls*) mediante sondas **eBPF**.

#### 1. Despliegue de Falco en Docker:
```bash
$ docker pull falcosecurity/falco:latest
$ docker run --rm -it \
  --privileged \
  -v /var/run/docker.sock:/host/var/run/docker.sock \
  -v /proc:/host/proc:ro \
  -v /etc:/host/etc:ro \
  falcosecurity/falco:latest
```

#### 2. Simulación de actividad maliciosa:
En otra terminal, ejecuta:
```bash
$ docker run --rm alpine /bin/sh -c "touch /etc/passwd"
```
Falco detectará la escritura en `/etc/passwd` y emitirá una alerta inmediata con los metadatos del contenedor.

#### 3. Regla personalizada de detección en Falco:
```yaml
- rule: Terminal shell in container
  desc: A shell was spawned by a container with an attached terminal
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
  output: >
    A shell was used in a container
    (user=%user.name user_loginuid=%user.loginuid
    container_id=%container.id container_name=%container.name
    image=%container.image.repository:%container.image.tag
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline
    terminal=%proc.tty)
  priority: WARNING
  tags: [container, shell, mitre_execution]
```
*Figura 13.12: Regla de Falco para detectar shells interactivos dentro de contenedores*

---

### Resumen

En este capítulo aprendimos:
- A asegurar la cadena de suministro generando inventarios SBOM con **Syft**.
- A realizar escaneo continuo de vulnerabilidades con **Trivy** e integrarlo como puerta de enlace en CI/CD.
- A garantizar la inmutabilidad y procedencia de imágenes firmándolas digitalmente con **Cosign**.
- A aplicar técnicas de endurecimiento: usuarios `non-root`, sistemas de archivos de solo lectura, reducción de capacidades del kernel y límites cgroups.
- A inyectar secretos en tiempo de ejecución mediante archivos montados evitando filtraciones en variables de entorno.
- A monitorizar el comportamiento en tiempo de ejecución y detectar anomalías con **Falco**.

---

### Referencias

- **Syft (Generación de SBOM)**: [https://github.com/anchore/syft](https://github.com/anchore/syft)
- **Trivy (Escaneo de vulnerabilidades y secretos)**: [https://trivy.dev](https://trivy.dev/)
- **Cosign y Sigstore**: [https://docs.sigstore.dev/cosign/](https://docs.sigstore.dev/cosign/)
- **Falco (Seguridad en tiempo de ejecución)**: [https://falco.org/](https://falco.org/)
- **Secretos en Docker Build**: [https://docs.docker.com/build/building/secrets/](https://docs.docker.com/build/building/secrets/)

---

### Preguntas

1. **¿Qué es un SBOM y por qué es importante en la seguridad de la cadena de suministro?**
2. **Describe cómo funcionan herramientas como Trivy y cómo integrarías una en tu pipeline de CI.**
3. **¿Qué es la confianza de contenido (*Content Trust*) y cómo ayuda Cosign a lograrla?**
4. **Menciona tres prácticas de endurecimiento de contenedores y explica los compromisos (*trade-offs*) asociados.**
5. **¿Por qué se deben evitar los secretos incrustados en imágenes o variables de entorno, y qué mecanismos existen para manejarlos de forma segura?**
6. **¿Qué rol cumplen las herramientas de seguridad en tiempo de ejecución como Falco y por qué no basta con escaneo estático y hardening?**
7. **Si tu escáner en CI rechaza una imagen por un CVE de severidad HIGH pero necesitas lanzar una versión urgente, ¿qué mitigaciones temporales aplicarías?**

---

### Respuestas

1. **Definición e importancia de SBOM**:  
   Un Software Bill of Materials (SBOM) es un inventario exhaustivo de todos los paquetes, componentes y dependencias presentes en una imagen. Proporciona visibilidad absoluta para auditorías de cumplimiento, gestión de parches y detección inmediata cuando se divulga una nueva vulnerabilidad de día cero en un componente upstream.

2. **Funcionamiento e integración de Trivy**:  
   Trivy compara los binarios y dependencias del contenedor con bases de datos de vulnerabilidades (CVEs). Se integra en CI ejecutando `trivy image --severity CRITICAL,HIGH --exit-code 1 <image>`, fallando el pipeline automáticamente si se superan los umbrales de riesgo permitidos.

3. **Content Trust con Cosign**:  
   Garantiza que la imagen descargada es idéntica a la generada por la organización y no ha sido manipulada ni suplantada. Cosign firma la imagen criptográficamente y almacena las firmas en registros OCI o registros de transparencia, permitiendo a los controladores de admisión rechazar cualquier imagen sin firma válida.

4. **Tres prácticas de hardening y sus trade-offs**:  
   - **Ejecutar como `non-root`**: Reduce el impacto de un escape, pero rompe aplicaciones que requieren enlazar puertos privilegiados (`<1024`) o alterar permisos de archivos.  
   - **Sistema de archivos de solo lectura (`--read-only`)**: Impide la inyección de código malicioso persistente, pero exige montar volúmenes específicos para carpetas que requieran escritura legítima (`/tmp`, `/logs`).  
   - **Imágenes mínimas / Distroless**: Reduce radicalmente la superficie de ataque, pero dificulta la depuración al carecer de shells y herramientas de diagnóstico.

5. **Riesgos de secretos en imágenes/ENV y alternativas seguras**:  
   Los secretos en imágenes quedan almacenados permanentemente en las capas intermedias y se exponen en registros públicos; las variables `ENV` son visibles en `docker inspect` y volcados de procesos. Deben inyectarse mediante Docker Secrets, secretos de Compose montados en archivos en memoria (`/run/secrets/`), o sistemas externos como HashiCorp Vault y AWS Secrets Manager. En compilaciones se usa `RUN --mount=type=secret`.

6. **Rol de Falco y seguridad en tiempo de ejecución**:  
   Falco monitoriza llamadas al sistema del kernel de Linux mediante eBPF en tiempo real, detectando actividades anómalas que el escaneo estático no puede prever (vulnerabilidades de día cero, ejecuciones de shells dentro de contenedores, intentos de escalada de privilegios o modificaciones en directorios del sistema).

7. **Mitigación temporal ante fallos en CI**:  
   Aplicar una excepción documentada y temporal en la lista de exclusión (*allow-list*) con fecha de caducidad estricta tras evaluar el riesgo real de explotación; aislar la red del contenedor para mitigar vectores de entrada; y priorizar de inmediato la actualización y reconstrucción del paquete afectado para eliminar la excepción lo antes posible.

