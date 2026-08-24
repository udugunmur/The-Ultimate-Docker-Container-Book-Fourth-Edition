# Parte 3: Fundamentos de la Orquestación
## Capítulo 11: Gestión de contenedores con Docker Compose

En el capítulo anterior, aprendimos cómo funcionan las redes de contenedores en un solo host de Docker, exploramos el Container Network Model (CNM), profundizamos en las redes bridge y configuramos Traefik como proxy inverso para enrutamiento en capa 7.

Este capítulo introduce el concepto de aplicaciones compuestas por múltiples servicios independientes donde cada uno se ejecuta dentro de su propio contenedor, y cómo **Docker Compose** nos permite compilar, ejecutar y escalar estas aplicaciones de forma sencilla mediante un **enfoque declarativo**.

---

### Temas tratados en este capítulo:
- Desmitificación de la orquestación declarativa frente a la imperativa
- Ejecución de aplicaciones multi-servicio
- Construcción de imágenes con Docker Compose
- Ejecución y ciclo de vida de aplicaciones con Docker Compose
- Escalado horizontal de servicios (*Scale-out*)
- Compilación y publicación (*push*) de imágenes hacia registros
- Uso de archivos de sobreescritura (*overrides*) para múltiples entornos
- Modularización de aplicaciones con la directiva `include`
- Cuándo utilizar Docker Compose frente a un orquestador completo (Kubernetes / Swarm)

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Explicar con claridad la diferencia entre los enfoques imperativo y declarativo.
- Distinguir entre un contenedor individual y un servicio de Docker Compose.
- Crear tus propios manifiestos `compose.yml` en YAML para aplicaciones multi-servicio.
- Compilar, ejecutar, inspeccionar, escalar y destruir aplicaciones completas con comandos de Compose.
- Adaptar configuraciones para desarrollo, pruebas e integración continua usando *overrides* e `include`.
- Decidir con criterio cuándo Docker Compose es la herramienta idónea y cuándo se requiere un orquestador de clúster.

---

### Requisitos técnicos

El código fuente de este capítulo se encuentra en el repositorio del libro:  
[https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-11/solutions](https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition/tree/main/chapter-11/solutions)

Prepara el entorno local:
```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir chapter-11 && cd chapter-11
```

> **Docker Compose V2 integrado**: Ya no es necesario instalar el binario independiente `docker-compose`. Docker Compose está totalmente integrado en el CLI oficial de Docker (`docker compose`).

---

### Desmitificación de la orquestación declarativa frente a la imperativa

Docker Compose es la herramienta estándar para definir y ejecutar aplicaciones multi-contenedor en un único host (desarrollo local, CI/CD, pruebas de QA y demostraciones).

- **Enfoque Imperativo**: Describe paso a paso las instrucciones exactas que el sistema debe ejecutar y cómo reaccionar ante cada fallo (por ejemplo, encadenar múltiples comandos `docker container run`, `docker network create` y scripts bash complejos).
- **Enfoque Declarativo**: Define el **estado deseado** final de la aplicación en un archivo YAML (`compose.yml`). El motor de Docker se encarga de determinar las acciones necesarias para alcanzar y mantener dicho estado.

---

### Novedades en Docker Compose moderno (V2)

1. **Integración nativa en Docker CLI**: Se ejecuta mediante `docker compose` en lugar del antiguo `docker-compose`.
2. **Especificación Compose sin versión**: Ya no se requiere la directiva `version:` en la cabecera del archivo YAML.
3. **BuildKit por defecto**: Compilaciones más rápidas, paralelizadas, con soporte multi-plataforma, SBOMs y metadatos de procedencia.
4. **Palabra clave `include`**: Permite dividir pilas gigantescas en módulos independientes con sus propias rutas relativas y variables de entorno.

---

### Ejecución de una aplicación multi-servicio

#### Paso 1: Configuración de base de datos y herramienta de administración

