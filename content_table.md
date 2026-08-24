# Tabla de Contenidos

## Parte 1: Introducción

### Capítulo 1: ¿Qué son los contenedores y por qué debería usarlos?
- ¿Qué son los contenedores?
- ¿Por qué son importantes los contenedores?
- ¿Cuál es el beneficio de usar contenedores para mí o para mi empresa?
- El proyecto Moby
- Productos de Docker
- Arquitectura de contenedores
- Novedades en la contenerización
- Resumen / Lecturas adicionales / Preguntas / Respuestas

### Capítulo 2: Configuración del entorno de trabajo
- Requisitos técnicos
- Distinción de los principales sistemas operativos
- La consola de comandos de Linux (*shell*)
- PowerShell para Windows
- Instalación y uso de un gestor de paquetes
- Instalación de Git y clonación del repositorio de código
- Selección e instalación de un editor de código
- Instalación de Docker Desktop en macOS, Windows o Linux
- Uso de Docker con WSL 2 en Windows
- Instalación de Docker Toolbox
- Habilitación de Kubernetes en Docker Desktop
- Instalación de Podman
- Instalación de minikube
- Instalación de kind
- Resumen / Lecturas adicionales / Preguntas / Respuestas

---

## Parte 2: Fundamentos de la Contenerización

### Capítulo 3: Dominando los Contenedores
- Requisitos técnicos
- Ejecución del primer contenedor
- Iniciar, detener y eliminar contenedores
  - Ejecutar un contenedor de preguntas de trivia aleatorias
  - Listar contenedores
  - Detener e iniciar contenedores
  - Eliminar contenedores
  - Inspeccionar contenedores
  - Ejecutar comandos dentro de un contenedor en ejecución
  - Conectarse (*attach*) a un contenedor en ejecución
  - Obtener los logs de un contenedor
- La anatomía de los contenedores
- Resumen / Lecturas adicionales / Preguntas / Respuestas

### Capítulo 4: Creación y gestión de imágenes de contenedores
- ¿Qué son las imágenes?
- Creación de imágenes de Docker
- Contenerización de una aplicación heredada (*legacy*) mediante *lift and shift*
- Compartir y distribuir imágenes
- Prácticas de seguridad en la cadena de suministro (*supply chain security*)
- Resumen / Preguntas / Respuestas

### Capítulo 5: Volúmenes de datos y configuración
- Requisitos técnicos
- Creación y montaje de volúmenes de datos
- Compartir datos entre contenedores
- Uso de volúmenes del host (*host volumes*)
- Definición de volúmenes en imágenes
- Configuración de contenedores
- Almacenamiento persistente y patrones de contenedores con estado (*stateful*)
- Resumen / Lecturas adicionales / Preguntas / Respuestas

### Capítulo 6: Depuración de código en ejecución dentro de contenedores
- Requisitos técnicos
- Evolución y pruebas de código en ejecución dentro de un contenedor
- Reinicio automático del código ante cambios (*auto-restarting*)
- Depuración de código línea por línea dentro de un contenedor
- Instrumentación de tu código para generar información de logging significativa
- Uso de OpenTelemetry y Jaeger para monitorización y resolución de problemas
- Resumen / Preguntas / Respuestas

### Capítulo 7: Pruebas de aplicaciones en ejecución en contenedores
- Requisitos técnicos
- Beneficios de realizar pruebas en contenedores
- Tipos de pruebas para aplicaciones contenerizadas
- Herramientas, frameworks y entornos de prueba
- Buenas prácticas para configurar un entorno de pruebas
- Consejos para la depuración y resolución de incidencias
- Desafíos y consideraciones al probar aplicaciones en contenedores
- Casos de estudio
- Resumen / Preguntas / Respuestas

### Capítulo 8: Aumento de la productividad con trucos y consejos de Docker
- Requisitos técnicos
- Mantener limpio tu entorno de Docker
- Uso del archivo `.dockerignore`
- Ejecución de tareas administrativas simples en un contenedor
- Limitación del uso de recursos de un contenedor
- Evitar ejecutar un contenedor como root
- Ejecución de comandos de la CLI de Docker desde dentro de Docker
- Optimización del proceso de construcción (*build*)
- Escaneo de vulnerabilidades y secretos
- Ejecución de tu entorno de desarrollo en un contenedor
- Resumen / Preguntas / Respuestas

---

## Parte 3: Fundamentos de Orquestación

### Capítulo 9: Aprendiendo sobre la arquitectura de aplicaciones distribuidas
- ¿Qué es una arquitectura de aplicaciones distribuidas?
- Patrones y mejores prácticas
- Ejecución en producción
- Patrones modernos de microservicios
- Resumen / Lecturas adicionales / Preguntas / Respuestas

### Capítulo 10: Uso de redes de host único (*Single-Host Networking*)
- Requisitos técnicos
- Análisis del modelo de red de contenedores (*Container Network Model*)
- Cortafuegos y filtrado de red (*firewalling*)
- Trabajo con la red *bridge*
- Tipos de redes *host* y *null*
- Ejecución dentro de un espacio de nombres de red (*network namespace*) existente
- Gestión de puertos de contenedores
- Enrutamiento a nivel HTTP utilizando un proxy inverso
- Resumen / Lecturas adicionales / Preguntas / Respuestas

