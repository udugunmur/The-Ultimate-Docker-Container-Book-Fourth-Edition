# Parte 2: Fundamentos de la Contenerización
## Capítulo 7: Pruebas de aplicaciones en ejecución dentro de contenedores

En los capítulos anteriores, aprendimos a contenerizar nuestras aplicaciones desarrolladas en diversos lenguajes como Node.js, Python, Java, C# y .NET. Todos sabemos que limitarse a escribir código y enviarlo a producción no es suficiente: debemos garantizar que el código esté libre de errores y cumpla con sus especificaciones de diseño. Esto es lo que comúnmente denominamos **aseguramiento de la calidad (*Quality Assurance* o QA)**.

Corregir un defecto o error descubierto en producción resulta exponencialmente más costoso que solucionarlo durante la etapa de desarrollo. La forma más rentable y eficaz de evitarlo es que el propio desarrollador cree **pruebas automatizadas** que aseguren la alta calidad del código y verifiquen el cumplimiento de los criterios de aceptación.

---

### Temas tratados en este capítulo:
- Beneficios de las pruebas en contenedores
- Tipos de pruebas para aplicaciones contenerizadas
- Herramientas, frameworks y entornos de prueba
- Depuración, resolución de problemas y puertas de calidad en CI/CD (*Quality Gates*)
- Desafíos, consideraciones y casos de estudio reales

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Explicar las ventajas y el valor empresarial de ejecutar pruebas de software dentro de contenedores.
- Configurar un entorno productivo para crear y ejecutar pruebas para aplicaciones y servicios contenerizados.
- Desarrollar pruebas unitarias y de integración para código que se ejecuta dentro de un contenedor.
- Ejecutar pruebas unitarias y de integración dentro del mismo contenedor que aloja la aplicación bajo prueba.
- Ejecutar un contenedor de pruebas dedicado para pruebas funcionales de caja negra (*black-box tests*) sobre la aplicación.
- Gestionar dependencias de la aplicación y generar datos de prueba aislados.

---

### Requisitos técnicos

Para seguir los ejercicios prácticos de este capítulo, necesitarás Docker Desktop, una terminal y un editor de código como VS Code.

Prepara tu entorno creando el directorio correspondiente:

```bash
$ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
$ mkdir chapter-07 && cd chapter-07
```

Las soluciones completas para los ejercicios de este capítulo están disponibles en la carpeta `chapter-07/solutions`.

---

### Beneficios de las pruebas en contenedores

#### ¿Por qué realizamos pruebas?
El software moderno requiere entregar nuevas funcionalidades a un ritmo acelerado. Bajo esta presión constante, los analistas y desarrolladores cometen errores humanos. Si estos errores llegan a producción, los usuarios finales los experimentarán directamente, acarreando graves consecuencias operativas y comerciales.

#### Pruebas manuales frente a pruebas automatizadas
Tradicionalmente, un equipo de testers manuales ejecuta suites de pruebas de regresión en un entorno de **pruebas de aceptación de usuario (*User Acceptance Testing* o UAT)**. En aplicaciones empresariales complejas con cientos de casos de prueba, una ronda completa de pruebas manuales de regresión puede tardar varias semanas.

Durante ese tiempo:
- El entorno UAT queda bloqueado para nuevas versiones.
- El código nuevo se acumula en el backlog de desarrollo.
- Se incrementa significativamente el riesgo de los despliegues debido al tamaño masivo de los cambios acumulados (*large batch sizes*).

**Las pruebas manuales de regresión no son escalables, resultan monótonas y son propensas a errores humanos**. La solución consiste en automatizar el 100% de las pruebas de regresión y aceptación, reduciendo el ciclo de retroalimentación de semanas a minutos. Los evaluadores humanos deben dedicar su creatividad y experiencia exclusivamente a **pruebas exploratorias (*exploratory testing*)**.

#### ¿Por qué ejecutar pruebas dentro de contenedores?
1. **Aislamiento (*Isolation*)**: Aislamiento completo entre el entorno de prueba y el sistema host, garantizando resultados consistentes y repetibles.
2. **Consistencia de entorno (*Environment consistency*)**: El entorno de prueba completo (código, dependencias, librerías del sistema y configuración) viaja empaquetado en una unidad autocontenida.
3. **Facilidad de uso**: Elimina la necesidad de instalar dependencias o motores de bases de datos locales en la máquina del desarrollador.
4. **Portabilidad**: Las mismas pruebas se ejecutan idénticamente en la máquina local de desarrollo, en los agentes de CI/CD y en entornos de ensayo (*staging*).
5. **Escalabilidad horizontal**: Permite paralelizar la ejecución de suites de pruebas en múltiples contenedores simultáneos.

