# Parte 1: Introducción
## Capítulo 1: ¿Qué son los contenedores y por qué debería usarlos?

Este primer capítulo te introducirá en el mundo de los contenedores, mostrando cómo optimizan la cadena de suministro de software moderna y abordan los desafíos de seguridad que a menudo surgen con los modelos de despliegue tradicionales. Asumiremos un conocimiento previo nulo o mínimo sobre contenedores, por lo que nuestros primeros pasos se centrarán en ilustrar los puntos de fricción en los flujos de trabajo heredados y demostrar cómo los contenedores reducen significativamente esa fricción. 

Sobre esta base, exploraremos tanto el ecosistema clásico —donde los componentes de código abierto (*upstream OSS*), conocidos colectivamente como **Moby**, sirven como bloques de construcción detrás de los productos familiares de Docker— como las últimas tendencias en contenerización que han surgido o se han consolidado desde 2022, momento en que se escribió la edición anterior del libro. 

Aprenderás no solo por qué los contenedores fueron un concepto revolucionario cuando aparecieron por primera vez, sino también cómo características como los modos de operación sin privilegios de root (*rootless*), las mejoras en la seguridad de la cadena de suministro y las nuevas técnicas de orquestación están dando forma al panorama actual de los contenedores. Al final de este capítulo, comprenderás cómo se ensamblan los contenedores y por qué son más importantes que nunca para entregar aplicaciones seguras y portables.

---

### Temas tratados en este capítulo:
- ¿Qué son los contenedores?
- ¿Por qué son importantes los contenedores?
- ¿Cuál es el beneficio de usar contenedores para mí o para mi empresa?
- El proyecto Moby
- Productos de Docker
- Arquitectura de contenedores
- Novedades en la contenerización

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Explicar qué son los contenedores a una persona interesada sin conocimientos técnicos, utilizando analogías cotidianas como los contenedores físicos de carga frente al transporte a granel.
- Justificar por qué los contenedores son tan importantes comparando su enfoque con la diferencia entre apartamentos y viviendas unifamiliares, o ejemplos simplificados similares.
- Nombrar al menos cuatro componentes de código abierto (*upstream OSS*) agrupados bajo Moby que impulsan productos de Docker como Docker Desktop.
- Dibujar un esquema de alto nivel de la arquitectura de contenedores de Docker para ilustrar cómo encajan las imágenes por capas y los *namespaces*.
- Identificar los desarrollos recientes (posteriores a 2022) en la contenerización, incluidas nuevas medidas de seguridad, modos *rootless* y depuración mejorada en Kubernetes, y explicar cómo continúan transformando los despliegues modernos.

¡Comencemos!

> **Tu compra incluye una copia gratuita en PDF + extras exclusivos**  
> Tu compra incluye una copia en PDF sin DRM de este libro, una prueba de 7 días de la biblioteca Packt+ (no requiere tarjeta de crédito) y extras exclusivos adicionales. Consulta la sección *Beneficios gratuitos con tu libro* en el Prefacio para desbloquearlos al instante y maximizar tu aprendizaje.

---

### ¿Qué son los contenedores?

Un contenedor de software es algo bastante abstracto, por lo que puede ser útil comenzar con una analogía que debería resultar familiar para la mayoría: el **contenedor de carga** en la industria del transporte.

A lo largo de la historia, las personas han transportado mercancías de un lugar a otro por diversos medios. Antes de la invención de la rueda, los bienes probablemente se transportaban en sacos, cestas o baúles sobre los hombros de los propios seres humanos, o se utilizaban animales como burros, camellos o elefantes. Con la invención de la rueda, el transporte se volvió más eficiente a medida que se construían caminos para trasladar carros, permitiendo transportar muchas más mercancías a la vez. Cuando se introdujeron las primeras máquinas de vapor y, más tarde, los motores de gasolina, el transporte se volvió aún más potente. Hoy en día transportamos enormes cantidades de bienes en aviones, trenes, barcos y camiones. Al mismo tiempo, los tipos de mercancías se volvieron cada vez más diversos y complejos de manejar.

En todos estos miles de años, una cosa no cambió: la necesidad de **descargar mercancías** en una ubicación de destino y tal vez **cargarlas en otro medio de transporte**. Pensemos, por ejemplo, en un agricultor que lleva un carro lleno de manzanas a una estación central de tren donde las manzanas se cargan en un vagón junto con las de muchos otros agricultores. O en un viticultor que transporta barriles de vino en camión hasta el puerto, donde se descargan y se transfieren a un barco para transportarlos al extranjero.

