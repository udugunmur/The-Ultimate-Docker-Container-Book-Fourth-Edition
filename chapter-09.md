# Parte 3: Fundamentos de la Orquestación
## Capítulo 9: Comprensión de la arquitectura de aplicaciones distribuidas

Este capítulo introduce la **arquitectura de aplicaciones distribuidas**, explorando los patrones de diseño y las mejores prácticas indispensables para operar estos sistemas con éxito. También analizaremos los requisitos operativos indispensables para entornos de producción.

Podrías preguntarte: ¿qué relación tiene esto con los contenedores de Docker? Es una duda comprensible. A primera vista parecen temas desconectados, pero como descubrirás, los contenedores hacen que sea extraordinariamente sencillo construir sistemas compuestos por múltiples servicios independientes, lo que conduce de forma natural a despliegues distribuidos a través de clústeres de nodos o máquinas virtuales. Es fundamental comprender la complejidad intrínseca que esto conlleva y evitar los errores más comunes.

---

### Temas tratados en este capítulo:
- ¿Qué es una arquitectura de aplicaciones distribuidas?
- Patrones de diseño y mejores prácticas
- Operación y ejecución en producción
- Patrones modernos para microservicios

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Diseñar y esquematizar la arquitectura general de una aplicación distribuida identificando sus patrones clave.
- Detectar vulnerabilidades y fallos de diseño en arquitecturas acopladas.
- Enumerar los patrones arquitectónicos fundamentales para resolver los retos de los sistemas distribuidos.
- Identificar al menos cuatro patrones indispensables para garantizar la preparación para producción (*production-readiness*).

---

### ¿Qué es una arquitectura de aplicaciones distribuidas?

Antes de profundizar, conviene recordar una famosa advertencia formulada por **Martin Fowler** en su obra clásica *Patterns of Enterprise Application Architecture* (2002):

> **Primera Ley del Diseño de Objetos Distribuidos**:  
> *«¡No distribuyas tus objetos!»*

Cuando Fowler acuñó esta máxima, los sistemas de objetos distribuidos como CORBA, DCOM y RMI prometían transparencia total, haciendo que las llamadas remotas por red parecieran invocaciones a métodos locales en memoria. En la práctica, esa abstracción falló: la latencia de red, los fallos parciales, los problemas de versionado y el acoplamiento en los despliegues convirtieron estos sistemas en estructuras extremadamente frágiles.

Muchos equipos aprendieron por las malas que la distribución no es un simple detalle técnico: **la distribución cambia radicalmente los modos de fallo de un sistema**. Aunque la tecnología ha evolucionado de forma drástica, la lección principal sigue plenamente vigente: la distribución introduce una complejidad que debe diseñarse de forma explícita.

---

### Definición de terminología clave

| Concepto | Descripción |
| :--- | :--- |
| **Máquina Virtual (VM)** | Simulación por software de una computadora física ejecutada sobre un host mediante un hipervisor, proporcionando un sistema operativo y recursos aislados. |
| **Clúster (*Cluster*)** | Conjunto de servidores interconectados que colaboran como un sistema unificado para ofrecer alta disponibilidad, escalabilidad y mayor rendimiento. |
| **Nodo (*Node*)** | Servidor o instancia individual (física o virtual) dentro de un clúster de computación. |
| **Red (*Network*)** | Conjunto de dispositivos interconectados que intercambian datos. En nuestro contexto, representa las rutas de comunicación físicas y definidas por software (SDN) entre nodos y procesos. |
| **Puerto (*Port*)** | Punto de conexión de comunicación lógico en un dispositivo de red asociado a un protocolo (HTTP, TCP, UDP) identificado por un número único. |
| **Servicio (*Service*)** | Pieza de software autónoma que implementa un conjunto específico de funcionalidades consumidas por otras partes de la aplicación. |

*Tabla 9.1: Conceptos esenciales en sistemas distribuidos*

---

### Arquitectura monolítica frente a distribuida

