# Parte 3: Fundamentos de la Orquestación
## Capítulo 10: Uso de redes en un solo host

En el capítulo anterior, aprendimos acerca de los patrones arquitectónicos y las mejores prácticas esenciales al trabajar con arquitecturas de aplicaciones distribuidas.

En este capítulo, presentaremos el **Modelo de Red de Contenedores (*Container Network Model* o CNM)** de Docker y su implementación en un solo host mediante la red **bridge**. También exploraremos el concepto de **redes definidas por software (*Software-Defined Networks* o SDNs)** y cómo utilizarlas para proteger y aislar aplicaciones contenerizadas. Además, demostraremos cómo publicar puertos de contenedores para hacer accesibles los servicios al exterior y cómo utilizar **Traefik** como proxy inverso para habilitar enrutamiento avanzado a nivel HTTP (capa 7).

---

### Temas tratados en este capítulo:
- Análisis del Modelo de Red de Contenedores (CNM)
- Cortafuegos y aislamiento de red (*Network firewalling*)
- Trabajo con redes de tipo bridge
- Los tipos de red `host` y `none` (null)
- Ejecución de contenedores en un espacio de nombres de red existente
- Gestión y publicación de puertos de contenedores
- Enrutamiento a nivel HTTP mediante un proxy inverso (Traefik)

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Crear, inspeccionar y eliminar redes bridge personalizadas.
- Ejecutar contenedores conectados a redes bridge de usuario.
- Aislar contenedores entre sí ubicándolos en redes bridge independientes.
- Publicar puertos de contenedores en puertos específicos del host.
- Configurar Traefik como proxy inverso dinámico para enrutar tráfico HTTP hacia microservicios.

---

### Requisitos técnicos

Para seguir los ejercicios prácticos de este capítulo, solo necesitas un host con Docker Desktop (macOS, Windows o Linux).

Prepara la carpeta de trabajo:

```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir chapter-10 && cd chapter-10
```

---

### Análisis del Modelo de Red de Contenedores (CNM)

Las aplicaciones empresariales modernas están formadas por múltiples contenedores que necesitan comunicarse e intercambiar datos de forma segura. En Docker, estas vías de comunicación se denominan **redes**.

Docker formalizó esta arquitectura a través de la especificación **Container Network Model (CNM)**:

*Figura 10.1 – El Container Network Model (CNM) de Docker*

#### Elementos fundamentales del CNM:
1. **Network Sandbox**: Aísla por completo la pila de red de un contenedor (interfaces de red, tablas de enrutamiento y resolución DNS). Por defecto, bloquea el tráfico entrante no autorizado.
2. **Endpoint**: Punto de conexión que vincula un Network Sandbox a una red concreta (equivalente a una tarjeta de red virtual conectada a un conmutador). Un Sandbox puede tener cero, uno o múltiples endpoints simultáneamente.
3. **Network**: Vía de comunicación que transporta paquetes de datos entre endpoints interconectados, ya sea en un solo host (local) o a través de un clúster (global).

#### Controladores de red (*Network Drivers*)

| Red | Proveedor | Alcance | Descripción |
| :--- | :--- | :--- | :--- |
| **`bridge`** | Docker | Local | Red basada en puentes virtuales Linux para comunicación en un único host. Es el tipo de red por defecto. |
| **`macvlan`** | Docker | Local | Asigna direcciones MAC directas a cada contenedor, haciéndolos aparecer como dispositivos físicos en la red LAN. |
| **`ipvlan`** | Docker | Local | Similar a Macvlan pero comparte una única dirección MAC y enruta por direcciones IP. |
| **`overlay`** | Docker | Global | Red multi-nodo basada en VXLAN que permite la comunicación entre contenedores en diferentes hosts de un clúster/Swarm. |
| **`weave`** | Weaveworks | Global | Red multi-host resiliente con descubrimiento automático y cifrado integrado. |
| **`contiv`** | Cisco | Global | Red multi-inquilino basada en políticas integrada con infraestructura física. |
| **`calico`** | Tigera | Global | Red enrutada de alto rendimiento en Capa 3 sin sobrecarga de túneles con políticas de seguridad avanzadas. |

*Tabla 10.1 – Tipos de controladores de red para Docker*

---

### Cortafuegos y aislamiento de red (*Network Firewalling*)

Docker sigue el principio de **seguridad por defecto**:
- Los contenedores conectados a la **misma red definida por software (SDN)** pueden comunicarse libremente entre sí.
- Los contenedores conectados a **redes distintas están completamente aislados** y no pueden comunicarse, actuando como un cortafuegos nativo (*Figura 10.2*).