Este proceso de descarga y recarga entre distintos medios de transporte era complejo y tedioso. Cada tipo de producto se empaquetaba a su manera y requería un manejo particular. Además, las mercancías sueltas corrían el riesgo de ser robadas o sufrir daños durante la manipulación.

*Figura 1.1 – Marineros descargando mercancías de un barco*

Entonces aparecieron los **contenedores**, revolucionando por completo la industria del transporte. Un contenedor es simplemente una caja metálica con dimensiones estandarizadas: la longitud, el ancho y la altura de cada contenedor son iguales. Este es un punto fundamental: sin un acuerdo global sobre un tamaño estándar, el concepto de contenedor no habría tenido el éxito que tiene hoy.

Con contenedores estandarizados, las empresas empaquetan sus productos dentro de ellos y contratan a una empresa de transporte. Esta llega con un medio de transporte estandarizado: camiones adaptados para transportar un contenedor, vagones de tren capaces de llevar uno o varios contenedores, o buques portacontenedores especializados en trasladar miles de ellos.

Los transportistas ya no necesitan desempaquetar y reempaquetar las mercancías. Para ellos, un contenedor es una **caja negra**: no necesitan saber qué hay dentro en la mayoría de los casos. Empaquetar los bienes queda en manos de quienes envían la carga, quienes conocen la mejor forma de manipularla. Como todos los contenedores tienen la misma forma y dimensiones acordadas, se pueden utilizar herramientas estandarizadas como grúas que transfieren contenedores entre trenes, camiones y barcos. Un solo tipo de grúa basta para gestionar cualquier contenedor. Gracias a esta estandarización, todos los procesos logísticos se hicieron mucho más eficientes.

*Figura 1.2 – Barco portacontenedores siendo cargado en un puerto*

Esta analogía es perfecta porque los **contenedores de software** cumplen exactamente la misma función en la denominada **cadena de suministro de software** (*software supply chain*).

Veamos lo que esto significa en la industria de TI y el desarrollo de software. Antiguamente, los desarrolladores creaban aplicaciones y, una vez terminadas, las entregaban a los ingenieros de operaciones para que las instalaran y ejecutaran en los servidores de producción. Con suerte, los desarrolladores proporcionaban un documento con instrucciones de instalación más o menos precisas.

El problema surgía cuando múltiples equipos de desarrollo creaban aplicaciones muy diferentes que debían convivir en los mismos servidores de producción. Cada aplicación tenía dependencias externas: frameworks, librerías, etc. En ocasiones, dos aplicaciones requerían versiones distintas e incompatibles del mismo framework. La vida de los ingenieros de operaciones se complicaba enormemente: instalar una nueva versión de una aplicación se convertía en un proyecto complejo que requería meses de planificación y pruebas para no romper nada. Existía una gran fricción en la cadena de suministro de software.

Hoy en día, las empresas necesitan ciclos de lanzamiento cada vez más cortos; no pueden permitirse actualizar aplicaciones solo una o dos veces al año, sino que deben hacerlo en cuestión de semanas, días o incluso varias veces al día.

Una de las primeras soluciones fue el uso de **máquinas virtuales (VMs)**: ejecutar una única aplicación por máquina virtual. Esto eliminó los problemas de compatibilidad, pero introdujo una gran sobrecarga: cada VM incluye un sistema operativo completo (Linux o Windows Server) solo para ejecutar una única aplicación. Es el equivalente a utilizar un barco entero para transportar un solo camión de plátanos: una solución ineficiente y costosa.

La solución definitiva consistió en proporcionar un mecanismo mucho más ligero que las máquinas virtuales, capaz de encapsular a la perfección la aplicación junto con todas sus dependencias externas (frameworks, librerías, configuraciones, etc.): el **contenedor Docker**.

Los desarrolladores empaquetan sus aplicaciones y dependencias en contenedores Docker y los envían a los equipos de pruebas u operaciones. Para ellos, el contenedor es una **caja negra estandarizada**. Todos los contenedores se gestionan de forma idéntica: si un contenedor funciona en un servidor, cualquier otro debería funcionar también. De ahí surgió el lema de Docker: **"Build, ship, and run anywhere"** (Construir, distribuir y ejecutar en cualquier lugar).