---

### Tipos de pruebas para aplicaciones contenerizadas

1. **Pruebas unitarias (*Unit tests*)**:
   - Prueban funciones, métodos o clases aisladas a nivel granular (caja blanca).
   - Rápidas y sin dependencias externas (bases de datos, red, APIs).
   - Se ejecutan frecuentemente en el flujo local y en cada commit de CI.

2. **Pruebas de integración (*Integration tests*)**:
   - Verifican la interacción entre múltiples componentes, servicios o dependencias externas (bases de datos, brokers de mensajería, APIs).
   - Más lentas y complejas de configurar que las pruebas unitarias.

3. **Pruebas de aceptación y caja negra (*Acceptance / Black-box tests*)**:
   - Evalúan el sistema desde la perspectiva del usuario final o de negocio según los criterios de aceptación.
   - El sistema bajo prueba (**System Under Test** o **SUT**) se trata como una caja negra: el código de prueba solo interactúa mediante sus interfaces públicas (APIs REST, colas de mensajería).

*Figura 7.1: Prueba de aceptación interactuando con el sistema bajo prueba*

Las pruebas suelen estructurarse bajo el patrón **Arrange-Act-Assert (AAA)** o **Given-When-Then (GWT)**:
- **Arrange (Given)**: Configuración del estado inicial y datos de prueba.
- **Act (When)**: Ejecución de la acción sobre la interfaz pública del SUT.
- **Assert (Then)**: Verificación de que el resultado obtenido coincide con el esperado.

---

### Implementación del componente de ejemplo (Java Spring Boot)

Implementaremos una API REST para gestionar especies animales y razas asociadas, utilizando Java 21, Spring Boot y la base de datos en memoria H2.