1. Crear el directorio de trabajo:
   ```bash
   $ mkdir step-1 && cd step-1
   ```

2. Crear el archivo `compose.yml`:
   ```yaml
   services:
     db:
       image: postgres:alpine
       environment:
         POSTGRES_USER: postgres
         POSTGRES_PASSWORD: password
         POSTGRES_DB: pets
       volumes:
         - pg-data:/var/lib/postgresql/data
         - ./db:/docker-entrypoint-initdb.d

     pgadmin:
       image: dpage/pgadmin4
       ports:
         - "5050:80"
       environment:
         PGADMIN_DEFAULT_EMAIL: admin@acme.com
         PGADMIN_DEFAULT_PASSWORD: admin
       volumes:
         - pgadmin-data:/var/lib/pgadmin

   volumes:
     pg-data:
     pgadmin-data:
   ```
   *Figura 11.1 – Archivo Docker Compose simple*

3. Crear el script de inicialización SQL `db/init-db.sql`:
   ```sql
   CREATE TABLE images (
       id SERIAL PRIMARY KEY,
       name VARCHAR(255) NOT NULL,
       url VARCHAR(255) NOT NULL
   );

   INSERT INTO images (name, url) VALUES
   ('Elephant', 'https://images.unsplash.com/photo-1557050543-4d5f4e07ef46'),
   ('Lion', 'https://images.unsplash.com/photo-1546182990-dffeafbe841d'),
   ('Giraffe', 'https://images.unsplash.com/photo-1538121609141-9c8e5eac3a55'),
   ('Zebra', 'https://images.unsplash.com/photo-1501705388883-4ed8a543392c'),
   ('Cheetah', 'https://images.unsplash.com/photo-1534177616072-ef7dc120449d'),
   ('Hippopotamus', 'https://images.unsplash.com/photo-1575550959106-5a7defe28b56'),
   ('Rhinoceros', 'https://images.unsplash.com/photo-1534567153574-2b12153a87f0'),
   ('Leopard', 'https://images.unsplash.com/photo-1561731216-c3a4d99437d5'),
   ('Buffalo', 'https://images.unsplash.com/photo-1547721064-da6cfb341d50'),
   ('Hyena', 'https://images.unsplash.com/photo-1579762715118-a6f17e093365'),
   ('Warthog', 'https://images.unsplash.com/photo-1582845512747-e42001c95638'),
   ('Baboon', 'https://images.unsplash.com/photo-1540573133985-87b6da6d54a9');
   ```
   *Figura 11.2 – Script de inicialización de base de datos*

4. Iniciar la aplicación con Docker Compose:
   ```bash
   $ docker compose up
   ```
   *Figura 11.3, Figura 11.4, Figura 11.5 y Figura 11.6 – Descarga de imágenes, creación de red `step1_default`, volúmenes e inicio de contenedores*

5. Abrir en el navegador `http://localhost:5050` e iniciar sesión con `admin@acme.com` / `admin`. Registrar un nuevo servidor conectando al host `db`, puerto `5432`, usuario `postgres` y contraseña `password` (*Figura 11.7*).

6. Detener la aplicación y eliminar volúmenes asociados:
   ```bash
   $ docker compose down -v
   ```
   *Figura 11.8: Detención y eliminación de recursos de la aplicación*

---

### Construcción de imágenes con Docker Compose

#### Paso 2: Creación de la aplicación web en Node.js / Express

