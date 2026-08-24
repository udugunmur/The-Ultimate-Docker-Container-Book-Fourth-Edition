# Parte 1: Introducción
## Capítulo 2: Configuración de un entorno de trabajo

En el capítulo anterior, aprendimos qué son los contenedores Docker y por qué son importantes. Conocimos qué tipo de problemas resuelven los contenedores en una cadena de suministro de software moderna. En este capítulo, vamos a preparar nuestro entorno personal o de trabajo para operar de manera eficiente y eficaz con Docker. Discutiremos en detalle cómo configurar un entorno ideal para desarrolladores, ingenieros de DevOps y operadores que pueda utilizarse al trabajar con contenedores Docker.

---

### Temas tratados en este capítulo:
- Distinción de los principales sistemas operativos
- La consola de comandos de Linux (*shell*)
- PowerShell para Windows
- Instalación y uso de un gestor de paquetes
- Instalación de Git y clonación del repositorio de código
- Selección e instalación de un editor de código
- Instalación de Docker Desktop en macOS o Windows
- Uso de Docker con WSL 2 en Windows
- Instalación de Docker Toolbox
- Habilitación de Kubernetes en Docker Desktop
- Instalación de Podman
- Instalación de minikube
- Instalación de kind

---

### Objetivos de aprendizaje
Tras completar este capítulo, serás capaz de:
- Configurar un entorno de desarrollo de nivel profesional para el desarrollo de software contenerizado en macOS, Windows o Linux.
- Utilizar un gestor de paquetes, una consola (*shell*) y un editor de código para configurar tu sistema local para trabajar con contenedores.
- Instalar y verificar Docker, Podman y herramientas de Kubernetes como minikube y kind en todas las plataformas compatibles.
- Probar tu configuración de extremo a extremo (*end-to-end*) para asegurarte de que los contenedores y las cargas de trabajo de Kubernetes se puedan construir, ejecutar y orquestar localmente sin problemas.

---

### Requisitos técnicos

Para este capítulo, necesitarás un portátil o una estación de trabajo con **macOS** o **Windows** (preferiblemente Windows 11 Professional) instalado. También debes tener acceso libre a Internet para descargar aplicaciones y permisos de administrador para instalarlas en tu equipo. 

Asimismo, es posible seguir este libro si utilizas una distribución de **Linux** como sistema operativo, como Ubuntu 24.10 o superior. Haré todo lo posible por indicar dónde los comandos y ejemplos difieren significativamente de los de macOS, que será mi plataforma principal a lo largo de este libro.

---

### Distinción de los principales sistemas operativos

Aunque Docker está disponible para las tres plataformas principales (macOS, Windows y Linux), cada entorno tiene sus particularidades. Antes de profundizar en los detalles del capítulo, hagamos un breve resumen de los tres sistemas operativos:

#### macOS
- **Requisitos del sistema**: Los Mac basados en Intel requieren macOS 10.14 o superior, mientras que los chips Apple Silicon (M1/M2/M3) necesitan macOS 11 o posterior. Ten en cuenta que versiones muy antiguas pueden requerir Docker Toolbox.
- **Instalación preferida**: Utiliza la versión dedicada de Docker Desktop para Mac ([https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)). Se integra perfectamente con el hipervisor de macOS (HyperKit en Intel, el framework de hipervisor nativo de Apple en Apple Silicon).
- **Gestor de paquetes**: La instalación de herramientas adicionales (como `git` o `jq`) es típicamente más sencilla a través de **Homebrew**.

#### Windows
- **Requisitos del sistema**: Las ediciones Windows 10 u 11 Professional o Enterprise son compatibles con Docker Desktop mediante WSL 2 o Hyper-V. Las ediciones Home pueden usar WSL 2, aunque pueden requerir configuración adicional.
- **Instalación preferida**: Docker Desktop para Windows utiliza Hyper-V o WSL 2 como virtualización subyacente. Si estás en Windows Home, puedes instalar WSL 2 y ejecutar Docker Desktop con él.
- **Gestor de paquetes**: **Chocolatey** (o el más reciente Administrador de paquetes de Windows, `winget`) simplifica la instalación de herramientas de desarrollo.