---

### ¿Por qué son importantes los contenedores?

El tiempo entre lanzamientos de software se reduce constantemente, mientras que la complejidad de las aplicaciones aumenta. Necesitamos simplificar la cadena de suministro de software. Además, los ciberataques y las brechas de seguridad continúan en aumento, comprometiendo datos altamente sensibles de clientes y secretos corporativos.

Los contenedores aportan beneficios clave en múltiples áreas:

1. **Seguridad mejorada**: Un informe de Gartner destacó que las aplicaciones que se ejecutan en contenedores son más seguras que sus equivalentes sin contenerizar. Los contenedores utilizan primitivas del núcleo de Linux:
   - **Namespaces del kernel de Linux**: Aíslan (*sandbox*) los diferentes procesos y aplicaciones que se ejecutan en el mismo equipo.
   - **Control Groups (cgroups)**: Evitan el problema del "vecino ruidoso" (*noisy-neighbor*), impidiendo que una aplicación consuma todos los recursos del servidor y afecte a las demás.
   - **Imágenes inmutables**: Permiten el escaneo automatizado en busca de vulnerabilidades y exposiciones comunes (**CVEs**).
   - **Confianza en el contenido (*Content Trust*)**: Garantiza criptográficamente la identidad del autor de una imagen y asegura que no haya sido alterada durante la transferencia (previniendo ataques *Man-In-The-Middle* o MITM).

2. **Simulación de entornos de producción en local**: Facilitan la recreación de entornos idénticos a producción en la máquina de desarrollo. Permiten levantar bases de datos relacionales complejas (como Oracle, PostgreSQL o SQL Server) en segundos sin instalaciones invasivas en el sistema host, y eliminarlas sin dejar rastro al finalizar las pruebas.

3. **Estandarización de la infraestructura para Operaciones**: Los operadores pueden centrarse en aprovisionar infraestructura estandarizada donde cada servidor es simplemente un *Docker host*. No se requiere instalar librerías específicas de cada aplicación en el sistema anfitrión, solo el sistema operativo y un entorno de ejecución de contenedores (*container runtime*).

---

### ¿Cuál es el beneficio de usar contenedores para mí o para mi empresa?

Como se suele decir: *"...hoy en día, cualquier empresa de cierto tamaño debe reconocer que necesita ser una empresa de software..."*. Un banco moderno es una empresa de software especializada en finanzas. El software impulsa los negocios.

Para mantenerse competitiva, una empresa debe contar con una cadena de suministro de software **segura, automatizada y estandarizada**. Grandes organizaciones han reportado que, al contenerizar aplicaciones heredadas (*legacy*) y establecer cadenas de suministro automatizadas:
- Reducen los costos de mantenimiento de aplicaciones críticas entre un **50% y un 60%**.
- Reducen el tiempo entre lanzamientos de aplicaciones tradicionales hasta en un **90%**.

La adopción de tecnologías de contenedores genera un ahorro económico sustancial, acelera los tiempos de desarrollo y reduce el *time-to-market*.

---

### El proyecto Moby

Originalmente, Docker Engine era una pieza de software monolítica y de código abierto que incluía el runtime de contenedores, librerías de red, API REST, CLI y más. Diversos proyectos y proveedores (como Red Hat o Kubernetes) utilizaban solo partes específicas de Docker Engine o aplicaban parches propios manteniendo el nombre de Docker.

Para separar claramente los componentes abiertos de los productos comerciales y proteger la marca, nació el **proyecto Moby**.

- Moby actúa como un proyecto paraguas para los componentes de código abierto que Docker desarrolla y mantiene (gestión de imágenes, secretos, configuración, red y aprovisionamiento).
- Proporciona herramientas para ensamblar componentes en artefactos ejecutables.
- Varios componentes clave fueron donados por Docker a la **Cloud Native Computing Foundation (CNCF)**, destacando:
  - **containerd** y **runc**: Forman el núcleo del runtime de contenedores.
  - **notary**: Utilizado para la firma y confianza en el contenido (*content trust*).

En palabras de Docker: *"...Moby es un framework abierto creado por Docker para ensamblar sistemas de contenedores especializados sin reinventar la rueda. Proporciona un 'juego de Lego' de docenas de componentes estándar y un framework para ensamblarlos en plataformas personalizadas..."*.