1. Preparar la estructura:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-11
   $ mkdir step-2 && cd step-2
   $ cp -r ../step-1/db .
   $ cp ../step-1/compose.yml .
   ```

2. Crear `web/package.json`:
   ```json
   {
     "name": "wild-animals",
     "version": "2.0.0",
     "description": "Wild Animals of Massai Mara",
     "main": "src/server.js",
     "scripts": {
       "start": "node src/server.js"
     },
     "dependencies": {
       "express": "^4.19.2",
       "pg": "^8.11.5"
     }
   }
   ```
   *Figura 11.9 – package.json de la aplicación web*

3. Crear `web/src/server.js`:
   ```javascript
   const express = require('express');
   const { Pool } = require('pg');
   const path = require('path');

   const app = express();
   const port = process.env.PORT || 3000;

   app.use(express.static(path.join(__dirname, '../public')));

   const pool = new Pool({
     user: process.env.DB_USER || 'postgres',
     host: process.env.DB_HOST || 'localhost',
     database: process.env.DB_NAME || 'pets',
     password: process.env.DB_PASSWORD || 'password',
     port: process.env.DB_PORT || 5432,
   });

   app.get('/animal', async (req, res) => {
     try {
       const result = await pool.query('SELECT * FROM images ORDER BY RANDOM() LIMIT 1');
       if (result.rows.length === 0) {
         return res.status(404).send('No images found');
       }
       const animal = result.rows[0];
       res.sendFile(path.join(__dirname, '../public/index.html'));
     } catch (err) {
       console.error(err);
       res.status(500).send('Database error');
     }
   });

   app.get('/api/animal', async (req, res) => {
     try {
       const result = await pool.query('SELECT * FROM images ORDER BY RANDOM() LIMIT 1');
       res.json(result.rows[0]);
     } catch (err) {
       console.error(err);
       res.status(500).json({ error: 'Database error' });
     }
   });

   app.listen(port, () => {
     console.log(`DB_HOST: ${process.env.DB_HOST || 'localhost'}`);
     console.log(`Application listening on port ${port}`);
   });
   ```
   *Figura 11.10 – server.js de la aplicación web*

4. Crear `web/src/index.html` y los estilos en `web/public/css/main.css`:
   ```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
     <meta charset="UTF-8">
     <title>Wild Animals</title>
     <link rel="stylesheet" href="/css/main.css">
   </head>
   <body>
     <div class="container">
       <h1>Wild Animals of Massai Mara National Park</h1>
       <div id="image-container">
         <img id="animal-image" src="" alt="Animal Image">
         <p id="animal-name"></p>
       </div>
       <button onclick="fetchAnimal()">Refresh</button>
     </div>
     <script>
       async function fetchAnimal() {
         const response = await fetch('/api/animal');
         const data = await response.json();
         document.getElementById('animal-image').src = data.url;
         document.getElementById('animal-name').textContent = data.name;
       }
       fetchAnimal();
     </script>
   </body>
   </html>
   ```
   *Figura 11.11 – Plantilla index.html*

   ```css
   body {
     font-family: Arial, sans-serif;
     text-align: center;
     background-color: #f4f4f9;
     padding: 20px;
   }
   .container {
     max-width: 600px;
     margin: 0 auto;
   }
   #animal-image {
     max-width: 100%;
     height: auto;
     border-radius: 8px;
   }
   button {
     margin-top: 15px;
     padding: 10px 20px;
     font-size: 16px;
     cursor: pointer;
   }
   ```
   *Figura 11.12 – main.css*

5. Descargar las imágenes de animales en `web/public/images/`.

6. En `compose.yml`, exponer el puerto de la base de datos para pruebas locales:
   ```yaml
       ports:
         - "5432:5432"
   ```
   *Figura 11.13 – Mapeo de puerto en el servicio db*

7. Crear el `web/Dockerfile`:
   ```dockerfile
   FROM node:20-alpine
   WORKDIR /app
   COPY package*.json ./
   RUN npm install
   COPY . .
   EXPOSE 3000
   CMD ["npm", "start"]
   ```
   *Figura 11.16 – Dockerfile de la aplicación web*

8. Añadir la definición del servicio `web` en `compose.yml`:
   ```yaml
     web:
       image: gnschenker/ch11-web:1.0
       build: web
       ports:
         - "3000:3000"
       environment:
         DB_HOST: db
         DB_NAME: pets
         DB_USER: postgres
         DB_PASSWORD: password
       depends_on:
         - db
   ```
   *Figura 11.17 – Servicio web en compose.yml*

9. Compilar la imagen con Docker Compose:
   ```bash
   $ docker compose build web
   ```
   *Figura 11.18 – Compilación de la imagen mediante Docker Compose y BuildKit*

---

### Ejecución de la aplicación con Docker Compose

1. Iniciar toda la aplicación en segundo plano (*detached mode*):
   ```bash
   $ docker compose up -d
   ```

2. Listar los servicios en ejecución:
   ```bash
   $ docker compose ps
   ```
   *Figura 11.20 – Salida de docker compose ps*

3. Probar en el navegador accediendo a `http://localhost:3000/animal`.