#### Linux
- **Requisitos del sistema**: Una distribución de Linux moderna (Ubuntu, Debian, Fedora, CentOS, etc.). El kernel debe soportar *cgroups* y *namespaces*. Para distribuciones más antiguas, consulta la documentación oficial de Docker.
- **Instalación preferida**: Instala Docker Engine directamente desde los repositorios de paquetes de tu distribución o utiliza el repositorio oficial de Docker. Herramientas como minikube pueden requerir un hipervisor específico (KVM, VirtualBox).
- **Gestor de paquetes**: Varía según la distribución (`apt` para Debian/Ubuntu, `dnf` o `yum` para Fedora/CentOS, etc.).

---

### La consola de comandos de Linux (*shell*)

Los contenedores Docker se desarrollaron por primera vez en Linux y para Linux. Por lo tanto, es natural que la herramienta de línea de comandos principal utilizada para trabajar con Docker, también llamada *shell*, sea una shell de Unix (recordemos que Linux deriva de Unix). La mayoría de los desarrolladores utilizan la shell **Bash**. En algunas distribuciones ligeras de Linux, como Alpine, Bash no viene instalado de forma predeterminada y, en consecuencia, se debe utilizar la Bourne shell más simple, llamada simplemente `sh`. Siempre que estemos trabajando en un entorno Linux, como dentro de un contenedor o en una máquina virtual Linux, utilizaremos `/bin/bash` o `/bin/sh`, dependiendo de su disponibilidad.

Aunque macOS de Apple no es un sistema operativo Linux, tanto Linux como macOS son variantes de Unix y, por ende, admiten el mismo conjunto de herramientas, entre ellas las consolas de comandos. Por lo tanto, al trabajar en macOS, probablemente estarás utilizando la shell **Bash** o **Zsh**.