#### Arquitectura monolítica tradicional
Históricamente, las aplicaciones empresariales se han construido como un único programa fuertemente acoplado que se ejecuta en un servidor específico con una dirección IP fija o un nombre de host estático:

*Figura 9.1 – Arquitectura de aplicación monolítica*

En este modelo, el servidor `blue-box-12a` (`172.52.13.44`) ejecuta una aplicación monolítica `pet-shop` compuesta por un módulo principal y librerías fuertemente vinculadas.

#### Arquitectura distribuida basada en microservicios
En un sistema distribuido, la aplicación se divide en múltiples servicios desacoplados (`pet-api`, `pet-web`, `pet-inventory`) que se ejecutan en múltiples instancias distribuidas a través de un clúster de nodos identificados por identificadores únicos (UUIDs):

*Figura 9.2 – Arquitectura de aplicación distribuida*

Los contenedores Docker y las plataformas de orquestación (como Kubernetes) simplifican enormemente la gestión y el despliegue de estos sistemas distribuidos, aunque la arquitectura subyacente sigue requiriendo patrones rigurosos para gestionar su complejidad operativa.

---

### Patrones y mejores prácticas

#### 1. Componentes débilmente acoplados (*Loosely Coupled Components*)
Dividir un sistema complejo en componentes más pequeños e independientes. Cada servicio expone una interfaz pública bien definida (API REST, gRPC, colas de mensajería) y no asume nada sobre el funcionamiento interno de los demás servicios, facilitando pruebas con *mocks* o *stubs* y despliegues autónomos.

#### 2. Componentes con estado frente a sin estado (*Stateful vs. Stateless*)
- **Componentes con estado (*Stateful*)**: Crean, modifican o almacenan datos persistentes (bases de datos, almacenes de archivos). Son más complejos de escalar y migrar entre nodos.
- **Componentes sin estado (*Stateless*)**: No almacenan datos persistentes localmente. Pueden escalarse dinámicamente hacia arriba o hacia abajo, destruirse y reiniciarse en cualquier nodo del clúster de forma inmediata.

> **Principio de diseño**: Diseña la mayor cantidad posible de servicios como *stateless* y empuja los componentes *stateful* hacia los límites de la arquitectura.

#### 3. Descubrimiento de servicios (*Service Discovery*)
En un clúster dinámico, las instancias de los servicios se crean, destruyen y migran constantemente de nodo, cambiando sus direcciones IP y puertos:
- **Acoplamiento rígido (*Hardwiring*)**: Configurar IPs estáticas en archivos de configuración falla en entornos distribuidos (*Figura 9.3*).
- **Descubrimiento dinámico**: Se utiliza un registro central o servicio DNS interno que mantiene la topología actualizada del clúster (*Figura 9.4*). Los servicios consultan al DNS por nombre lógico (por ejemplo, `http://pet-api/`) para obtener la dirección dinámica de destino.

#### 4. Enrutamiento (*Routing*)
Mecanismo para enviar paquetes de datos desde un componente de origen a uno de destino a través de diferentes capas del modelo OSI:
- **Capa 2 / 3**: Enrutamiento de bajo nivel basado en direcciones MAC e IP.
- **Capa 4 (TCP/UDP)**: Enrutamiento por puertos de transporte.
- **Capa 7 (Aplicación)**: Enrutamiento de alto nivel basado en URLs, rutas HTTP y encabezados (por ejemplo, `/pets` -> `pet-service`).

#### 5. Balanceo de carga (*Load Balancing*)
Distribuye el tráfico entrante equitativamente entre las múltiples instancias activas y saludables de un servicio (*Figura 9.5*):
- **Algoritmo Round-Robin**: Asigna peticiones de forma rotatoria secuencial (instancia 1, 2, ..., n).
- **Comprobación de salud (*Health Checking*)**: El balanceador solo redirige tráfico a instancias que hayan superado las pruebas de disponibilidad.

---

### Programación defensiva (*Defensive Programming*)

En un sistema distribuido, las dependencias de red y los servicios de terceros pueden fallar en cualquier momento. El código debe asumir los fallos de forma explícita:

#### Reintentos con retroceso exponencial (*Retries with Exponential Backoff*)
Si una llamada remota falla o agota el tiempo de espera (*timeout*), el cliente reintenta la petición tras un intervalo breve. Si vuelve a fallar, incrementa progresivamente el tiempo de espera antes de desistir y activar un modo de servicio degradado (*degraded mode*).

#### Registro centralizado de eventos (*Logging*)
Categorizar los registros por niveles de severidad (`DEBUG`, `INFO`, `WARNING`, `ERROR`, `FATAL`) y enviarlos a un sistema de agregación central. **Nunca registrar datos personales (PII), datos de salud protegidos (PHI) ni información financiera confidencial (SFI)**.

#### Manejo de errores y principio *Fail Fast*
Descubrir los errores irrecuperables lo antes posible (por ejemplo, validando parámetros de entrada) y abortar la operación de inmediato, registrando detalles estructurados en `STDERR`/`STDOUT` y devolviendo códigos de error claros al llamador.

#### Redundancia
Eliminar cualquier punto único de fallo (*Single Point of Failure* o SPOF) ejecutando múltiples réplicas de cada microservicio, base de datos y componente de infraestructura (balanceadores, routers).

> *«La cadena es tan fuerte como su eslabón más débil.»*  
> — Thomas Reid, *Essays on the Intellectual Powers of Man* (1786).

#### Pruebas de salud y autorreparación (*Health Checks & Auto-Healing*)
El orquestador sondea periódicamente cada contenedor. Si una instancia no responde o devuelve un estado no saludable, el sistema la destruye automáticamente y crea una nueva instancia en su lugar.

#### Patrón Circuit Breaker (*Disyuntor*)
Evita fallos en cascada en efecto dominó (*Figura 9.6*). Si las llamadas a un servicio remoto dependiente superan un umbral de fallos consecutivos, el circuito se "abre" (*trips*), fallando de inmediato las peticiones posteriores sin sobrecargar al servicio averiado, permitiéndole recuperarse.

#### Limitador de tasa (*Rate Limiter*)
Controla la cantidad máxima de peticiones permitidas en una ventana de tiempo, protegiendo los servicios contra picos repentinos de tráfico y ataques de denegación de servicio.

#### Patrón Bulkhead (*Mamparo*)
Aísla grupos de recursos (pools de conexiones, hilos, memoria) para que el fallo o saturación en un área del sistema no agote los recursos necesarios para otras operaciones críticas.

---

### Operación y ejecución en producción

#### Observabilidad y monitorización
- **Coste del logging**: El registro exhaustivo genera volúmenes masivos de datos que impactan en almacenamiento y costes de red en la nube. Debe equilibrarse la granularidad con la eficiencia operativa.
- **Trazabilidad distribuida (*Distributed Tracing*)**: Identifica el recorrido completo de cada petición individual a través de la malla de microservicios, midiendo latencias y cuellos de botella por componente.
- **Métricas en tiempo real**: Paneles de control (*dashboards*) para supervisar métricas no funcionales (CPU, RAM, reinicios) y métricas de negocio (pedidos procesados, carritos abandonados).

---

### Estrategias de actualización de aplicaciones sin tiempo de inactividad

#### 1. Actualizaciones graduales (*Rolling Updates*)
Reemplaza progresivamente las instancias antiguas de un servicio por instancias nuevas. El balanceador de carga redirige el tráfico a las nuevas versiones a medida que superan los *health checks*, garantizando servicio continuo.

#### 2. Despliegues Azul-Verde (*Blue-Green Deployments*)
Se mantiene el entorno de producción actual (**Azul**) mientras se despliega la nueva versión completa en paralelo (**Verde**) (*Figura 9.7*). Tras ejecutar pruebas de humo (*smoke tests*) en Verde, el router conmuta el 100% del tráfico a la nueva versión. En caso de incidencia, la reversión al entorno Azul es instantánea.