---

### Productos de Docker

Hasta 2019, Docker dividía sus productos en dos líneas: **Community Edition (CE)** (gratuita y de código abierto) y **Enterprise Edition (EE)** (de código cerrado con soporte 24/7 y licencias anuales). En 2019, Docker vendió la división Enterprise a Mirantis para reenfocarse por completo en las herramientas orientadas a desarrolladores.

#### Docker Desktop
Aplicación de escritorio fácil de instalar disponible para **macOS, Windows y Linux**. Permite construir, depurar y probar aplicaciones contenerizadas localmente. Se integra profundamente con los hipervisores, subsistemas de red y sistemas de archivos de cada sistema operativo (como WSL 2 en Windows).
*(Nota: Docker Toolbox está obsoleto/deprecated y ya no se encuentra en desarrollo activo).*

#### Docker Hub
El servicio más popular para buscar, alojar y compartir imágenes de contenedores. Ofrece cuentas individuales y de organizaciones, con soporte para repositorios públicos gratuitos y repositorios privados bajo suscripción comercial.

#### Docker EE
Adquirida por Mirantis en noviembre de 2019. Comprendía Universal Control Plane (UCP), Docker Trusted Registry (DTR) y el motor empresarial. Mirantis continúa desarrollando estas tecnologías integrándolas en soluciones enfocadas en Kubernetes, mientras Docker se enfoca en herramientas de productividad para desarrolladores.

#### Docker Swarm
Característica nativa de orquestación integrada directamente en Docker Engine (no es un producto independiente). Permite desplegar y gestionar aplicaciones distribuidas a escala utilizando la misma CLI de Docker, con balanceo de carga integrado, descubrimiento de servicios, actualizaciones progresivas (*rolling updates*) y redes seguras multihost.

---

### Arquitectura de contenedores

A continuación se muestra cómo se estructura a nivel arquitectónico un sistema capaz de ejecutar contenedores Docker (*Docker host*):

*Figura 1.3 – Diagrama de arquitectura de alto nivel de Docker Engine*

La arquitectura se compone de tres capas esenciales:
1. **Sistema operativo Linux** (en la base).
2. **Runtime del contenedor** (*container runtime*, en el medio).
3. **Docker Engine** (en la parte superior).

```
+-------------------------------------------------------+
|                    Docker CLI                         |
+-------------------------------------------------------+
                           | (REST API)
+-------------------------------------------------------+
|                   Docker Engine                       |
+-------------------------------------------------------+
|   containerd (Gestión de imágenes, redes, plugins)    |
+-------------------------------------------------------+
|   runc (Creación y gestión de bajo nivel)             |
+-------------------------------------------------------+
|   Núcleo Linux (Namespaces, cgroups, Layer FS)        |
+-------------------------------------------------------+
```

- **Primitivas del kernel de Linux**:
  - **Namespaces** (como PID, NET, IPC, MNT, UTS, USER): Aíslan y encapsulan los procesos dentro del contenedor.
  - **cgroups**: Limitan y controlan el uso de recursos (CPU, memoria RAM, I/O) para evitar el acaparamiento por parte de un único contenedor.
- **Runtime de contenedores**:
  - **runc**: Componente de bajo nivel para la creación e instanciación de contenedores.
  - **containerd**: Capa de nivel superior sobre runc que gestiona el ciclo de vida completo: descarga imágenes (*pull*), inicializa, ejecuta, detiene y elimina contenedores.
- **Docker Engine**: Agrega funcionalidades de red, plugins y expone una **API REST**, la cual es consumida por la **CLI de Docker**.

---

### Novedades en la contenerización

Desde 2022, el ecosistema de contenedores ha experimentado una notable consolidación y madurez en herramientas, seguridad y entornos de ejecución.

#### Seguridad mejorada en la cadena de suministro
- **Firma y verificación de imágenes**: Herramientas como **Notary v2** y **Cosign** (Sigstore) permiten firmar criptográficamente las imágenes en los pipelines de CI/CD para certificar su autenticidad e integridad.
- **Generación de SBOM (*Software Bill of Materials*)**: Herramientas como **Syft**, **Anchore** y **CycloneDX** generan en tiempo de compilación una lista exhaustiva de dependencias y versiones incluidas en la imagen, facilitando la detección inmediata de vulnerabilidades críticas (como *log4j*).

