# Guía de Instalación de Herramientas

Esta guía detalla los pasos necesarios para configurar tu entorno de desarrollo para el curso de Aplicaciones FullStack con NodeJs.

## 1. Node.js 24

Node.js es un entorno de ejecución de JavaScript en el servidor que incluye `npm`, el gestor de paquetes. Es fundamental para el frontend porque nos permite instalar librerías de Angular, compilar el proyecto y lanzar el servidor de desarrollo con `ng serve`.

- **Descarga**: Ve a [nodejs.org](https://nodejs.org/) y selecciona la versión 24 (elige la LTS para máxima estabilidad). Esta instalación incluye el runtime de Node y el administrador de paquetes `npm`.
- **Verificación**:

  ```bash copy
  node -v
  ```

  - Gestor de paquetes **NPM** (por defecto)

    Para verificar la version de NPM:

    ```bash copy
    npm -v
    ```

    **Nota 1**: Si estas usando PowerShell en Windows y te sale un error al intentar validar la versión de NPM:

    ```terminal
    npm : File C:\Program Files\nodejs\npm.ps1 cannot be loaded because running scripts is disabled on this system. For more information, see about_Execution_Policies at https:/go.microsoft.com/fwlink/?LinkID=135170.
    At line:1 char:1
    + npm -v
    + ~~~
        + CategoryInfo          : SecurityError: (:) [], PSSecurityException
        + FullyQualifiedErrorId : UnauthorizedAccess
    ```

    Debes ejecutar el siguiente comando para habilitar la ejecución de scripts en PowerShell:

    ```powershell copy
    Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
    ```

    Con esto, al ejecutar la verificación de NPM te debe devolver la version actual sin errores.

    [![Instalación NodeJS 24](https://img.youtube.com/vi/mf9D-t8mlM4/sddefault.jpg)](https://youtu.be/mf9D-t8mlM4)

  - Gestor de paquetes **PNPM**

    Dado las recientes noticias sobre las vulnerabilidades del gestor de paquetes NPM, entonces es posible reemplazarlo por [PNPM](https://pnpm.io) o [YARN](https://yarnpkg.com). En esta guía seguiremos usando **PNPM**.

    Para instalar la ultima version de PNPM:

    ```bash copy
    npm install -g pnpm
    ```

    Para verificar la version instalada:

    ```bash copy
    pnpm -v
    ```

## 2. Git y GitHub

Git es el sistema de control de versiones que usamos para llevar un historial de los cambios en el código. GitHub es una plataforma remota que nos permite alojar repositorios, colaborar con otros y hacer copias de seguridad de nuestro trabajo.

- **Git**: Descárgalo desde [git-scm.com](https://git-scm.com/). La instalación te dará acceso a comandos como `git commit`, `git push` y `git pull`.
- **GitHub**: Crea una cuenta en [github.com](https://github.com/) si aún no la tienes; allí alojarás tus repositorios y podrás compartir proyectos y colaborar.
- **Configuración inicial**: Una vez instalado Git, define tu identidad global con estos comandos para que tus commits estén correctamente atribuidos:

  ```bash
  git config --global user.name "Tu Nombre"
  git config --global user.email "tu@email.com"
  ```

  También puedes configurar otros ajustes como el editor predeterminado o el formato de línea de finalización dependiendo de tu entorno.

[![Git y Github](https://img.youtube.com/vi/-f_WEMKD0NI/sddefault.jpg)](https://youtu.be/-f_WEMKD0NI)

## 3. Docker Desktop

Docker Desktop es una aplicación que facilita la construcción, el envío y la ejecución de aplicaciones y microservicios en contenedores Docker. Es esencial para nuestro curso, ya que nos permitirá levantar bases de datos (como PostgreSQL) y otros servicios de forma aislada y reproducible, sin interferir con el sistema operativo principal.

- **Descarga**: Visita el sitio oficial de [Docker Desktop](https://www.docker.com/products/docker-desktop/) y descarga la versión adecuada para tu sistema operativo.

- **Requisitos del Sistema y Verificación**:
  - **Virtualización**: Docker Desktop depende de la virtualización a nivel de hardware. Debes asegurarte de que esté habilitada en la BIOS/UEFI de tu equipo.

  - **Verificación en Windows**:
    - **Requisitos**: Windows 10/11 de 64 bits (compilación 19044 o superior), 4 GB de RAM y soporte para WSL 2.
    - **Comando (PowerShell/CMD)**: Abre una terminal y ejecuta `systeminfo`. Busca la sección "Hyper-V Requirements". Si "Virtualization Enabled In Firmware" dice "Yes", estás listo.

      ```bash copy
      systeminfo
      ```

      También, es necesario confirmar que la version de WSL2 (Windows Subsystem for Linux) está instalada en la version adecuada:

      ```bash copy
      wsl --version
      ```

      Este comando muestra la versión de WSL instalada. Si ves "WSL Version: 2.X.X", tienes WSL 2. Si no aparece un número de versión, tienes WSL 1.

      Si tienes privilegios de administrador, puedes instalar actualizaciones del kernel de WSL utilizando PowerShell o el Símbolo del sistema. Escribe PowerShell o Símbolo del sistema en la barra de búsqueda de Windows y selecciona "Ejecutar como administrador". Luego, escribe `wsl --update` y presiona Enter.

- **Verificación en macOS**:
  - **Requisitos**: macOS 12 (Monterey) o superior, 4 GB de RAM.
  - **Comando (Terminal)**: Los Macs modernos (con Apple Silicon o Intel post-2010) suelen tener la virtualización por defecto. Para verificar en un Mac con Intel, puedes ejecutar:

      ```bash
      sysctl -a | grep -o VMX
      ```

      Si el comando devuelve "VMX", la virtualización está soportada. Los Mac con Apple Silicon la soportan de forma nativa.

- **Verificación en Linux**:
  - **Requisitos**: Una distribución de 64 bits con entorno de escritorio (Ubuntu, Debian, Fedora, Arch), 4 GB de RAM y soporte para KVM.
  - **Comando (Terminal)**: Para comprobar si tu CPU soporta virtualización de hardware, ejecuta:

      ```bash
      grep -c -E '(vmx|svm)' /proc/cpuinfo
      ```

      Un resultado de `1` o superior indica que es compatible.

- **Instalación (Windows)**:
  - Ejecuta el instalador (`Docker Desktop Installer.exe`).
  - Asegúrate de que la opción "Use WSL 2 instead of Hyper-V" esté marcada. WSL 2 (Windows Subsystem for Linux 2) es el backend recomendado para Docker Desktop en Windows.
  - Sigue las instrucciones del asistente. Puede que necesites reiniciar tu equipo.
  - Después de la instalación, Docker Desktop se iniciará automáticamente. Si no lo hace, búscalo en el menú de inicio y ejecútalo.
  - Es posible que te pida habilitar la virtualización en la BIOS/UEFI de tu equipo si no está activa.

- **Instalación (macOS)**:
  - Una vez descargado el archivo `.dmg`, ábrelo.
  - Arrastra el icono de Docker a la carpeta de Aplicaciones.
  - Ve a la carpeta de Aplicaciones y ejecuta Docker. Un icono de Docker en la barra de menú indicará que se está ejecutando.
  - Para Macs con Apple Silicon, es recomendable instalar Rosetta 2 si no lo tienes, ya que algunas herramientas de línea de comandos opcionales lo requieren.

- **Instalación (Linux)**:
  - Docker Desktop para Linux se ejecuta en una máquina virtual, lo que lo aísla de cualquier instalación de Docker Engine que ya tengas.
  - Descarga el paquete apropiado para tu distribución (por ejemplo, `.deb` para Debian/Ubuntu, `.rpm` para Fedora/CentOS, o `.pkg.tar.zst` para Arch).
  - Instala el paquete usando el gestor de paquetes de tu sistema. Por ejemplo, en Ubuntu: `sudo apt-get update && sudo apt-get install ./docker-desktop-<version>-<arch>.deb`.
  - Inicia Docker Desktop desde el menú de aplicaciones de tu escritorio o ejecutando `systemctl --user start docker-desktop` en una terminal.

[![Docker Desktop](https://img.youtube.com/vi/rIPI41LMfFQ/sddefault.jpg)](https://youtu.be/rIPI41LMfFQ)

## 4. Postgresql (Opcional)

PostgreSQL es un sistema de gestión de bases de datos relacionales de código abierto y potente. En este curso, lo utilizaremos para almacenar de forma persistente toda la información de nuestra aplicación (reservas, usuarios, productos, etc.), aprovechando su robustez y compatibilidad con Spring Data JPA.

- **Descarga**: Ve a la página oficial de [PostgreSQL](https://www.postgresql.org/download/) y selecciona el instalador para tu sistema operativo (Windows, macOS o Linux).
- **Instalación**:
  - Ejecuta el instalador y sigue los pasos del asistente.
  - **Contraseña del superusuario**: Durante la instalación, se te pedirá una contraseña para el usuario `postgres`. Asegúrate de recordarla, ya que la necesitarás para configurar la conexión en Spring Boot.
  - **Puerto**: Por defecto utiliza el `5432`. Déjalo así a menos que ya esté ocupado.
  - **Stack Builder**: Al finalizar, no es necesario instalar complementos adicionales a través de Stack Builder para este curso.
- **Herramienta de Administración (pgAdmin)**: El instalador suele incluir **pgAdmin**, una interfaz gráfica que te permitirá crear la base de datos `mi_basededatos` de forma visual antes de conectar la aplicación.
- **Verificación**: Abre pgAdmin o utiliza la terminal (`psql -U postgres`) para confirmar que puedes acceder al servidor de base de datos.

[![Instalación PostgreSQL 18](https://img.youtube.com/vi/l6fb-KINGiE/sddefault.jpg)](https://youtu.be/l6fb-KINGiE)

---
_Con todas las herramientas instaladas y configuradas, ¡llegó el momento de empezar a construir nuestra aplicación!_