#### 3. Lanzamientos Canario (*Canary Releases*)
Se despliega la nueva versión en paralelo y se le redirige un porcentaje muy reducido del tráfico real (por ejemplo, 1%). Se monitoriza de cerca su comportamiento y se incrementa el porcentaje gradualmente (5%, 25%, 50%, 100%) hasta sustituir completamente la versión anterior.

#### 4. Cambios de esquema irreversibles en bases de datos
Los cambios estructurales en bases de datos no deben desplegarse simultáneamente con el código que los requiere. Se aplica una estrategia en tres fases (*Figura 9.8*):
1. **Fase 1**: Desplegar una migración de esquema y datos compatible hacia atrás (*backward-compatible*).
2. **Fase 2**: Desplegar el nuevo código de la aplicación que utiliza la nueva estructura.
3. **Fase 3**: Limpiar y eliminar las estructuras obsoletas de compatibilidad en la base de datos.

---

### Patrones modernos para microservicios

- **API Gateway**: Punto único de entrada para clientes externos que gestiona enrutamiento, autenticación, limitación de tasa y agregación de peticiones hacia los microservicios internos.
- **Patrón Saga**: Gestión de transacciones distribuidas entre múltiples servicios sin bloqueos de dos fases (*2PC*), coordinando transacciones locales mediante eventos y acciones de compensación en caso de error.
- **CQRS (Command Query Responsibility Segregation)**: Separa los modelos de escritura (Comandos) de los modelos de lectura (Consultas), permitiendo optimizar el rendimiento y escalabilidad de forma independiente.
- **Event Sourcing**: Almacena los cambios de estado como una secuencia inmutable de eventos de negocio en lugar de sobrescribir el estado final, ofreciendo auditoría completa y capacidad de reproducción histórica.
- **Patrón Strangler (Estrangulador)**: Migración incremental y paulatina de sistemas monolíticos antiguos hacia microservicios, reemplazando funcionalidades gradualmente hasta desmantelar el monolito sin requerir una reescritura total desde cero.

---

### Resumen

En este capítulo aprendimos:
- Los conceptos fundamentales de las arquitecturas distribuidas frente a las estructuras monolíticas tradicionales.
- La relevancia de la Primera Ley de Fowler y el coste inherente de la complejidad distribuida.
- Patrones arquitectónicos esenciales: desacoplamiento, servicios sin estado (*stateless*), descubrimiento por DNS, enrutamiento y balanceo de carga.
- Técnicas de programación defensiva: reintentos con *backoff*, redundancia, *health checks*, *circuit breakers*, *rate limiters* y *bulkheads*.
- Prácticas para entornos productivos: observabilidad, gestión de costes de logs, despliegues *rolling*, *blue-green*, lanzamientos canario y migraciones de datos en fases.
- Patrones modernos de microservicios: *API Gateway*, *Saga*, *CQRS*, *Event Sourcing* y *Strangler Pattern*.

---

### Lecturas adicionales