4. Detener y limpiar recursos:
   ```bash
   $ docker compose down -v
   ```
   *Figura 11.21 – Limpieza con docker compose down -v*

> **Nombres de proyectos**: Por defecto, Compose prefija los recursos con el nombre del directorio (`step-2_`). Puedes cambiarlo con el flag `--project-name` o la variable `COMPOSE_PROJECT_NAME`:
> ```bash
> $ docker compose --project-name demo up --detach
> ```

---

### Escalado de servicios (*Scaling*)

Para escalar horizontalmente un servicio (por ejemplo, a 3 réplicas):
```bash
$ docker compose up --scale web=3
```

> [!WARNING]
> **Conflicto de puertos**: Si el servicio tiene un puerto fijo en el host (`3000:3000`), las réplicas adicionales fallarán al iniciar (*Figura 11.22*).

#### Solución con asignación dinámica de puertos (Paso 3):
1. Preparar la carpeta `step-3`:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-11
   $ mkdir step-3 && cd step-3
   $ cp -r ../step-2/ .
   ```

2. Modificar el mapeo de puertos en `compose.yml` para especificar solo el puerto del contenedor:
   ```yaml
       ports:
         - 3000
   ```

3. Iniciar y escalar:
   ```bash
   $ docker compose up -d --scale web=3
   ```
   *Figura 11.23 – Escalado exitoso a tres réplicas*

4. Inspeccionar los puertos efímeros asignados:
   ```bash
   $ docker compose ps
   ```
   *Figura 11.24 – Puertos asignados dinámicamente (e.g. 50162, 50165, 50166)*

5. Probar una de las réplicas:
   ```bash
   $ curl -4 localhost:50162
   ```

---

### Compilación y publicación de imágenes (Paso 4)

Podemos personalizar la configuración de compilación especificando un contexto y un Dockerfile alternativo (`Dockerfile.dev`):

1. Preparar `step-4`:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-11
   $ mkdir step-4 && cd step-4
   $ cp -r ../step-2 .
   $ cp web/Dockerfile web/Dockerfile.dev
   $ cp compose.yml compose.dev.yml
   ```

2. Configurar `compose.dev.yml`:
   ```yaml
       build:
         context: web
         dockerfile: Dockerfile.dev
       depends_on:
         - db
   ```
   *Figura 11.25 y Figura 11.26 – Configuración de compilación personalizada*

3. Compilar con el archivo alternativo:
   ```bash
   $ docker compose -f compose.dev.yml build
   ```

4. Publicar la imagen compilada a Docker Hub:
   ```bash
   $ docker login -u <tu-usuario-dockerhub>
   $ docker compose -f compose.dev.yml push
   ```
   *Figura 11.27 – Publicación de imágenes a Docker Hub*

---

### Uso de archivos de sobreescritura (*Overrides* - Paso 5)

Permite separar la configuración base común de las configuraciones específicas para cada entorno (Desarrollo, CI, Staging):