### Capítulo 11: Gestión de contenedores con Docker Compose
- Requisitos técnicos
- Desmitificando la orquestación declarativa frente a la imperativa de contenedores
- Qué ha cambiado desde la última edición del libro
- Ejecución de una aplicación multiservicio
- Construcción de imágenes con Docker Compose
- Ejecución de una aplicación con Docker Compose
- Escalado de un servicio
- Construcción y publicación (*push*) de una aplicación
- Uso de sobrescrituras (*overrides*) en Docker Compose
- Modularización de aplicaciones con `include`
- Cuándo usar Docker Compose frente a un sistema de orquestación completo
- Resumen / Lecturas adicionales / Preguntas / Respuestas

### Capítulo 12: Envío de logs y monitorización de contenedores
- Requisitos técnicos
- Por qué importan el logging y la monitorización
- Envío de logs de contenedores (*log shipping*)
- Envío de logs del demonio de Docker
- Consulta de logs centralizados con Kibana
- Recolección y extracción de métricas con Prometheus
- Monitorización de una aplicación contenerizada
- Observabilidad y monitorización de seguridad
- Resumen / Preguntas / Respuestas

### Capítulo 13: Seguridad en contenedores
- Requisitos técnicos
- Seguridad en la cadena de suministro (*Supply chain security*)
- Escaneo de vulnerabilidades en imágenes y confianza en el contenido (*content trust*)
- Prácticas de endurecimiento (*hardening*) de contenedores
- Gestión de secretos
- Herramientas de seguridad en tiempo de ejecución (*runtime security*)
- Resumen / Referencias / Preguntas / Respuestas

### Capítulo 14: Introducción a la orquestación de contenedores
- ¿Qué son los orquestadores y por qué los necesitamos?
- Las tareas de un orquestador
- Descripción general de los orquestadores más populares
- Tendencias emergentes en orquestación
- Resumen / Lecturas adicionales / Preguntas / Respuestas

---

## Parte 4: Docker, Kubernetes y la Nube

### Capítulo 15: Introducción a Kubernetes
- Requisitos técnicos
- Comprensión de la arquitectura de Kubernetes
- Nodos maestros (*master nodes*) de Kubernetes
- Nodos del clúster
- Introducción a Kubernetes local
- Introducción a los Pods
- Kubernetes ReplicaSet
- Kubernetes Deployments
- Kubernetes Services
- Enrutamiento basado en contexto
- Herramientas populares: GitOps, Helm 3 y Kustomize
- Resumen / Lecturas adicionales / Preguntas / Respuestas

### Capítulo 16: Despliegue, actualización y seguridad de una aplicación con Kubernetes
- Requisitos técnicos
- Desplegando nuestra primera aplicación
- Definición de sondas de vida (*liveness*), preparación (*readiness*) e inicio (*startup*)
- Despliegues sin tiempo de inactividad (*Zero-downtime deployments*)
- Mejores prácticas de seguridad
- Resumen / Lecturas adicionales / Preguntas / Respuestas

### Capítulo 17: Ejecución de una aplicación contenerizada en la nube
- Requisitos técnicos
- Preparación de imágenes multiplataforma
- ¿Por qué elegir un servicio administrado de Kubernetes?
- Descripción general de los servicios administrados de Kubernetes cubiertos en este capítulo
- Ejecución de una aplicación contenerizada en Amazon EKS
- Ejecución de una aplicación contenerizada en Microsoft Azure AKS
- Ejecución de una aplicación contenerizada en Google Kubernetes Engine (GKE)
- Contenedores sin servidor (*Serverless*) y el futuro de las arquitecturas nativas de la nube (*cloud-native*)
- Resumen / Preguntas / Respuestas

### Capítulo 18: Monitorización y resolución de problemas de una aplicación en producción
- Requisitos técnicos
- Instrumentación de servicios con OpenTelemetry
- Recolección y visualización de métricas con Prometheus y Grafana
- Definición de alertas y guías de actuación (*runbooks*)
- Resumen / Preguntas / Respuestas

### Capítulo 19: IA y Automatización en DevOps
- Requisitos técnicos
- ¿Por qué IA para DevOps?
- Casos de uso y patrones de IA en DevOps
- Herramientas y frameworks
- Pasos prácticos de implementación
- Resumen / Preguntas / Respuestas

### Capítulo 20: Patrones de contenerización en el mundo real
- Modernización de sistemas legacy: Transformación de monolitos a contenedores
- Microservicios en acción: Descomposición y escalado de aplicaciones complejas
- CI/CD en la práctica: Aceleración de la entrega de software con contenedores
- Historias de migración a la nube: Despliegue de aplicaciones contenerizadas en AWS, Azure y GCP
- Orquestación en producción: Lecciones aprendidas de Kubernetes
- Seguridad en el contenedor: Enfoques del mundo real para la seguridad de aplicaciones
- Resumen / Epílogo: Reflexiones finales / Próximos pasos / Nota del autor

---

### Capítulo 21: Desbloquea tus beneficios exclusivos
- Desbloquea los beneficios gratuitos de este libro en tres sencillos pasos
- Índice alfabético