- **Patrón Circuit Breaker (Martin Fowler)**: [http://bit.ly/1NU1sgW](http://bit.ly/1NU1sgW)
- **El modelo OSI explicado**: [http://bit.ly/1UCcvMt](http://bit.ly/1UCcvMt)
- **Despliegues Blue-Green (Martin Fowler)**: [http://bit.ly/2r2IxNJ](http://bit.ly/2r2IxNJ)

---

### Preguntas

1. **¿Cuándo y por qué cada elemento en una arquitectura de aplicaciones distribuidas debe ser redundante?**
2. **¿Por qué necesitamos servicios de descubrimiento basados en DNS en sistemas distribuidos?**
3. **¿Qué es un *Circuit Breaker* y por qué es indispensable?**
4. **¿Cuáles son las diferencias más importantes entre una aplicación monolítica y una distribuida?**
5. **¿Qué es un despliegue Azul-Verde (*Blue-Green*)?**
6. **¿Cuál es la diferencia entre componentes con estado (*stateful*) y sin estado (*stateless*), y por qué se prefieren los segundos?**
7. **Explica el propósito de los *health checks* y cómo contribuyen a la autorreparación (*auto-healing*).**
8. **Describe el patrón *Circuit Breaker* y cómo se complementa con *Rate Limiters* y *Bulkheads*.**
9. **¿Qué son los lanzamientos Canario y en qué se diferencian de los despliegues Azul-Verde?**
10. **¿Por qué es fundamental el logging en aplicaciones distribuidas y qué precauciones deben tomarse respecto a sus costes y datos confidenciales?**

---

### Respuestas

1. **Necesidad de redundancia**:  
   En entornos productivos de alta disponibilidad, la probabilidad de fallo de un componente individual crece con la cantidad de piezas interconectadas. Es inevitable que cualquier servidor, contenedor o enlace de red falle con el tiempo. La redundancia en cada capa garantiza que el fallo de una instancia no provoque la caída global del sistema.

2. **Propósito del DNS y descubrimiento de servicios**:  
   En clústeres dinámicos, las instancias de los servicios se crean, destruyen y trasladan continuamente entre nodos, modificando sus direcciones IP y puertos. Los clientes no pueden depender de configuraciones estáticas; el servicio DNS centraliza la resolución de nombres lógicos a ubicaciones físicas dinámicas en tiempo real.

3. **Función del Circuit Breaker**:  
   Mecanismo que interrumpe temporalmente las llamadas hacia un servicio que está fallando o no responde, evitando que las solicitudes se acumulen y provoquen una degradación o fallo en cascada en los servicios dependientes.

4. **Monolito frente a sistema distribuido**:  
   - **Monolito**: Fácil de desplegar y operar inicialmente (un único artefacto), pero difícil de escalar componentes concretos y riesgoso de actualizar sin afectar a todo el sistema.
   - **Distribuido**: Permite escalado granular y despliegues autónomos por equipo, pero introduce alta complejidad de red, fallos parciales, consistencia eventual y mayor esfuerzo de observabilidad.

5. **Despliegue Azul-Verde**:  
   Técnica de actualización con cero tiempo de inactividad que mantiene la versión actual en producción (**Azul**) mientras despliega la nueva versión en un entorno idéntico aislado (**Verde**). Tras superar las pruebas, el enrutador conmuta todo el tráfico a Verde, permitiendo una reversión instantánea a Azul si se presentan problemas.

6. **Stateful vs Stateless**:  
   Los componentes *stateful* gestionan datos persistentes que requieren sincronización y preservación, dificultando su migración. Los componentes *stateless* no retienen datos entre peticiones, lo que permite escalarlos, reiniciarlos o distribuirlos en cualquier nodo del clúster de forma inmediata y sin pérdida de información.

7. **Health Checks y Auto-healing**:  
   Sondeos periódicos que verifican la operatividad de cada contenedor. Permiten al balanceador de carga retirar de la rotación a las instancias enfermas y al orquestador destruirlas y reemplazarlas automáticamente por réplicas sanas sin intervención manual.

8. **Resiliencia con Circuit Breakers, Rate Limiters y Bulkheads**:  
   El *Circuit Breaker* corta las peticiones a dependencias caídas; el *Rate Limiter* restringe el caudal de entrada para mitigar picos de tráfico; y el *Bulkhead* aísla grupos de recursos críticos para evitar que el agotamiento de un componente afecte al resto del sistema.

9. **Lanzamientos Canario frente a Blue-Green**:  
   El despliegue Azul-Verde conmuta el 100% del tráfico de una sola vez tras validar el nuevo entorno. El lanzamiento Canario redirige una fracción mínima del tráfico real (1-5%) a la nueva versión y la incrementa paulatinamente tras verificar su estabilidad en producción, minimizando el impacto de posibles errores.

10. **Consideraciones de Logging**:  
    Permite reconstruir flujos de ejecución y diagnosticar fallos en sistemas donde no es posible depurar interactivamente. Sin embargo, el volumen masivo de logs genera costes elevados de red y almacenamiento en la nube, requiriendo un balance adecuado de granularidad y la exclusión estricta de datos personales (PII), de salud (PHI) o financieros (SFI).