#### Estructura del proyecto:
1. Generar el proyecto desde [https://start.spring.io](https://start.spring.io/) con Maven, Java 21, Spring Boot y las dependencias: **Spring Web**, **Spring Data JPA**, **H2 Database** y **Lombok** (*Figura 7.2*).
2. Descomprimir en `chapter-07/library` y abrir en VS Code.
3. Crear las carpetas `controllers`, `models` y `repositories` dentro de `src/main/java/com/example/library/` (*Figura 7.3*).

#### Modelos de datos:
- `src/main/java/com/example/library/models/Race.java`:
  ```java
  package com.example.library.models;

  import jakarta.persistence.Entity;
  import jakarta.persistence.Id;

  @Entity
  public class Race {
      @Id
      private int id;
      private String name;
      private int speciesId;

      public Race() {}

      public Race(int id, String name, int speciesId) {
          this.id = id;
          this.name = name;
          this.speciesId = speciesId;
      }

      public int getId() { return id; }
      public void setId(int id) { this.id = id; }
      public String getName() { return name; }
      public void setName(String name) { this.name = name; }
      public int getSpeciesId() { return speciesId; }
      public void setSpeciesId(int speciesId) { this.speciesId = speciesId; }
  }
  ```
  *Figura 7.4: Clase de datos Race*

- `src/main/java/com/example/library/models/Species.java`:
  ```java
  package com.example.library.models;

  import jakarta.persistence.Entity;
  import jakarta.persistence.Id;

  @Entity
  public class Species {
      @Id
      private int id;
      private String name;
      private String description;

      public Species() {}

      public Species(int id, String name, String description) {
          this.id = id;
          this.name = name;
          this.description = description;
      }

      public int getId() { return id; }
      public void setId(int id) { this.id = id; }
      public String getName() { return name; }
      public void setName(String name) { this.name = name; }
      public String getDescription() { return description; }
      public void setDescription(String description) { this.description = description; }
  }
  ```
  *Figura 7.5: Clase de datos Species*

#### Repositorios:
- `src/main/java/com/example/library/repositories/RaceRepository.java`:
  ```java
  package com.example.library.repositories;

  import com.example.library.models.Race;
  import org.springframework.data.jpa.repository.JpaRepository;
  import java.util.List;

  public interface RaceRepository extends JpaRepository<Race, Integer> {
      List<Race> findBySpeciesId(int speciesId);
  }
  ```
  *Figura 7.6: Repositorio RaceRepository*

- `src/main/java/com/example/library/repositories/SpeciesRepository.java`:
  ```java
  package com.example.library.repositories;

  import com.example.library.models.Species;
  import org.springframework.data.jpa.repository.JpaRepository;

  public interface SpeciesRepository extends JpaRepository<Species, Integer> {
  }
  ```
  *Figura 7.7: Repositorio SpeciesRepository*

#### Controladores REST:
- `RacesController.java` y `SpeciesController.java` gestionan las operaciones CRUD sobre `/races` y `/species` (*Figura 7.8* y *Figura 7.9*).

#### Configuración de la aplicación (`src/main/resources/application.properties`):
```properties
spring.datasource.url=jdbc:h2:mem:inventory
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.h2.console.enabled=true
```
*Figura 7.10: Configuración de la base de datos H2*

#### Ejecución y pruebas locales:
Inicia la aplicación ejecutando `LibraryApplication.java` (*Figura 7.11*).
Prueba con `curl`:
```bash
$ curl -X POST -d '{"id": 1, "name": "Elephant"}' \
  -H 'Content-Type: application/json' \
  localhost:8080/species
```
Respuesta:
```json
{"id":1,"name":"Elephant","description":null}
```

Listar especies:
```bash
$ curl localhost:8080/species
```
Respuesta:
```json
[{"id":1,"name":"Elephant","description":null}]
```

#### Contenerización de la aplicación Library:
Crear `Dockerfile` en `chapter-07/library/Dockerfile`:
```dockerfile
FROM gradle:8.10-jdk21-alpine AS build
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts ./
RUN gradle build -x test --continue || true
COPY src ./src
RUN gradle bootJar -x test

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```
*Figura 7.12: Dockerfile optimizado para el componente library*

Construcción y ejecución:
```bash
$ docker image build -t library library
$ docker container run -d --rm -p 8080:8080 library
```

---

### Implementación y ejecución de pruebas unitarias y de integración

#### Pruebas unitarias (`src/test/java/com/example/library/LibraryUnitTests.java`):
```java
package com.example.library;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertEquals;

public class LibraryUnitTests {
    private class Calculator {
        public int add(int a, int b) {
            return a + b;
        }
    }

    @Test
    void testAddition() {
        // Arrange
        Calculator calculator = new Calculator();

        // Act
        int result = calculator.add(2, 3);

        // Assert
        assertEquals(5, result);
    }
}
```
*Figura 7.13: Prueba unitaria con patrón AAA*  
*Figura 7.14: Resultados de la ejecución de la prueba*

Ejecución local:
```bash
$ ./gradlew test
```

#### Pruebas de integración (`src/test/java/com/example/library/LibraryIntegrationTests.java`):
```java
package com.example.library;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
public class LibraryIntegrationTests {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void shouldReturnDefaultMessage() throws Exception {
        this.mockMvc.perform(get("/species"))
                .andExpect(status().isOk());
    }
}
```
*Figura 7.15: Prueba de integración con MockMvc*

#### Configuración de reporte de pruebas en `build.gradle.kts`:
```kotlin
tasks.withType<Test> {
    useJUnitPlatform()
    testLogging {
        events("passed", "skipped", "failed")
    }
}
```
*Figura 7.16: Configuración del logging de pruebas*

#### Ejecución de pruebas unitarias e integración dentro del contenedor:
Construimos la imagen incluyendo el código de pruebas y ejecutamos sobrescribiendo el comando de inicio:
```bash
$ docker image build -t library .
$ docker container run --rm library gradle test
```
*Figura 7.17: Salida de la ejecución de pruebas dentro del contenedor*

---

### Implementación y ejecución de pruebas de caja negra (*Black-Box Tests*)

Para desacoplar totalmente las pruebas del sistema bajo prueba, implementaremos las pruebas de caja negra en un proyecto independiente utilizando **.NET 9 y C#**.

1. Crear el proyecto xUnit:
   ```bash
   $ dotnet new xunit -o library-component-tests
   $ dotnet test library-component-tests
   $ code library-component-tests
   ```

2. Definir el modelo y las pruebas en `UnitTest1.cs`:
   ```csharp
   using System.Text.Json;

   public record Species(int id, string name, string description);
   ```

3. Prueba para crear una especie:
   ```csharp
   [Fact]
   public async Task can_add_species()
   {
       // Arrange
       using var client = new HttpClient();
       var species = new Species(1, "Tiger", "Big cat");
       var content = new StringContent(
           JsonSerializer.Serialize(species),
           System.Text.Encoding.UTF8,
           "application/json"
       );

       // Act
       var response = await client.PostAsync("http://localhost:8080/species", content);

       // Assert
       response.EnsureSuccessStatusCode();
       var responseString = await response.Content.ReadAsStringAsync();
       var returnedSpecies = JsonSerializer.Deserialize<Species>(responseString, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
       Assert.Equal("Tiger", returnedSpecies?.name);
   }
   ```
   *Figura 7.18: Prueba de componente para agregar especie*

4. Prueba para consultar una especie por ID:
   ```csharp
   [Fact]
   public async Task can_get_a_species_by_id()
   {
       // Arrange
       using var client = new HttpClient();

       // Act
       var response = await client.GetAsync("http://localhost:8080/species/1");

       // Assert
       response.EnsureSuccessStatusCode();
       var responseString = await response.Content.ReadAsStringAsync();
       var returnedSpecies = JsonSerializer.Deserialize<Species>(responseString, new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
       Assert.Equal(1, returnedSpecies?.id);
   }
   ```
   *Figura 7.19: Prueba de componente para leer especie por ID*

5. Ejecutar las pruebas localmente contra el contenedor de la aplicación:
   ```bash
   $ docker container run --rm -p 8080:8080 library
   $ dotnet test
   ```
   *Figura 7.20: Ejecución de las pruebas de caja negra*

#### Contenerización de las pruebas de caja negra:
Crear `library-component-tests/Dockerfile`:
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0
WORKDIR /app
COPY . .
RUN dotnet restore
CMD ["dotnet", "test"]
```
*Figura 7.21: Dockerfile para las pruebas de componente*

Construir la imagen de pruebas:
```bash
$ docker image build -t library-component-tests .
```

#### Ejecución de la aplicación y las pruebas en contenedores separados:
1. Iniciar el contenedor de la aplicación:
   ```bash
   $ docker container run -d --rm \
     --network host \
     -p 8080:8080 library
   ```
2. Ejecutar el contenedor de pruebas de caja negra:
   ```bash
   $ docker container run library-component-tests
   ```
   *Figura 7.22: Resultados de la ejecución de pruebas de caja negra en contenedor*

---

### Buenas prácticas para configurar un entorno de pruebas

1. **Separación de entornos**: Ejecutar las pruebas en un entorno aislado e independiente del entorno de producción.
2. **Aislamiento de red**: Utilizar redes virtuales dedicadas o redes overlay en Docker para evitar interferencias de tráfico.
3. **Gestión rigurosa de datos de prueba**: Utilizar herramientas de generación de datos sintéticos, snapshots de bases de datos o bases de datos efímeras en memoria.
4. **Restricción de recursos (*Resource constraints*)**: Definir límites de CPU y memoria (`--memory`, `--cpus`) para prevenir que las pruebas colapsen el host.
5. **Orquestación de pruebas**: Utilizar Docker Compose o Testcontainers para coordinar el arranque y apagado ordenado de dependencias (bases de datos, colas, servicios).
6. **Monitorización continua**: Analizar el consumo de recursos de los contenedores durante las suites de pruebas intensivas.

---

### Consejos para depuración y resolución de problemas

- **Inspección de logs**: Consultar `docker container logs <container-name>` para examinar trazas y errores de arranque.
- **Acceso interactivo**: Ejecutar `docker container exec -it <container-name> /bin/sh` para inspeccionar el sistema de archivos y variables del contenedor activo.
- **Depurador remoto acoplado**: Conectar depuradores de IDEs como VS Code o IntelliJ a puertos de depuración expuestos en el contenedor.
- **Contenedores de depuración dedicados**: Replicar entornos problemáticos usando imágenes auxiliares con utilidades de red y diagnóstico (`curl`, `netcat`, `tcpdump`).

---

### Desafíos y consideraciones al probar en contenedores

- **Aislamiento vs Diagnóstico**: El aislamiento dificulta en ocasiones el acceso a métricas directas del host o depuración profunda.
- **Consistencia entre hosts**: Diferencias entre kernels o versiones de Docker Engine en distintos entornos de CI/CD.
- **Gestión del estado de datos**: Garantizar la limpieza de bases de datos entre ejecuciones concurrentes de pruebas.
- **Consumo de recursos**: La ejecución de múltiples suites en paralelo puede saturar memoria y CPU si no se gestionan límites adecuados.
- **Sincronización en pruebas de integración**: Esperar a que los servicios dependientes (bases de datos, message brokers) alcancen el estado *ready* antes de iniciar los tests.

---

### Casos de estudio

1. **Plataforma de comercio electrónico**: Automatizó y paralelizó la ejecución de pruebas funcionales y de regresión en contenedores por cada rama de funcionalidad (*feature branch*), reduciendo el tiempo de ejecución de días a menos de una hora y facilitando despliegues continuos diarios.
2. **Entidad de servicios financieros**: Implementó suites automatizadas en contenedores para su plataforma de trading, detectando anomalías de integración tempranamente y minimizando el riesgo de caídas en producción.
3. **Organización sanitaria**: Empleó pruebas en contenedores para validar sistemas de historiales clínicos electrónicos (EMR), garantizando el cumplimiento normativo estricto y la fiabilidad de los datos médicos de los pacientes.

---

### Resumen

En este capítulo analizamos:
- Las ventajas del aseguramiento de la calidad mediante pruebas automatizadas frente a los cuellos de botella del testing manual.
- Las diferencias entre pruebas unitarias, de integración y de caja negra (*black-box*).
- El desarrollo de una API REST en Spring Boot con persistencia H2 y su empaquetado en Docker.
- La ejecución de pruebas unitarias y de integración dentro del contenedor de la aplicación.
- La creación de pruebas de aceptación de caja negra independientes en .NET 9 ejecutadas en un contenedor separado.
- Las mejores prácticas, técnicas de resolución de incidencias y lecciones aprendidas de casos de estudio reales.

---

### Preguntas

1. **¿Cuáles son los principales beneficios de ejecutar pruebas en contenedores?**
2. **Nombra y describe brevemente las tres capas de pruebas en un flujo de trabajo contenerizado.**
3. **¿Qué comandos y herramientas de Docker te ayudan a depurar una prueba fallida dentro de un contenedor?**
4. **¿Cómo ejecutamos pruebas unitarias para una aplicación dentro de un contenedor?**
5. **¿Las imágenes de Docker destinadas a producción deben contener código de pruebas? Justifica tu respuesta.**
6. **¿Dónde se ejecutan habitualmente las pruebas unitarias y de integración que se ejecutan dentro de contenedores?**
7. **Enumera algunas ventajas de ejecutar pruebas unitarias y de integración en contenedores.**
8. **¿Cuáles son algunos de los desafíos que puedes enfrentar al ejecutar pruebas en contenedores?**
9. **Menciona un ejemplo de cómo las pruebas en contenedores aceleraron los ciclos de lanzamiento en un caso de estudio real.**

---

### Respuestas

1. **Beneficios principales**:  
   Garantiza el aislamiento y la repetibilidad del entorno en cualquier máquina, acelera los ciclos de retroalimentación empaquetando dependencias junto con el código y permite escalar horizontalmente los ejecutores de pruebas en pipelines de CI/CD.

2. **Tres capas de pruebas**:  
   - **Pruebas unitarias**: Pruebas rápidas de caja blanca que validan funciones y clases individuales de forma aislada.
   - **Pruebas de integración**: Pruebas de caja gris que coordinan múltiples componentes o contenedores (servicio y base de datos) para verificar sus interfaces.
   - **Pruebas de caja negra / aceptación**: Pruebas de extremo a extremo ejecutadas desde un contenedor independiente que interactúan exclusivamente con la superficie de la API pública del servicio.

3. **Herramientas de depuración**:  
   - `docker logs <container-name>` para examinar registros.
   - `docker exec -it <container-name> /bin/sh` para inspeccionar el contenedor en caliente.
   - Conexión de un depurador remoto desde el IDE (VS Code / IntelliJ) a un puerto de depuración publicado.

4. **Ejecución de pruebas unitarias en contenedor**:  
   Sobrescribiendo el comando por defecto de la imagen (`CMD`) al iniciar el contenedor. Por ejemplo, en Java sustituyendo `CMD java -jar /app/my-app.jar` por `gradle test` o `mvn test`, y en Node.js sustituyendo `CMD node index.js` por `npm test`.

5. **Exclusión de pruebas en imágenes de producción**:  
   **No deben contener código de pruebas**. Incluir tests incrementa el tamaño de la imagen (mayor tiempo de descarga e inicio) y amplía la superficie de ataque al incorporar librerías y utilidades de testing innecesarias en producción.

6. **Ubicación habitual de ejecución**:  
   En la máquina local del desarrollador antes de realizar el push y en los agentes de construcción (*build agents*) durante la etapa de Integración Continua (CI) del pipeline de CI/CD.

7. **Ventajas clave**:  
   - Aislamiento total sin necesidad de instalar SDKs o librerías en el host.
   - Máxima reproducibilidad y consistencia entre entornos locales y servidores de CI.

8. **Desafíos habituales**:  
   - Mayor complejidad para depurar fallos cuando no se tiene acceso directo al host.
   - Dificultad para coordinar y sincronizar el arranque de múltiples contenedores dependientes.
   - Posibles restricciones de recursos de hardware (límites de CPU/memoria por cgroups).

9. **Ejemplo de aceleración en caso real**:  
   Un equipo de comercio electrónico migró su suite completa de regresión a contenedores paralelizados por rama, reduciendo el tiempo de ejecución de varios días a menos de una hora y permitiendo despliegues diarios a producción.