#### Depuración y operaciones en Kubernetes
- **Contenedores efímeros (*Ephemeral Containers*)**: Permiten acoplar temporalmente un contenedor con herramientas de diagnóstico (como `curl`, `netcat`, analizadores de red) a un Pod en ejecución en producción sin detenerlo ni reconstruirlo.

#### Extensiones de Docker Desktop
- **Extensiones de escaneo de seguridad**: Integración directa en la interfaz de Docker Desktop con herramientas como **Snyk** o **Trivy** para detectar vulnerabilidades localmente antes de subir el código.
- **Gestión multiservicio**: Visualización, configuración y orquestación de servicios, volúmenes y redes desde una única interfaz gráfica.

#### Evolución de la gestión de recursos
- **Adopción de cgroups v2**: Soporte completo y estable en las principales distribuciones de Linux, proporcionando mayor precisión en el control y métricas de CPU, memoria y E/S.
- **Modos sin privilegios de root (*Rootless*)**: Ejecución de Docker sin requerir privilegios de superusuario en el host, mitigando riesgos de seguridad y facilitando su adopción en entornos corporativos con estrictas políticas de cumplimiento.

---

### Resumen

En este capítulo vimos cómo los contenedores reducen drásticamente la fricción en la cadena de suministro de software y refuerzan la seguridad integral, apoyándose en los componentes de código abierto del proyecto Moby en el núcleo de Docker. También exploramos las tendencias clave consolidadas desde 2022, como la firma de imágenes, la generación de SBOM y el funcionamiento *rootless*.

En el próximo capítulo, comenzaremos con la práctica directa en la terminal, aprendiendo a ejecutar, detener e inspeccionar contenedores mientras exploramos su anatomía básica.

---

### Lecturas adicionales