#### Patrón de aislamiento de tres capas:
Si tenemos un servicio web público (`webAPI`), un catálogo interno (`productCatalog`) y una base de datos confidencial (`database`):
- `webAPI` se conecta únicamente a la red `front-net`.
- `database` se conecta únicamente a la red `back-net`.
- `productCatalog` se conecta a ambas redes (`front-net` y `back-net`), actuando como intermediario controlado (*Figura 10.3*). Si `webAPI` es vulnerada, el atacante no puede acceder directamente a `database`.

#### Soporte Dual-Stack (IPv4 e IPv6):
```bash
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/16 \
  --ipv6 --subnet fd00:dead:beef::/48 \
  mynet-v6
```

#### Reglas de cortafuegos en Linux moderno (`nftables` vs `iptables`):
En distribuciones recientes (Ubuntu 24.04+, Fedora 38+), Docker gestiona las reglas mediante `nftables`:
```bash
$ sudo nft list ruleset
```
*(En distribuciones anteriores se consultaba con `sudo iptables -L`).*

---

### Trabajo con redes de tipo bridge

Al iniciar el daemon de Docker, se crea automáticamente un puente virtual Linux llamado `docker0` asociado a la red por defecto `bridge`.

1. Listar las redes disponibles:
   ```bash
   $ docker network ls
   ```
   *Figura 10.4 – Listado de redes por defecto de Docker*

2. Inspeccionar la red `bridge` por defecto:
   ```bash
   $ docker network inspect bridge
   ```
   *Figura 10.5 – Inspección de la red bridge por defecto*

   El bloque IPAM asigna por defecto la subred `172.17.0.0/16`. La dirección `172.17.0.1` queda reservada para la puerta de enlace (*gateway*) del puente Linux.

> **Evitar conflictos de subred**: Si la subred `172.17.0.0/16` colisiona con tu red corporativa o VPN, modifica el archivo `/etc/docker/daemon.json`:
> ```json
> { "bip": "192.168.100.1/24" }
> ```

*Figura 10.6 – Topología de la red bridge*  
*Figura 10.7 – Detalles de conexión veth entre el host y el contenedor*

---

### Creación de redes bridge personalizadas

Las redes bridge creadas por el usuario proporcionan **resolución DNS automática por nombre de contenedor**, a diferencia de la red bridge por defecto.

1. Crear una red bridge personalizada llamada `sample-net`:
   ```bash
   $ docker network create \
     --driver bridge \
     sample-net
   ```

2. Comprobar la subred asignada:
   ```bash
   $ docker network inspect sample-net | grep Subnet
   ```
   Salida:
   ```text
   "Subnet": "172.18.0.0/16",
   ```

3. Crear una red con subred personalizada:
   ```bash
   $ docker network create \
     --driver bridge \
     --subnet "10.1.0.0/16" \
     test-net
   ```

4. Ajustar el tamaño de MTU (útil en VPNs):
   ```bash
   $ docker network create \
     --driver bridge \
     --opt com.docker.network.driver.mtu=1400 \
     mynet-mtu
   ```

---

### Conexión de contenedores a redes bridge

1. Ejecutar un contenedor en la red bridge por defecto:
   ```bash
   $ docker container run \
     --name c1 -it --rm \
     alpine:3.22 /bin/sh
   ```

2. En otra terminal, inspeccionar la configuración de red del contenedor:
   ```bash
   $ docker container inspect c1 | jq '.[0].NetworkSettings'
   ```
   *Figura 10.9 – Sección NetworkSettings de los metadatos del contenedor*

3. Dentro del contenedor `c1`, comprobar la interfaz `eth0` y rutas:
   ```bash
   / # ip addr show eth0
   / # ip route
   ```
   Salida de rutas:
   ```text
   default via 172.17.0.1 dev eth0
   172.17.0.0/16 dev eth0 scope link src 172.17.0.2
   ```
   *Figura 10.10 y Figura 10.11 – Interfaz eth0 vista desde dentro del contenedor*

4. Ejecutar un segundo contenedor `c2` en segundo plano:
   ```bash
   $ docker container run \
     --name c2 -d --rm \
     alpine:3.22 ping 127.0.0.1
   ```

5. Ejecutar dos contenedores (`c3` y `c4`) en la red personalizada `sample-net`:
   ```bash
   $ docker container run \
     --name c3 --rm -d \
     --network sample-net \
     alpine:3.22 ping 127.0.0.1
   $ docker container run \
     --name c4 --rm -d \
     --network sample-net \
     alpine:3.22 ping 127.0.0.1
   ```