En este libro, esperamos que estés familiarizado con los comandos de scripting más básicos en Bash y en PowerShell (si trabajas en Windows). Si eres un principiante absoluto, te recomendamos encarecidamente que te familiarices con las siguientes hojas de referencia (*cheat sheets*):
- **Linux Command Line Cheat Sheet** por Dave Child: [http://bit.ly/2mTQr8l](http://bit.ly/2mTQr8l)
- **PowerShell Basic Cheat Sheet**: [http://bit.ly/2EPHxze](http://bit.ly/2EPHxze)

---

### PowerShell para Windows

En un ordenador, portátil o servidor Windows, disponemos de múltiples herramientas de línea de comandos. La más conocida es la consola de comandos tradicional (`cmd`), presente en Windows desde hace décadas. Es una shell muy simple. Para scripting más avanzado, Microsoft desarrolló **PowerShell**, que es muy potente y popular entre los ingenieros de Windows. Por último, en Windows 10 o posterior, contamos con el denominado **Windows Subsystem for Linux (WSL)**, que nos permite utilizar cualquier herramienta de Linux, como las shells Bash o Bourne. Aparte de esto, otras herramientas también instalan una shell Bash en Windows, como Git Bash.

En este libro, todos los comandos utilizarán sintaxis Bash. La mayoría de los comandos también se ejecutan sin problemas en PowerShell. Por lo tanto, te recomendamos utilizar PowerShell o cualquier otra herramienta basada en Bash para trabajar con Docker en Windows.

---

### Instalación y uso de un gestor de paquetes

La forma más sencilla de instalar software en un portátil Linux, macOS o Windows es utilizar un buen gestor de paquetes. En macOS, la mayoría de los usuarios utilizan **Homebrew**, mientras que en Windows, el Administrador de paquetes de Windows (`winget`) o **Chocolatey** son excelentes opciones. Si utilizas una distribución de Linux basada en Debian como Ubuntu, el gestor de paquetes preferido es `apt`, que viene instalado por defecto.

#### Instalación de Homebrew en macOS
Homebrew es el gestor de paquetes más popular en macOS, fácil de usar y muy versátil. La instalación es sencilla; sigue las instrucciones en [https://brew.sh/](https://brew.sh/):

1. Abre una nueva ventana de Terminal y ejecuta el siguiente comando para instalar Homebrew:
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```

2. Una vez finalizada la instalación, comprueba si Homebrew funciona ejecutando `brew --version` en la Terminal. Deberías ver algo similar a esto:
   ```bash
   $ brew --version
   Homebrew 4.4.23
   ```

3. Ahora estamos listos para usar Homebrew para instalar herramientas y utilidades. Si, por ejemplo, queremos instalar el editor de texto Vim (ten en cuenta que esta no es una herramienta que usaremos en el libro; sirve solo como ejemplo), podemos hacerlo así:
   ```bash
   $ brew install vim
   ```
   Esto descargará e instalará el editor por ti.

#### Instalación de Chocolatey en Windows
Chocolatey es un gestor de paquetes popular para Windows, construido sobre PowerShell. Para instalar Chocolatey, sigue estas instrucciones:

1. **Abrir PowerShell como administrador**: Presiona `Win + S`, escribe `PowerShell` y selecciona *Ejecutar como administrador*.
2. **Establecer la directiva de ejecución**: En la ventana de PowerShell, verifica la directiva de ejecución actual:
   ```powershell
   Get-ExecutionPolicy
   ```
   Si devuelve `Restricted`, cámbiala a `AllSigned` o `Bypass` para permitir la ejecución del script de instalación:
   ```powershell
   Set-ExecutionPolicy Bypass -Scope Process -Force
   ```
3. **Instalar Chocolatey**: Ejecuta el siguiente comando en la ventana de PowerShell:
   ```powershell
   [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
   ```
4. Tras completar la instalación, verifica que Chocolatey está instalado:
   ```powershell
   choco
   ```
5. Comprueba la versión instalada:
   ```powershell
   choco --version
   ```
   Deberías ver una salida como esta:
   ```text
   2.4.3
   ```
6. Intenta instalar una aplicación con Chocolatey, como Vim:
   ```powershell
   choco install -y vim
   ```
   El parámetro `-y` asegura que la instalación se realice sin solicitar confirmación. Una vez instalado, es posible que debas abrir una nueva ventana de PowerShell para usar la aplicación.

---

### Instalación de Git y clonación del repositorio de código

Utilizaremos Git para clonar el código de ejemplo que acompaña a este libro desde su repositorio de GitHub. Si ya tienes Git instalado en tu ordenador, puedes omitir esta sección:

- Para instalar Git en **macOS**, ejecuta en una ventana de Terminal:
  ```bash
  $ brew install git
  ```
- Para instalar Git en **Windows**, abre PowerShell y usa Chocolatey:
  ```powershell
  PS> choco install git -y
  ```
- En tu máquina **Debian o Ubuntu**, abre una consola Bash y ejecuta:
  ```bash
  $ sudo apt update && sudo apt install -y git
  ```

Una vez instalado Git, verifica su funcionamiento en cualquier plataforma:
```bash
$ git --version
```
Esto mostrará la versión de Git instalada. En el MacBook Pro M2 del autor, la salida es:
```text
git version 2.49.0
```
Si ves una versión más antigua en macOS, probablemente sea la que incluye el sistema por defecto; usa Homebrew para instalar la última versión con `$ brew install git`.

Ahora que Git funciona, clonamos el código fuente del libro desde GitHub:
```bash
$ cd ~
$ git clone https://github.com/PacktPublishing/The-Ultimate-Docker-Container-Book-Fourth-Edition.git
```
Esto clonará el contenido de la rama `main` en tu carpeta local `~/The-Ultimate-Docker-Container-Book-Fourth-Edition`. Esta carpeta contendrá todas las soluciones de ejemplo para los laboratorios que realizaremos en el libro.

---

### Selección e instalación de un editor de código

Contar con un buen editor de código es esencial para trabajar de forma productiva con Docker. La elección del editor depende de tus preferencias personales: Vim, Emacs, Atom, Sublime o Visual Studio Code (VS Code), entre otros. **VS Code** es completamente gratuito, ligero, potente y está disponible para macOS, Windows y Linux. Según Stack Overflow, es actualmente el editor más popular. Si aún no tienes una preferencia clara, te recomiendo que pruebes VS Code.

Si ya tienes un editor favorito, puedes continuar usándolo mientras permita editar archivos de texto y soporte resaltado de sintaxis para Dockerfiles, JSON y YAML. La única excepción será el [Capítulo 6](https://subscription.packtpub.com/book/cloud-and-networking/9781805804390/6), *Depuración de código en ejecución dentro de contenedores*, cuyos ejemplos estarán fuertemente adaptados a VS Code.

#### Instalación de VS Code en macOS
1. Abre una ventana de Terminal y ejecuta:
   ```bash
   $ brew install --cask visual-studio-code
   ```
2. Una vez instalado, navega a tu directorio home:
   ```bash
   cd ~
   ```
3. Abre VS Code desde esta carpeta:
   ```bash
   $ code The-Ultimate-Docker-Container-Book-Fourth-Edition
   ```
   VS Code se iniciará y abrirá la carpeta del repositorio como espacio de trabajo.

#### Instalación de VS Code en Windows
1. Abre una ventana de PowerShell en modo administrador y ejecuta:
   ```powershell
   PS> choco install vscode -y
   ```
2. Cierra la ventana de PowerShell y abre una nueva para asegurarte de que VS Code esté en tu variable `PATH`.
3. Navega a tu directorio home y abre la carpeta:
   ```powershell
   PS> cd ~
   PS> code The-Ultimate-Docker-Container-Book-Fourth-Edition
   ```

#### Instalación de VS Code en Linux (usando Snap)
1. Verifica si Snap está instalado:
   ```bash
   $ snap --version
   ```
   Si no está instalado en sistemas basados en Debian, instálalo con:
   ```bash
   $ sudo apt update
   $ sudo apt install snapd
   ```
2. Instala VS Code mediante Snap con el flag `--classic`:
   ```bash
   $ sudo snap install --classic code
   ```
   *(Para distribuciones no basadas en Debian/Ubuntu, consulta: [https://code.visualstudio.com/docs/setup/linux](https://code.visualstudio.com/docs/setup/linux)).*
3. Navega a tu directorio home y abre la carpeta del proyecto:
   ```bash
   $ cd ~
   $ code The-Ultimate-Docker-Container-Book-Fourth-Edition
   ```

#### Instalación de extensiones de VS Code
Las extensiones hacen de VS Code un editor sumamente versátil. En las tres plataformas, puedes instalarlas desde la consola:

Abre una consola Bash (o PowerShell en Windows) y ejecuta los siguientes comandos:
```bash
code --install-extension vscjava.vscode-java-pack
code --install-extension ms-dotnettools.csharp
code --install-extension ms-python.python
code --install-extension ms-azuretools.vscode-docker
code --install-extension eamodio.gitlens
```

Estas extensiones nos permiten trabajar de forma productiva con Java, C#, .NET, Python y Docker. Tras instalarlas, reinicia VS Code. Puedes verificar las extensiones instaladas ejecutando:
```bash
$ code --list-extensions
```

#### Instalación de cursor.ai
En la actualidad, la IA está transformando el desarrollo de software. **cursor.ai** es un asistente inteligente integrado directamente en una base de Visual Studio Code que proporciona sugerencias de código en tiempo real, autocompletado consciente del contexto y asistencia para depuración.

Pasos para instalar cursor.ai:
1. **Descargar el instalador**: Visita [https://www.cursor.com/](https://www.cursor.com/) y haz clic en *Download*.
2. **Ejecutar el instalador**: Abre el archivo descargado y sigue las instrucciones en pantalla.
3. **Iniciar y configurar**: Inicia Cursor desde tu menú de inicio o carpeta de aplicaciones, inicia sesión y configura tus preferencias.

---

### Instalación de Docker Desktop en macOS, Windows o Linux

Si utilizas macOS o Windows 10/11, te recomendamos encarecidamente instalar **Docker Desktop**. Desde 2022, Docker también ofrece una versión para Linux.

1. **Para Linux**: Dirígete a [https://docs.docker.com/desktop/install/linux/](https://docs.docker.com/desktop/install/linux/) y sigue las instrucciones.
2. **Para Windows y macOS**: Visita la página de inicio de Docker en [https://www.docker.com/get-started](https://www.docker.com/get-started):
   
   *Figura 2.1: Primeros pasos con Docker*

3. En la esquina superior derecha, haz clic en **Sign In** para iniciar sesión o crear una cuenta gratuita en Docker Hub.
4. Haz clic en el botón azul **Download Docker Desktop** y selecciona el instalador correspondiente a tu sistema operativo (Intel o Apple Silicon en Mac, instalador x86_64 en Windows):

   *Figura 2.2: Lista de opciones de descarga de Docker Desktop*

5. Una vez descargado el paquete, ejecuta la instalación haciendo doble clic en el instalador.

#### Comprobación de Docker Engine
Probemos la instalación ejecutando un contenedor simple desde la línea de comandos:

1. Abre una ventana de Terminal y ejecuta:
   ```bash
   $ docker version
   ```
   *Figura 2.3: Versión de Docker en Docker Desktop*

   La salida consta de dos partes: **Client** y **Server** (Docker Engine, responsable de alojar y ejecutar contenedores; versión 27.4.0 al momento de redacción).

2. Para comprobar la ejecución de contenedores, ingresa:
   ```bash
   $ docker container run hello-world
   ```
   *Figura 2.4: Ejecución de hello-world en Docker Desktop para macOS*

   Si Docker no encuentra la imagen `hello-world:latest` localmente, la descargará del registro, creará el contenedor y mostrará el mensaje que comienza con `Hello from Docker!`.

3. Prueba otra imagen divertida de verificación:
   ```bash
   $ docker container run rancher/cowsay hello
   ```
   *Figura 2.5: Ejecución de la imagen cowsay de Rancher*

#### Comprobación de Docker Desktop
El icono de Docker es una pequeña ballena que transporta contenedores:
- **Mac**: En la barra de menús superior, a la derecha.
- **Windows**: En la bandeja del sistema (*system tray*).
- **Linux**: En el menú de aplicaciones, busca y abre Docker Desktop.

Pasos en Docker Desktop:
1. Haz clic en el icono de la ballena para ver el menú contextual:
   
   *Figura 2.6: Menú contextual de Docker Desktop*

2. Selecciona **Dashboard** para abrir el panel de control:
   
   *Figura 2.7: Panel de control (Dashboard) de Docker Desktop*

3. En la pestaña **Containers**, verás los contenedores creados anteriormente a partir de las imágenes `hello-world` y `rancher/cowsay`, ambos con estado *Exited*.

*(Nota: Cerrar la ventana del Dashboard no detiene Docker Desktop; el motor continúa en ejecución en segundo plano. Para detenerlo por completo, selecciona **Quit Docker Desktop** en el menú contextual).*

---

### Uso de Docker con WSL 2 en Windows

En Windows 10 u 11, puedes aprovechar el **Windows Subsystem for Linux versión 2 (WSL 2)** para obtener un rendimiento de contenedores prácticamente nativo de Linux sin necesidad de una máquina virtual pesada.

1. **Habilitar WSL 2**: Instala o actualiza WSL siguiendo la documentación oficial de Microsoft en [https://learn.microsoft.com/en-us/windows/wsl/install](https://learn.microsoft.com/en-us/windows/wsl/install).
2. **Configuración en Docker Desktop**: Abre Docker Desktop, ve a *Settings* > *General*, y confirma que la opción **Use the WSL 2 based engine** esté activada.
3. **Ventajas de WSL 2**:
   - **Rendimiento mejorado**: E/S de archivos más rápida y velocidad nativa de Linux.
   - **Mejor gestión de recursos**: Menor consumo de memoria y CPU en comparación con Hyper-V tradicional.
   - **Integración fluida del sistema de archivos**: Acceso directo entre los archivos locales de Windows y el entorno Linux.

---

### Instalación de Docker Toolbox

Docker Toolbox fue la herramienta previa a Docker Desktop que permitía ejecutar contenedores en Windows y macOS mediante máquinas virtuales gestionadas con VirtualBox. 

**Docker Toolbox ha quedado obsoleto (*deprecated*)** y ya no se encuentra en desarrollo activo, por lo que no se profundizará en él en este libro.

---

### Habilitación de Kubernetes en Docker Desktop

Docker Desktop incluye soporte integrado para Kubernetes (desactivado por defecto).

> **¿Qué es Kubernetes?**  
> Kubernetes es una plataforma potente para automatizar el despliegue, escalado y gestión de aplicaciones contenerizadas en clústeres.

Para habilitarlo en Docker Desktop:
1. Abre el Dashboard de Docker Desktop.
2. Haz clic en el icono del engranaje (*Settings*) en la esquina superior derecha.
3. En la barra lateral izquierda, selecciona la pestaña **Kubernetes** y activa la casilla **Enable Kubernetes**:

   *Figura 2.8: Habilitación de Kubernetes en Docker Desktop*

4. Haz clic en **Apply & restart** y espera a que Docker descargue los componentes necesarios e inicie el clúster.

*(Nota: Docker Desktop proporciona un clúster de un solo nodo. Para configuraciones multinodo locales, se recomienda utilizar kind o minikube).*

---

### Instalación de Podman

**Podman** es un motor de contenedores de código abierto y sin demonio (*daemonless*) que sirve como alternativa a Docker. Es ampliamente compatible con la CLI de Docker y permite ejecutar contenedores en modo *rootless* (sin privilegios de root) de manera predeterminada.

#### Instalación de Podman en macOS
```bash
$ brew install podman
$ podman machine init
$ podman machine start
$ podman --version
$ podman info
```

#### Instalación de Podman en Windows
```powershell
$ choco install podman -y
$ podman machine init
$ podman machine start
$ podman --version
$ podman info
```

#### Instalación de Podman en Linux (Debian/Ubuntu)
```bash
$ sudo apt-get update
$ sudo apt-get install -y podman
$ podman --version
```

#### Comparativa: Pros y contras de Podman frente a Docker Desktop
- **Ventajas de Podman**:
  - **Arquitectura sin demonio (*daemonless*)**: Reduce la superficie de ataque y el consumo de recursos en reposo.
  - **Operación Rootless**: Mayor seguridad al ejecutar contenedores con privilegios de usuario estándar.
  - **Compatibilidad con Docker CLI**: Los comandos habituales funcionan de forma transparente (`alias docker=podman`).
  - **Ligero**: Menor consumo de recursos del sistema.
- **Desventajas de Podman**:
  - **Herramientas gráficas (GUI) limitadas**: Depende fundamentalmente de la CLI.
  - **Soporte en plataformas no Linux**: Requiere la inicialización de máquinas virtuales (`podman machine`) en Windows y macOS.
  - **Ecosistema**: Docker Desktop cuenta con mayores integraciones nativas y extensiones de terceros.

---

### Instalación de minikube

**minikube** aprovisiona un clúster local de Kubernetes (de uno o varios nodos) en tu estación de trabajo y se gestiona mediante `kubectl`.

Consulta los detalles oficiales en: [https://kubernetes.io/docs/tasks/tools/install-minikube/](https://kubernetes.io/docs/tasks/tools/install-minikube/).

*Figura 2.9: Requisitos previos para minikube*  
*Figura 2.10: Selección de la instalación correcta para minikube*

#### Instalación de minikube en macOS con Homebrew
1. Instala minikube:
   ```bash
   $ brew install minikube
   ```
2. Comprueba la versión:
   ```bash
   $ minikube version
   minikube version: v1.35.0 commit: dd5d320e41b5451cdf3c01891bc4e13d189586ed
   ```
3. Inicia un clúster por defecto:
   ```bash
   $ minikube start
   ```
   Salida generada:
   ```text
   minikube v1.35.0 on Darwin 15.3.1 (arm64) ✨ Automatically selected the docker driver Using Docker Desktop driver with root privileges Starting "minikube" primary control-plane node in "minikube" cluster Pulling base image v0.0.46 ... Downloading Kubernetes v1.32.0 preload ... > gcr.io/k8s-minikube/kicbase...: 452.84 MiB / 452.84 MiB 100.00% 21.62 M > preloaded-images-k8s-v18-v1...: 303.97 MiB / 314.92 MiB 96.52% 14.01 Mi
   ```
   Al finalizar, confirmará:
   ```text
   🏄 Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
   ```

#### Pruebas con minikube y kubectl
1. Verifica el contexto actual de `kubectl`:
   ```bash
   $ kubectl config get-contexts
   ```
   *Figura 2.11: Lista de contextos para kubectl tras instalar minikube*

2. Lista los nodos del clúster:
   ```bash
   $ kubectl get nodes
   ```
   *Figura 2.12: Lista de nodos del clúster minikube*

3. Despliega un pod con el servidor web Nginx:
   ```bash
   $ cd ~/The-Ultimate-Docker-Container-Book-Fourth-Edition
   $ kubectl apply -f setup/nginx.yaml
   ```
   Salida:
   ```text
   pod/nginx created
   ```

4. Verifica el estado del pod:
   ```bash
   $ kubectl get pods
   ```
   Salida:
   ```text
   NAME READY STATUS RESTARTS AGE
   nginx 1/1 Running 0 29s
   ```

5. Expón el pod mediante un servicio `NodePort`:
   ```bash
   $ kubectl expose pod nginx --type=NodePort --port=80
   ```
   Salida:
   ```text
   service/nginx exposed
   ```

6. Lista los servicios del clúster:
   ```bash
   $ kubectl get services
   ```
   *Figura 2.13: Lista de servicios en el clúster minikube*

7. Abre un túnel para acceder al servicio Nginx desde el navegador:
   ```bash
   $ minikube service nginx
   ```
   *Figura 2.14: Apertura de acceso al clúster de Kubernetes en minikube*  
   *Figura 2.15: Pantalla de bienvenida de Nginx ejecutándose en minikube*

8. **Limpieza**:
   - Detén el túnel presionando `Ctrl + C`.
   - Elimina el servicio y el pod:
     ```bash
     $ kubectl delete service nginx
     $ kubectl delete pod nginx
     ```
   - Detén el clúster:
     ```bash
     $ minikube stop
     ```
     *Figura 2.16: Detención de minikube*

#### Trabajo con un clúster multinodo en minikube
Para crear un clúster de tres nodos llamado `demo`:
```bash
$ minikube start --nodes 3 -p demo
```
Verifica los nodos creados:
```bash
$ kubectl get nodes
NAME STATUS ROLES AGE VERSION
demo Ready control-plane 84s v1.32.0
demo-m02 Ready <none> 45s v1.32.0
demo-m03 Ready <none> 22s v1.32.0
```
Para detener y eliminar los clústeres:
```bash
$ minikube stop -p demo
$ minikube delete --all
```

---

### Instalación de kind

**kind** (*Kubernetes IN Docker*) es otra herramienta popular que ejecuta clústeres locales de Kubernetes utilizando contenedores Docker como nodos.

- **macOS (Homebrew)**:
  ```bash
  $ brew install kind
  ```
- **Windows (Chocolatey)**:
  ```powershell
  $ choco install kind -y
  ```
- **Linux**:
  ```bash
  $ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
  $ chmod +x ./kind
  $ sudo mv ./kind /usr/local/bin/kind
  ```

Verifica la instalación:
```bash
$ kind version
kind v0.27.0 go1.24.0 darwin/arm64
```

#### Creación y gestión de clústeres con kind
1. Crear un clúster predeterminado:
   ```bash
   $ kind create cluster
   ```
   *Figura 2.17: Creación de un clúster con kind*

2. Listar clústeres existentes:
   ```bash
   $ kind get clusters
   ```
3. Crear un clúster con nombre personalizado:
   ```bash
   $ kind create cluster --name demo
   ```
4. Eliminar todos los clústeres de kind:
   ```bash
   $ kind delete clusters --all
   ```

#### Pruebas comparativas entre kind y minikube
1. Iniciar ambos clústeres:
   ```bash
   $ minikube start -p minikube-demo
   $ kind create cluster --name kind-demo
   ```
2. Inspeccionar contextos de `kubectl`:
   ```bash
   $ kubectl config get-contexts
   ```
   *Figura 2.18: Lista de contextos definidos para kubectl*

3. Cambiar de contexto activo:
   ```bash
   $ kubectl config use-context minikube-demo
   $ kubectl get nodes
   ```
   *Figura 2.19: Lista de nodos en el clúster minikube*

4. Desplegar Nginx y realizar reenvío de puertos (*port-forward*):
   ```bash
   $ kubectl apply -f setup/nginx.yaml
   $ kubectl port-forward nginx 8080:80
   ```
   Salida:
   ```text
   Forwarding from 127.0.0.1:8080 -> 80
   Forwarding from [::1]:8080 -> 80
   ```
   Accede a `http://localhost:8080` en tu navegador para ver la pantalla de Nginx (*Figura 2.20*).

5. Limpieza final:
   ```bash
   $ kubectl delete -f setup/nginx.yaml
   $ minikube delete --all
   $ kind delete cluster --name kind-demo
   ```

---

### Resumen

En este capítulo, nos enfocamos en establecer y configurar un entorno de trabajo robusto adaptado para gestionar eficazmente contenedores Docker.

- Comenzamos destacando el valor de los **gestores de paquetes** (`Homebrew`, `Chocolatey`, `apt`) para automatizar la instalación de utilidades.
- Subrayamos la importancia de consolas como **Bash** y **PowerShell**, y configuramos **Visual Studio Code** con sus extensiones esenciales y herramientas de IA como **cursor.ai**.
- Instalamos y verificamos los motores de contenedores principales: **Docker Desktop** (con soporte WSL 2 en Windows) y **Podman** (con su arquitectura *rootless* y *daemonless*).
- Por último, configuramos entornos locales de orquestación con **Kubernetes** a través de **Docker Desktop**, **minikube** y **kind**, desplegando y exponiendo aplicaciones de prueba como Nginx en clústeres de uno y varios nodos.

En el próximo capítulo, comenzaremos a explorar los comandos fundamentales de Docker para ejecutar, detener, listar, inspeccionar y eliminar contenedores, adentrándonos en su anatomía interna.

---

### Lecturas adicionales

- **Chocolatey – El gestor de paquetes para Windows**: [https://chocolatey.org/](https://chocolatey.org/)
- **Instalación de Docker Toolbox en Windows**: [https://dockr.ly/2nuZUkU](https://dockr.ly/2nuZUkU)
- **Ejecución de Docker en Hyper-V con Docker Machine**: [http://bit.ly/2HGMPiI](http://bit.ly/2HGMPiI)
- **Desarrollo dentro de un contenedor en VS Code**: [https://code.visualstudio.com/docs/remote/containers](https://code.visualstudio.com/docs/remote/containers)

---

### Preguntas

1. **¿Por qué es importante instalar y utilizar un gestor de paquetes en nuestro ordenador local?**
2. **Con Docker Desktop, puedes desarrollar y ejecutar contenedores Linux.**
   - a. Verdadero
   - b. Falso
3. **¿Por qué son esenciales las buenas habilidades de scripting (como Bash o PowerShell) para el uso productivo de contenedores?**
4. **¿Por qué es fundamental probar la instalación de Docker mediante comandos como `docker version` y `docker container run hello-world`?**
5. **¿De qué manera benefician las herramientas locales de Kubernetes como minikube y kind al desarrollo de aplicaciones contenerizadas?**
6. **¿Cuáles son los pros y contras de utilizar Docker CLI, Docker Desktop y Podman para la gestión de contenedores?**

---

### Respuestas

1. **Importancia de los gestores de paquetes**:  
   Gestores como `apk`, `apt` o `yum` en Linux, `Homebrew` en macOS y `Chocolatey` en Windows facilitan la automatización y repetibilidad en la instalación de aplicaciones, librerías y herramientas, eliminando la necesidad de interactuar manualmente con asistentes gráficos.

2. **La respuesta es Verdadero**:  
   Sí, con Docker Desktop para Windows, macOS y Linux puedes desarrollar y ejecutar contenedores Linux (en Windows también es posible ejecutar contenedores nativos de Windows).

3. **Relevancia del scripting**:  
   Los scripts permiten automatizar tareas repetitivas y reducir errores humanos. Construir, probar, compartir y ejecutar contenedores son procesos que deben automatizarse para garantizar consistencia y confiabilidad.

4. **Verificación de la instalación**:  
   `docker version` confirma que el cliente y el servidor (Docker Engine) se comunican correctamente, mientras que `docker container run hello-world` valida la capacidad del sistema para descargar imágenes desde un registro y ejecutar contenedores de forma operativa.

5. **Beneficio de minikube y kind**:  
   Permiten simular clústeres reales de Kubernetes (de uno o varios nodos) de forma local y sin incurrir en costes de nube, facilitando pruebas de despliegue, estrategias de escalado y resolución de problemas en un entorno controlado antes de pasar a producción.

6. **Comparativa de herramientas de gestión de contenedores**:
   - **Docker CLI**:
     - *Pros*: Ligero, directo, ideal para automatización y pipelines de CI/CD.
     - *Contras*: Curva de aprendizaje más pronunciada para principiantes; carece de interfaz gráfica.
   - **Docker Desktop**:
     - *Pros*: Interfaz gráfica completa e intuitiva, configuración integral en macOS/Windows, integración directa con Kubernetes.
     - *Contras*: Mayor consumo de recursos del sistema; sujeto a licencias comerciales en entornos corporativos grandes.
   - **Podman**:
     - *Pros*: Sin demonio en segundo plano (*daemonless*), soporte nativo para ejecución sin root (*rootless*), bajo consumo de recursos y alta compatibilidad con comandos de Docker.
     - *Contras*: Interfaz gráfica menos desarrollada y configuración adicional requerida en macOS y Windows mediante máquinas virtuales.