1. Preparar `step-5`:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-11
   $ mkdir step-5 && cd step-5
   $ cp -r ../step-2/ .
   ```

2. Crear `compose.base.yml`:
   ```yaml
   services:
     db:
       image: postgres:alpine
       volumes:
         - pg-data:/var/lib/postgresql/data
     web:
       build: web
       image: gnschenker/ch11-web:1.0
       depends_on:
         - db
   volumes:
     pg-data:
   ```
   *Figura 11.28 – compose.base.yml*

3. Crear `compose.ci.yml` para el entorno de integración continua:
   ```yaml
   services:
     db:
       environment:
         POSTGRES_USER: testuser
         POSTGRES_PASSWORD: testpassword
         POSTGRES_DB: testdb
     web:
       ports:
         - "3000:3000"
       environment:
         DB_HOST: db
         DB_NAME: testdb
         DB_USER: testuser
         DB_PASSWORD: testpassword
   ```
   *Figura 11.29 – compose.ci.yml*

4. Ejecutar combinando los dos archivos:
   ```bash
   $ docker compose -f compose.base.yml -f compose.ci.yml up -d --build
   ```

---

### Modularización de aplicaciones con `include` (Paso 6)

A diferencia de los *overrides*, la directiva `include` (disponible desde Docker Compose v2.20) importa archivos Compose completos respetando las rutas relativas de cada módulo y fallando explícitamente si existen colisiones de nombres.

1. Preparar `step-6`:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition/chapter-11
   $ mkdir step-6 && cd step-6
   $ cp -r ../step-2 .
   ```

2. Crear `db/compose.yml`:
   ```yaml
   services:
     db:
       image: postgres:alpine
       environment:
         POSTGRES_USER: ${POSTGRES_USER:-postgres}
         POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
         POSTGRES_DB: ${POSTGRES_DB:-pets}
       volumes:
         - pg-data:/var/lib/postgresql/data
         - ./db:/docker-entrypoint-initdb.d
   volumes:
     pg-data:
   ```
   *Figura 11.30 – db/compose.yml*

3. Crear `web/compose.yml`:
   ```yaml
   services:
     web:
       build: .
       image: gnschenker/ch11-web:1.0
       ports:
         - "3000:3000"
       environment:
         DB_HOST: db
         DB_NAME: ${POSTGRES_DB:-pets}
         DB_USER: ${POSTGRES_USER:-postgres}
         DB_PASSWORD: ${POSTGRES_PASSWORD:-password}
   ```
   *Figura 11.31 – web/compose.yml*

4. Crear el `compose.yml` raíz:
   ```yaml
   include:
     - ./db/compose.yml
     - ./web/compose.yml
   ```
   *Figura 11.32 – compose.yml raíz con include*

5. Sintaxis extendida para controlar rutas y archivos `.env`:
   ```yaml
   include:
     - path: ./db/compose.yaml
       project_directory: ./db
       env_file:
         - ./db/.env
     - path: ./web/compose.yaml
       project_directory: ./web
       env_file:
         - ./web/.env
   ```

6. Ejecución:
   ```bash
   $ docker compose up -d --build
   $ docker compose ps
   $ docker compose logs -f web
   ```

---

### Cuándo usar Docker Compose frente a un orquestador completo

| Aspecto | Docker Compose | Motor de Orquestación (Kubernetes / Swarm) |
| :--- | :--- | :--- |
| **Alcance** | Host único | Clúster multi-nodo |
| **Caso de uso típico** | Desarrollo local, CI/CD, pruebas de QA y demos | Producción a gran escala, alta disponibilidad |
| **Escalabilidad** | Manual, restringida a los recursos de un host | Declarativa, automatizada a través de múltiples nodos |
| **Gestión de estado** | Inicia servicios; sin autorreparación automática | Mantiene el estado deseado; reprograma contenedores en fallos |
| **Redes** | Redes bridge simples en un único host | Redes overlay avanzadas entre nodos, mallas de servicios |
| **Actualizaciones** | Detención y reinicio manual | Actualizaciones progresivas (*rolling updates*), cero *downtime* |
| **Complejidad** | Muy fácil de aprender y operar | Curva de aprendizaje más pronunciada, mayor potencia |