6. Probar la resolución DNS interna dentro de `sample-net`:
   ```bash
   $ docker container exec -it c3 /bin/sh
   / # ping c4
   ```
   La comunicación funciona de forma inmediata resolviendo el nombre `c4`.

7. Probar el aislamiento entre redes:
   ```bash
   / # ping c2
   ping: bad address 'c2'
   / # ping 172.17.0.3
   100% packet loss
   ```
   *(El tráfico hacia otras redes queda bloqueado por el firewall).*

8. Conectar un contenedor a múltiples redes:
   ```bash
   $ docker network create sample-net-2
   $ docker container run --name c5 --rm -d --network sample-net alpine:3.22 ping 127.0.0.1
   $ docker container run --name c6 --rm -d --network sample-net-2 alpine:3.22 ping 127.0.0.1
   $ docker network connect sample-net c6
   ```
   *Figura 10.14: Contenedor c6 conectado a dos redes simultáneamente*

9. Limpieza de contenedores y redes:
   ```bash
   $ docker container rm -f $(docker container ls -aq)
   $ docker network prune --force
   ```
   *Figura 10.15: Poda de todas las redes Docker no utilizadas*

---

### Los tipos de red `host` y `none`

#### La red `host`:
Elimina el aislamiento de red; el contenedor comparte directamente la pila de red, interfaces e IPs del host:
```bash
$ docker container run \
  --rm -it \
  --network host \
  alpine:3.22 /bin/sh
```
*Figura 10.16 y Figura 10.17 – Dispositivo eth0 y rutas del host vistas desde el contenedor*

- **Casos de uso**: Aplicaciones que requieren latencia ultra-baja sin sobrecarga de NAT o escaneo/diagnóstico de interfaces de red del host.
- **Riesgo de seguridad**: Rompe el aislamiento; cualquier brecha en el contenedor expone directamente toda la red del host.

#### La red `none` (null):
Crea un Network Sandbox sin ninguna interfaz de red externa (solo interfaz loopback `lo`):
```bash
$ docker container run \
  --rm -it \
  --network none \
  alpine:3.22 /bin/sh
```
Comprobación:
```bash
/ # ip addr show eth0
ip: can't find device 'eth0'
```
- **Casos de uso**: Procesamiento de datos ultraseguro fuera de línea (*offline*), tareas por lotes de computación aisladas o análisis forense.

---

### Ejecución en un espacio de nombres de red existente (`container:`)

Docker permite que un contenedor comparta el **mismo Network Sandbox** que otro contenedor en ejecución:

*Figura 10.18 – Múltiples contenedores compartiendo un único espacio de nombres de red*

1. Iniciar un servidor web Nginx en una red personalizada:
   ```bash
   $ docker network create --driver bridge test-net
   $ docker container run --name web -d --network test-net nginx:1.29-alpine
   ```

2. Adjuntar un contenedor Alpine al espacio de nombres del contenedor `web`:
   ```bash
   $ docker container run -it --rm --network container:web alpine:3.22 /bin/sh
   ```

3. Desde el nuevo contenedor, acceder a Nginx a través de `localhost`:
   ```bash
   / # wget -qO- http://localhost:80
   ```
   *Figura 10.19: Acceso a Nginx desde otro contenedor vía localhost*

4. Limpieza:
   ```bash
   $ docker container rm --force web
   $ docker network rm test-net
   ```

> **Fundamento de los Pods de Kubernetes**: Este mecanismo es la base técnica de los Pods en Kubernetes, donde múltiples contenedores (e.g. contenedor principal y sidecars) comparten IP, puertos y el espacio de direcciones `localhost`.

#### Depuración con `netshoot`:
```bash
$ docker container run --rm -it \
  --network container:myapp \
  nicolaka/netshoot
```

---

### Gestión y publicación de puertos de contenedores

Por defecto, los puertos expuestos dentro de un contenedor no son accesibles desde el exterior. Para permitir el acceso externo, debemos **publicar (*publish*)** el puerto del contenedor hacia el host.

*Figura 10.20 – Mapeo de puertos de contenedores a puertos del host*

#### Publicación automática de puertos con `-P`:
Docker asigna automáticamente un puerto efímero libre del host (rango 32768–65535):
```bash
$ docker container run --name web -P -d nginx:1.29-alpine
$ docker container port web
```
Salida:
```text
80/tcp -> 0.0.0.0:55000
```
*Figura 10.21 y Figura 10.22 – Página de bienvenida de Nginx y mapeo en docker container ls*

#### Publicación explícita de puertos con `-p` (`<puerto-host>:<puerto-contenedor>`):
```bash
$ docker container run --name web2 -p 8080:80 -d nginx:1.29-alpine
```