- **Visión general de Docker**: [https://docs.docker.com/engine/docker-overview/](https://docs.docker.com/engine/docker-overview/)
- **El proyecto Moby**: [https://mobyproject.org/](https://mobyproject.org/)
- **Primeros pasos con Docker**: [https://www.docker.com/get-started](https://www.docker.com/get-started)
- **Docker Desktop**: [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/)
- **Cloud-Native Computing Foundation (CNCF)**: [https://www.cncf.io/](https://www.cncf.io/)
- **containerd**: [https://containerd.io/](https://containerd.io/)
- **Mirantis Kubernetes Engine 4**: [https://www.mirantis.com/software/mirantis-kubernetes-engine/](https://www.mirantis.com/software/mirantis-kubernetes-engine/)
- **Documentación de Docker Rootless**: [https://docs.docker.com/engine/security/rootless/](https://docs.docker.com/engine/security/rootless/)
- **Contenedores efímeros en Kubernetes**: [https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)
- **Firma de imágenes y seguridad en la cadena de suministro**:
  - Notary v2: [https://github.com/notaryproject/notaryproject](https://github.com/notaryproject/notaryproject)
  - Cosign / Sigstore: [https://docs.sigstore.dev](https://docs.sigstore.dev/)
- **cgroups v2 en la práctica**: [https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- **Herramientas de generación de SBOM**:
  - Syft: [https://github.com/anchore/syft](https://github.com/anchore/syft)
  - CycloneDX: [https://cyclonedx.org/](https://cyclonedx.org/)

---

### Preguntas

Responde a las siguientes preguntas para evaluar tu progreso de aprendizaje:

1. **¿Qué afirmaciones son correctas respecto a los contenedores? (Pueden aplicar varias respuestas)**
   - a. Un contenedor es esencialmente lo mismo que una máquina virtual ligera.
   - b. Un contenedor solo puede ejecutarse en un host Linux.
   - c. Un contenedor puede ejecutar exactamente un proceso y no más.
   - d. El proceso principal en un contenedor siempre tiene el PID 1 dentro del *namespace* de ese contenedor.
   - e. Un contenedor es uno o más procesos encapsulados por *namespaces* de Linux y restringidos por *cgroups*.

2. **Con tus propias palabras y usando analogías, explica qué es un contenedor.** *(Pista: Compáralo con un contenedor de transporte físico o una forma estandarizada de empaquetar).*

3. **¿Por qué se considera que los contenedores han cambiado las reglas del juego en TI? Nombra tres o cuatro razones clave.** *(Piensa en portabilidad, reducción de fricción, integración con la nube, inmutabilidad y seguridad).*

4. **¿Qué significa la afirmación "Si un contenedor se ejecuta en una plataforma determinada, se ejecuta en cualquier lugar"? Proporciona dos o tres razones.**

5. **¿Verdadero o falso? "Los contenedores Docker solo son útiles para aplicaciones modernas de nueva creación (*greenfield*) basadas en microservicios." Justifica tu respuesta.**

6. **¿Cuánto suelen ahorrar las empresas en mantenimiento al contenerizar sus aplicaciones heredadas (*legacy*)?**
   - a. 20%
   - b. 33%
   - c. 50%
   - d. 75%

7. **¿En qué dos conceptos fundamentales de Linux se basan los contenedores?** *(Pista: Incluye un método para aislar procesos y otro para controlar el uso de recursos).*

8. **¿Qué sistemas operativos son compatibles actualmente con Docker Desktop?**

9. **Nombra al menos dos nuevas características o prácticas de contenerización consolidadas a partir de 2022 y explica brevemente su importancia.**

---

### Respuestas

1. **Las respuestas correctas son D y E:**
   - **d.** Dentro del propio *namespace* del contenedor, el proceso principal tiene el PID 1.
   - **e.** Un contenedor está compuesto por uno o más procesos encapsulados mediante *namespaces* de Linux y limitados por *cgroups*.

2. **Explicación con analogía:**  
   Una analogía clara es el contenedor de carga estandarizado utilizado en el comercio global. Al igual que los contenedores físicos, los contenedores de software proporcionan un mecanismo de empaquetado uniforme. Una vez que los desarrolladores introducen una aplicación y sus dependencias dentro del contenedor, este puede transportarse y ejecutarse en cualquier lugar que admita contenedores, simplificando la logística y garantizando la coherencia entre entornos.

3. **Razones por las que los contenedores revolucionaron TI:**
   - Estandarizan y aíslan aplicaciones y dependencias, eliminando conflictos de entorno.
   - Son altamente portables (ejecución idéntica en entornos locales, nubes públicas o híbridas).
   - Fomentan lanzamientos rápidos y consistentes gracias a la inmutabilidad de las imágenes.
   - Refuerzan la seguridad mediante *namespaces*, *cgroups* y herramientas de escaneo de vulnerabilidades.

4. **Justificación de portabilidad ("Build, ship, run anywhere"):**
   - El contenedor empaqueta todas las dependencias dentro de su imagen, siendo completamente autosuficiente.
   - Cumple con estándares abiertos reconocidos (OCI), lo que permite su ejecución en cualquier motor compatible.
   - Abstrae las particularidades del sistema operativo anfitrión, minimizando discrepancias entre servidores o proveedores de nube.

5. **Falso:**  
   Los contenedores son igualmente beneficiosos para aplicaciones monolíticas o heredadas (*legacy*). Las empresas reportan ahorros de más del 50% en costes de mantenimiento y ciclos de entrega mucho más rápidos al migrar sistemas tradicionales a contenedores (*lift and shift*), sin necesidad de reescribir la lógica de la aplicación.

6. **La respuesta correcta es C (50% o más):**  
   Diversos casos de estudio demuestran reducciones de al menos un 50% a 60% en los costos de mantenimiento y hasta un 90% en los tiempos de lanzamiento.

7. **Conceptos clave de Linux:**  
   **Namespaces** (para aislar procesos, red, usuarios, puntos de montaje, etc.) y **cgroups** (para controlar, limitar y medir el consumo de recursos como CPU y memoria).

8. **Sistemas operativos compatibles con Docker Desktop:**  
   **macOS**, **Windows** y **Linux**.

9. **Nuevas características clave post-2022:**
   - **Contenedores efímeros en Kubernetes**: Permiten depuración en tiempo real acoplando herramientas de diagnóstico a Pods en producción sin reiniciarlos.
   - **Modo Docker Rootless**: Ejecución sin privilegios de root para reducir la superficie de ataque.
   - **Adopción de cgroups v2**: Mayor precisión y aislamiento en entornos multitenant.
   - **Firma de imágenes (Cosign / Notary v2) y SBOM (Syft / CycloneDX)**: Garantía criptográfica y trazabilidad completa frente a ataques en la cadena de suministro de software.