*Tabla 11.1: Comparativa entre Docker Compose y motores de orquestación*

---

### Resumen

En este capítulo aprendimos:
- Las ventajas del modelo declarativo frente al imperativo en la orquestación de contenedores.
- La evolución de Docker Compose a la versión V2 integrada de forma nativa en el CLI de Docker.
- La definición de servicios, volúmenes, redes y variables de entorno en archivos `compose.yml`.
- La construcción automática de imágenes locales mediante la directiva `build` con BuildKit.
- Las estrategias de escalado horizontal y resolución de colisiones de puertos con puertos dinámicos.
- La personalización para múltiples entornos utilizando archivos de sobreescritura (*overrides*).
- La modularización limpia y sin conflictos de proyectos grandes mediante la directiva `include`.
- Los criterios técnicos para determinar el límite entre Docker Compose y orquestadores como Kubernetes.

---

### Lecturas adicionales

- **Sitio oficial de YAML**: [http://www.yaml.org/](http://www.yaml.org/)
- **Documentación oficial de Docker Compose**: [http://dockr.ly/1FL2VQ6](http://dockr.ly/1FL2VQ6)
- **Referencia de archivos Compose V2**: [https://docs.docker.com/compose/compose-file/compose-file-v2/](https://docs.docker.com/compose/compose-file/compose-file-v2/)
- **Referencia de archivos Compose V3**: [https://docs.docker.com/compose/compose-file/compose-file-v3/](https://docs.docker.com/compose/compose-file/compose-file-v3/)
- **Compartir configuraciones de Compose**: [https://docs.docker.com/compose/extends/](https://docs.docker.com/compose/extends/)

---

### Preguntas

1. **¿Cuál es la principal diferencia entre un enfoque imperativo y uno declarativo al ejecutar contenedores?**
2. **¿Por qué `docker compose` es ahora la forma preferida en lugar del comando `docker-compose`?**
3. **¿Por qué el campo `version:` ya no es obligatorio en los archivos de Compose?**
4. **En un archivo Compose, ¿cuál es la diferencia entre un servicio y un contenedor?**
5. **¿Cómo inicias todos los servicios definidos en un archivo `compose.yml`?**
6. **¿Cómo ejecutas los servicios en segundo plano para no bloquear la terminal?**
7. **Si deseas escalar el servicio `web` a tres instancias, ¿qué comando utilizas?**
8. **¿Qué sucede si intentas escalar un servicio que tiene un mapeo de puerto estático al host (por ejemplo, `3000:3000`)?**
9. **¿Cuál es el propósito de los *overrides* en Docker Compose?**
10. **¿Qué función cumple la nueva palabra clave `include` en Docker Compose?**
11. **Crea un archivo `compose.yml` que defina un servicio `db` (PostgreSQL 16) y un servicio `pgadmin` con comprobación de salud (*healthcheck*).**
12. **Configura el servicio web para escalar a 3 réplicas con puertos dinámicos y verifica su funcionamiento.**
13. **Divide un proyecto Compose en submódulos (`db/compose.yml` y `web/compose.yml`) e intégralos en el archivo raíz mediante `include`.**

---

### Respuestas

1. **Imperativo frente a Declarativo**:  
   En el enfoque imperativo se indican paso a paso las instrucciones exactas que el sistema debe ejecutar. En el enfoque declarativo se describe el estado final deseado y el motor de Compose calcula automáticamente los pasos necesarios para alcanzarlo.

2. **Uso de `docker compose`**:  
   Porque la versión V1 (`docker-compose` en Python) quedó obsoleta (*deprecated*). La versión V2 está completamente integrada en Go dentro del CLI nativo de Docker.

3. **Eliminación del campo `version:`**:  
   Docker Compose ahora se rige por la especificación abierta *Compose Specification*, haciendo innecesaria la declaración de versión.

4. **Servicio frente a Contenedor**:  
   Un servicio es la definición abstracta de configuración que indica cómo debe ejecutarse una pieza de software. Un contenedor es la instancia de proceso real creada en tiempo de ejecución a partir de dicha definición.

5. **Iniciar servicios**:  
   ```bash
   $ docker compose up
   ```

6. **Ejecutar en segundo plano**:  
   ```bash
   $ docker compose up -d
   ```

7. **Comando de escalado**:  
   ```bash
   $ docker compose up --scale web=3
   ```

8. **Fallo por puerto estático**:  
   Solo el primer contenedor se iniciará con éxito; las réplicas restantes fallarán al intentar enlazar el mismo puerto TCP del host que ya se encuentra ocupado. Se debe especificar únicamente el puerto del contenedor (e.g. `- 3000`).

9. **Propósito de los Overrides**:  
   Permiten superponer configuraciones específicas de entorno (CI, QA, Producción) sobre un archivo base común sin duplicar código.

10. **Propósito de `include`**:  
    Permite importar archivos Compose completos manteniendo sus contextos y rutas relativas aisladas, evitando fusiones silenciosas y emitiendo errores ante colisiones de nombres.

11. **Solución ejercicio `compose.yml` con healthcheck**:  
    ```yaml
    services:
      db:
        image: postgres:16-alpine
        environment:
          POSTGRES_USER: dockeruser
          POSTGRES_PASSWORD: dockerpass
          POSTGRES_DB: pets
        volumes:
          - pg-data:/var/lib/postgresql/data
        healthcheck:
          test: ["CMD", "pg_isready", "-U", "dockeruser"]
          interval: 10s
          timeout: 5s
          retries: 5

      pgadmin:
        image: dpage/pgadmin4
        ports:
          - "5050:80"
        environment:
          PGADMIN_DEFAULT_EMAIL: admin@acme.com
          PGADMIN_DEFAULT_PASSWORD: admin
        volumes:
          - pgadmin-data:/var/lib/pgadmin
        depends_on:
          db:
            condition: service_healthy

    volumes:
      pg-data:
      pgadmin-data:
    ```

12. **Solución escalado con puertos dinámicos**:  
    ```bash
    # Compilar, iniciar y escalar
    docker compose up -d --build
    docker compose up -d --scale web=3
    docker compose ps
    ```
    Verificación mediante `curl`:
    ```bash
    curl -s http://localhost:50162/
    curl -s http://localhost:50165/
    curl -s http://localhost:50166/
    ```
    Limpieza:
    ```bash
    docker compose down -v
    ```

13. **Solución estructura modular con `include`**:  
    - `db/compose.yml`:
      ```yaml
      services:
        postgres:
          image: postgres:16-alpine
          environment:
            POSTGRES_USER: dockeruser
            POSTGRES_PASSWORD: pwd
            POSTGRES_DB: pets
          volumes:
            - pg-data:/var/lib/postgresql/data
          healthcheck:
            test: ["CMD", "pg_isready", "-U", "pwd"]
            interval: 10s
            timeout: 5s
            retries: 5

      volumes:
        pg-data:
      ```

    - `web/compose.yml`:
      ```yaml
      services:
        catalog:
          build: .
          environment:
            DB_HOST: postgres
            DB_NAME: pets
            DB_USER: dockeruser
            DB_PASSWORD: dockerpass
          ports:
            - 3000:3000
          depends_on:
            postgres:
              condition: service_healthy
      ```

    - `compose.yml` raíz:
      ```yaml
      include:
        - ./db/compose.yml
        - ./web/compose.yml
      ```

    - Comandos de prueba:
      ```bash
      $ docker compose up -d --build
      $ docker compose ps
      $ curl -s http://localhost:3000/
      $ docker compose down -v
      ```