- **Restringir al loopback (desarrollo seguro local)**:
  ```bash
  $ docker container run -d -p 127.0.0.1:8080:80 my/dev-app
  ```
- **Rango de puertos**:
  ```bash
  $ docker container run -d --name worker-a -p 12000-12010:12000-12010 my/worker
  ```
- **Protocolo UDP**:
  ```bash
  $ docker container run -d --name svc -p 9000:9000/udp my/proto-svc
  ```

---

### Enrutamiento a nivel HTTP mediante un proxy inverso (Traefik)

Para descomponer un monolito en microservicios sin modificar las URLs públicas de los clientes, utilizamos un proxy inverso en el borde (*edge router*).

#### 1. Contenerización del monolito (`city-tours`):
1. Copiar y preparar el proyecto:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-10
   $ cp -r solutions/city-tours .
   $ cd city-tours
   $ uv venv
   $ source .venv/bin/activate
   $ uv pip install -r requirements.txt
   $ export FLASK_APP=app.py
   $ flask run
   ```
   *Figura 10.23, Figura 10.24 y Figura 10.25 – Ejecución e inspección del endpoint /health*

2. Mapear `127.0.0.1 acme.com` en `/etc/hosts` (o usar `curl --resolve`).
3. Construir y ejecutar la imagen del monolito:
   ```bash
   $ docker image build -t acme/city-tours:1.0 .
   $ docker container run --rm -it --name city-tours -p 5000:5000 acme/city-tours:1.0
   ```
   *Figura 10.27 y Figura 10.28 – Pruebas del monolito en contenedor*

#### 2. Extracción del microservicio de catálogo (`catalog` en Node.js):
```bash
$ cp -r solutions/catalog .
$ docker image build -t acme/catalog:1.0 .
$ docker container run --rm -it --name catalog -p 3000:3000 acme/catalog:1.0
$ curl http://acme.com:3000/catalog/tours?city=paris
$ docker container rm -f city-tours catalog
```

#### 3. Configuración de Traefik y enrutamiento dinámico mediante etiquetas (*labels*):

1. Iniciar el microservicio `catalog` con alta prioridad para rutas `/catalog`:
   ```bash
   $ docker container run --rm -d \
     --name catalog \
     --label traefik.enable=true \
     --label traefik.port=3000 \
     --label traefik.priority=10 \
     --label traefik.http.routers.catalog.rule="Host(\"acme.com\") && PathPrefix(\"/catalog\")" \
     acme/catalog:1.0
   ```

2. Iniciar el monolito `city-tours` con menor prioridad como ruta por defecto:
   ```bash
   $ docker container run --rm -d \
     --name city-tours \
     --label traefik.enable=true \
     --label traefik.port=5000 \
     --label traefik.priority=1 \
     --label traefik.http.routers.city-tours.rule="Host(\"acme.com\")" \
     acme/city-tours:1.0
   ```

3. Iniciar el proxy inverso Traefik escuchando en el puerto 80 del host y montando el socket de Docker:
   ```bash
   $ docker container run -d \
     --name traefik \
     -p 8080:8080 \
     -p 80:80 \
     -v /var/run/docker.sock:/var/run/docker.sock \
     traefik:v2.0 \
     --api.insecure=true \
     --providers.docker=true \
     --entrypoints.web.address=:80 \
     --log.level=DEBUG \
     --providers.docker.exposedbydefault=false
   ```

4. Probar las peticiones unificadas en el puerto 80:
   - Petición al monolito:
     ```bash
     $ curl -sL http://acme.com/health | jq .
     ```
   - Petición enrutada transparentemente al nuevo microservicio Node.js:
     ```bash
     $ curl -sL "http://acme.com/catalog/tours?city=paris" | jq .
     ```

5. Limpieza:
   ```bash
   $ docker container rm -f traefik city-tours catalog
   ```

---

### Resumen

En este capítulo analizamos:
- Los tres pilares del Container Network Model (CNM): Sandbox, Endpoint y Network.
- El funcionamiento del aislamiento de red como cortafuegos natural mediante redes definidas por software (SDNs).
- La configuración de redes `bridge` personalizadas con resolución DNS interna automática y soporte IPv4/IPv6 dual-stack.
- Las características y riesgos de la red `host` frente a la red totalmente aislada `none`.
- El uso compartido de espacios de nombres con `--network container:` y su aplicación en Pods de Kubernetes y sidecars de depuración.
- La publicación de puertos con `-P` y `-p` y las mejores prácticas de seguridad en la exposición de interfaces.
- La integración de Traefik como proxy inverso dinámico para descomponer monolitos mediante enrutamiento en Capa 7 sin afectar a los clientes.

---

### Lecturas adicionales

- **Visión general de redes en Docker**: [http://dockr.ly/2sXGzQn](http://dockr.ly/2sXGzQn)
- **Redes de contenedores**: [http://dockr.ly/2HJfQKn](http://dockr.ly/2HJfQKn)
- **¿Qué es un Linux bridge?**: [https://bit.ly/2HyC3Od](https://bit.ly/2HyC3Od)
- **Uso de redes bridge**: [http://dockr.ly/2BNxjRr](http://dockr.ly/2BNxjRr)
- **Uso de redes Macvlan**: [http://dockr.ly/2ETjy2x](http://dockr.ly/2ETjy2x)
- **Uso de la red host**: [http://dockr.ly/2F4aI59](http://dockr.ly/2F4aI59)

---

### Preguntas

1. **Nombra los tres elementos centrales del Container Network Model (CNM).**
2. **¿Cómo creas una red bridge personalizada llamada `frontend`?**
3. **¿Cómo ejecutas dos contenedores `nginx:alpine` conectados a la red `frontend`?**
4. **Para la red `frontend`, ¿cómo obtienes las IPs de todos los contenedores conectados y su subred asociada?**
5. **¿Cuál es el propósito de la red `host`?**
6. **Menciona uno o dos escenarios donde sea apropiado utilizar la red `host`.**
7. **¿Cuál es el propósito de la red `none`?**
8. **¿En qué escenarios debe utilizarse la red `none`?**
9. **¿Por qué utilizaríamos un proxy inverso como Traefik junto con nuestra aplicación contenerizada?**

---

### Respuestas

1. **Tres elementos del CNM**:  
   - **Sandbox**: Espacio de nombres de red aislado donde reside la pila de red del contenedor.  
   - **Endpoint**: Interfaz virtual que conecta un sandbox con una red.  
   - **Network**: Conjunto de endpoints interconectados que pueden comunicarse directamente entre sí.

2. **Crear red bridge personalizada**:  
   ```bash
   $ docker network create --driver bridge \
     --subnet 172.25.0.0/16 frontend
   ```
   Para usarla en un contenedor:
   ```bash
   $ docker container run --network frontend <docker-image>
   ```

3. **Ejecutar contenedores en la red `frontend`**:  
   ```bash
   $ docker container run --name nginx1 --network frontend -d nginx:alpine
   $ docker container run --name nginx2 --network frontend -d nginx:alpine
   ```

4. **Obtener IPs y subred de la red `frontend`**:  
   - IPs de los contenedores:
     ```bash
     $ docker network inspect frontend --format='{{range .Containers}}{{.IPv4Address}} {{end}}'
     ```
   - Subred asignada:
     ```bash
     $ docker network inspect frontend --format='{{json .IPAM.Config}}' | jq -r '.[].Subnet'
     ```

5. **Propósito de la red `host`**:  
   Permite al contenedor utilizar directamente la pila de red del host sin aislamiento ni traducción de direcciones de red (NAT), mejorando el rendimiento de red a costa de perder la seguridad del aislamiento.

6. **Escenarios para la red `host`**:  
   - Aplicaciones con requisitos de latencia extremadamente baja donde la capa de NAT introduce sobrecarga inaceptable.  
   - Aplicaciones que necesitan escuchar en una gran cantidad de puertos dinámicos o realizar transmisiones *broadcast/multicast*.

7. **Propósito de la red `none`**:  
   Deshabilita completamente la conectividad de red del contenedor, dejándolo sin interfaces externas salvo la interfaz loopback local (`lo`).

8. **Escenarios para la red `none`**:  
   - Trabajos por lotes (*batch processing*) o tareas efímeras que procesan datos locales montados por volumen sin necesidad de red.  
   - Tareas de alta seguridad o análisis forense donde se requiere garantizar aislamiento total contra accesos externos o fugas de datos.

9. **Beneficios de un proxy inverso (Traefik)**:  
   - **Balanceo de carga**: Distribuye el tráfico equitativamente entre múltiples réplicas de los contenedores.  
   - **Enrutamiento por Capa 7**: Dirige peticiones según nombres de host y rutas URL hacia diferentes microservicios internos.  
   - **Terminación SSL/TLS**: Gestiona certificados y cifrado HTTPS de forma centralizada.  
   - **Seguridad**: Ofrece un punto único de entrada controlado, manteniendo los contenedores de aplicación en redes internas privadas sin publicar sus puertos directamente en el host.
