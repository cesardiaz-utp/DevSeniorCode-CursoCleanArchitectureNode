# TEMA 6: Despliegue en docker y en nube de Aplicaciones Full Stack

**Duración**: 120 min (30 min Teoría / 90 min Práctica)

## 1. INTRODUCCIÓN Y OBJETIVOS

¡Bienvenidos a la sesión final de nuestro programa *Full Stack Software Architecture*!

A lo largo de las sesiones anteriores hemos diseñado, modelado e implementado el *Fintech Core System*, garantizando aislamiento de dominio con Arquitectura Limpia, validaciones estrictas de tipos con TypeScript, consistencia transaccional ACID mediante PostgreSQL y Prisma ORM, y una interfaz reactiva y atómica en React.

No obstante, en la ingeniería de software profesional existe un axioma ineludible: **el software no genera valor mientras resida únicamente en el entorno local del desarrollador**. El clásico argumento *"en mi máquina sí funciona"* denota una falla estructural en el proceso de empaquetado, portabilidad y entrega de software.

En esta sesión abordaremos la transición de un sistema monolítico desacoplado hacia un artefacto listo para producción. Comprenderemos la contenerización a bajo nivel, implementaremos empaquetados reproducibles mediante *Multi-stage builds*, orquestaremos servicios con persistencia en PostgreSQL 18, diseñaremos estrategias seguras para migraciones automáticas en arranques en frío, e introduciremos las bases de Integración y Entrega Continua (CI/CD) con gestión profesional de secretos.

**Objetivos de la sesión:**

1. Comprender los fundamentos de la contenerización frente a la virtualización tradicional y su rol en la paridad de entornos.
2. Dominar la metodología de construcción multi-etapa (*Multi-stage builds*) para optimizar el peso, rendimiento y superficie de ataque de imágenes de Node.js (con `pnpm`) y React (con Nginx).
3. Analizar la diferencia entre cargas de trabajo con estado (*stateful*) y sin estado (*stateless*), mitigando condiciones de carrera durante la migración de esquemas en bases de datos relacionales efímeras.
4. Comprender la teoría de CI/CD (Integración Continua, Entrega Continua y Despliegue Continuo) y el ciclo de vida de un pipeline automatizado.
5. Orquestar el sistema completo en local mediante `docker-compose` e implementar el despliegue en la nube gestionando variables de entorno según los principios de [*The Twelve-Factor App*](https://12factor.net/es/).

## 2. MARCO TEÓRICO: Contenerización, Orquestación y CI/CD (30 min)

### 2.1. Máquinas Virtuales vs. Contenedores Docker

Tradicionalmente, desplegar implicaba configurar un servidor desde cero, instalar Node, PostgreSQL, configurar el proxy inverso, y esperar que las versiones no entraran en conflicto.

**Docker** cambia este paradigma empaquetando el código, las dependencias y el sistema de archivos necesario en una unidad estandarizada llamada **Contenedor**.

```mermaid
graph TD
  subgraph "Arquitectura Tradicional (VMs)"
      HW1[Hardware] --> HostOS1[Host OS]
      HostOS1 --> Hypervisor[Hypervisor]
      Hypervisor --> VM1[VM 1: Node + API]
      Hypervisor --> VM2[VM 2: Postgres DB]
  end

  subgraph "Arquitectura Docker (Contenedores)"
      HW2[Hardware] --> HostOS2[Host OS]
      HostOS2 --> DockerEngine[Docker Engine]
      DockerEngine --> C1[Contenedor API]
      DockerEngine --> C2[Contenedor DB]
      DockerEngine --> C3[Contenedor Frontend]
  end
```

#### 2.1.1. Máquinas Virtuales (VMs)

Las máquinas virtuales operan mediante una capa de abstracción de hardware denominada **Hypervisor** (ej. VMware, Hyper-V, KVM). Cada máquina virtual emula por completo una placa madre, interfaces de red, CPU virtual y, crucialmente, ejecuta un **Sistema Operativo Invitado (Guest OS) completo**.

- **Sobrecarga de recursos**: Una VM requiere gigabytes de memoria RAM y almacenamiento solo para el arranque de su propio kernel, módulos y demonios del sistema.
- **Tiempos de arranque**: El ciclo de encendido requiere el inicio completo del sistema operativo huésped (del orden de decenas de segundos o minutos).

#### 2.1.2. Contenedores (Containerization)

Un contenedor **no emula hardware ni ejecuta un kernel propio**. En su lugar, todos los contenedores en ejecución comparten de forma directa el **Kernel del Host**. El aislamiento se logra a través de tres características nativas del kernel de Linux:

1. **Namespaces (Aislamiento de visión)**: Segmentan lo que un proceso puede "ver". Existen namespaces para procesos (`PID`), redes (`NET`), sistemas de archivos (`MNT`), usuarios (`UID`) y comunicación entre procesos (`IPC`). Por tanto, un proceso en un contenedor cree que es el PID 1 y que posee su propia interfaz de red privada.
2. **Control Groups o Cgroups (Aislamiento de recursos)**: Gobiernan cuánto hardware puede consumir un proceso o grupo de procesos (límites estrictos de memoria RAM, cuotas de ciclos de reloj de CPU, I/O en disco).
3. **Union File System (UnionFS / OverlayFS)**: Permite superponer capas de archivos de solo lectura con una capa superior escribible efímera.

**En resumen**: un contenedor no es una máquina virtual ligera; **un contenedor es un proceso estándar de Linux aislado mediante namespaces** y restringido por **cgroups**.

### 2.2. Anatomía de Imágenes, Estratificación y *Multi-Stage Builds*

Una imagen de Docker es un paquete inmutable que contiene el código, dependencias, librerías del sistema y configuraciones requeridas para ejecutar un proceso.

#### 2.2.1. La Estructura de Capas (Layering) y Cache

Cada instrucción en un `Dockerfile` (como `COPY`, `RUN`, `ADD`) crea una **capa de solo lectura (*read-only layer*)**. Docker utiliza un mecanismo de almacenamiento en caché (*cache invalidation*):

- Si el contenido copiado en una capa no ha cambiado, Docker reutiliza el resultado de la compilación previa.
- En cuanto una capa se invalida (por ejemplo, modificamos un archivo de código fuente en un `COPY . .`), **todas las capas subsecuentes deben reconstruirse obligatoriamente**.

Por esta razón, en aplicaciones basadas en Node.js y `pnpm`, es un antipatrón arquitectónico copiar todo el código antes de resolver dependencias. La estrategia correcta consiste en:

1. Copiar manifiestos de dependencias (`package.json`, `pnpm-lock.yaml`).
2. Descargar e instalar las dependencias (capa cacheada mientras no se agreguen librerías).
3. Copiar el resto del código fuente y compilar.

```mermaid
flowchart LR
    subgraph Flujo Tradicional Ineficiente
        A1[COPY . .] --> B1[pnpm install] --> C1[pnpm build]
        Note1[Cualquier cambio de 1 línea de código invalida la descarga de dependencias]
    end

    subgraph Flujo Optimizado con Cache de Capas
        A2[COPY package.json pnpm-lock.yaml] --> B2[pnpm install --frozen-lockfile]
        B2 --> C2[COPY . .]
        C2 --> D2[pnpm build]
        Note2[El cache de librerías solo se invalida si cambia el lockfile]
    end
```

#### 2.2.2. Construcción Multi-Etapa (*Multi-Stage Builds*)

Para desplegar un proyecto TypeScript o React, necesitamos compiladores pesados (`tsc`, Vite, Babel), linters, tipados y utilidades de desarrollo (`devDependencies`). Sin embargo, para ejecutar el servidor en producción, **únicamente requerimos el código JavaScript transpilado** (`/dist`) y **el runtime de Node.js o el servidor web Nginx**.

Un *Multi-Stage Build* resuelve este dilema dividiendo el `Dockerfile` en etapas con alcances independientes:

- **Etapa de Construcción (`builder`)**: Dispone del SDK completo, gestores de paquetes y herramientas de desarrollo para producir los artefactos binarios o archivos estáticos.
- **Etapa de Ejecución (`runner`)**: Inicia desde una imagen base mínima (ej. `node:24-alpine` o `nginx:alpine`), desechando el compilador, las librerías de desarrollo y las herramientas de empaquetado. Únicamente se copian los artefactos resultantes desde la etapa `builder`.

**Beneficios arquitectónicos**:

- **Reducción de tamaño**: La imagen pasa de ocupar $\approx 1.2\text{ GB}$ a menos de $80\text{ MB}$.
- **Seguridad (Superficie de Ataque)**: Al eliminar gestores de paquetes, shells redundantes o compiladores del contenedor final, los vectores de explotación de vulnerabilidades en producción se minimizan radicalmente.

### 2.3. Cargas de Trabajo: *Stateful* vs. *Stateless* y el Ciclo de Vida de Migraciones

En la arquitectura de sistemas distribuidos, clasificamos nuestros componentes según cómo manejan el estado:

1. **Servicios Sin Estado (*Stateless Workloads*)**: La API backend y el cliente frontend no retienen información transaccional en su memoria volátil ni en su disco local. Pueden destruirse, reiniciarse o replicarse horizontalmente en decenas de instancias sin pérdida de datos.
2. **Servicios Con Estado (*Stateful Workloads*)**: El motor de base de datos (PostgreSQL 18) depende críticamente de la durabilidad y persistencia de sus bloques de almacenamiento.

```mermaid
graph TD
    subgraph Frontend y Backend - Stateless
        F[Frontend Pod/Contenedor]
        B[Backend Pod/Contenedor]
    end

    subgraph Base de Datos - Stateful
        PG[PostgreSQL 18 Container] --> Vol[(Volumen Persistente / Block Storage)]
    end

    F -->|HTTP / REST| B
    B -->|TCP / Connection Pool| PG
```

#### El Desafío de las Bases de Datos Efímeras en Despliegue

Cuando instanciamos un clúster local con `docker-compose` o aprovisionamos un servidor de base de datos administrado en la nube (PaaS/IaaS), el motor de base de datos arranca con un catálogo relacional completamente vacío. No existen las tablas `User`, `Account` ni `Transaction`.

Si el contenedor de la API inicia e intenta atender peticiones HTTP o ejecutar consultas de introspección antes de que el esquema exista, colapsará con errores de conexión o excepciones fatales (`Table "public"."User" does not exist`).

#### `prisma migrate deploy` vs. `prisma db push`

Es fundamental distinguir las herramientas de sincronización de esquemas:

- **`prisma db push` (Uso exclusivo en prototipado local)**: Fuerza a la base de datos a coincidir con el archivo `schema.prisma` actual sin registrar un historial determinista. Si detecta conflictos, puede provocar pérdida involuntaria de datos (*data loss*). **Está prohibido en entornos productivos**.
- **`prisma migrate deploy` (Estándar para Producción)**: Lee la carpeta de migraciones versionadas (`prisma/migrations/`) y compara los scripts SQL ejecutados con la tabla de auditoría interna `_prisma_migrations`. Aplica únicamente las migraciones pendientes en estricto orden cronológico dentro de transacciones atómicas.

**Estrategia de Inicialización (Cold-start synchronization)**:
El ciclo de arranque de nuestro contenedor backend debe implementar una orquestación secuencial:

$$\text{Arranque} \longrightarrow \text{Verificación de Conexión DB} \longrightarrow \text{Ejecución de Migraciones} \longrightarrow \text{Lanzamiento de API}$$

### 2.4. Fundamentos de CI/CD (Integración, Entrega y Despliegue Continuo)

La metodología DevOps y la entrega moderna de software se articulan en torno a pipelines automatizados:

```mermaid
flowchart LR
    Dev[Push de Código] --> CI[CI: Continuous Integration]
    CI --> CDeliv[CD: Continuous Delivery]
    CDeliv --> CDeploy[CD: Continuous Deployment]

    subgraph CI [Integración Continua]
        Lint[Linting / Typing] --> UnitTests[Pruebas Unitarias]
        UnitTests --> Build[Compilación / Empaque]
    end

    subgraph CDeliv [Entrega Continua]
        Staging[Generación de Release] --> Approval[Aprobación Manual / Gate]
    end

    subgraph CDeploy [Despliegue Continuo]
        AutoDeploy[Despliegue Automático a Producción]
    end
```

#### 2.4.1. Integración Continua (CI - Continuous Integration)

Práctica de desarrollo donde los miembros del equipo integran su trabajo de forma continua en el repositorio principal (generalmente `main` o `develop`). Cada integración desencadena la construcción automatizada del proyecto y la ejecución de la suite de pruebas.

- **Objetivo**: Detectar errores de tipado, regresiones de código y fallos de integración tan pronto como se escriben, impidiendo la mezcla de código defectuoso.
- **Herramientas**: GitHub Actions, GitLab CI, CircleCI, Jenkins.

#### 2.4.2. Entrega Continua (CD - Continuous Delivery)

Extensión de la Integración Continua donde el código validado se empaqueta automáticamente en un artefacto desplegable (por ejemplo, una imagen de Docker etiquetada y subida a un registro como Docker Hub o GitHub Packages). El artefacto está probado y listo para liberarse a producción, requiriendo únicamente una decisión o confirmación manual para su despliegue final.

#### 2.4.3. Despliegue Continuo (CD - Continuous Deployment)

Elimina por completo la aprobación manual. Cada cambio que pasa exitosamente todas las etapas del pipeline de CI/CD se despliega automáticamente en los entornos de producción en tiempo real. Requiere una cobertura de pruebas exhaustiva y mecanismos de telemetría y reversión (*rollback*) inmediata.

### 2.5. Gestión de Secretos y Configuración según *The Twelve-Factor App*

El estándar arquitectónico para el desarrollo de aplicaciones nativas en la nube (*Twelve-Factor App*) establece en su **Tercer Factor (Configuración)**: *Almacena la configuración en el entorno*.

#### Secretos en Tiempo de Compilación vs. Tiempo de Ejecución

Existe una divergencia crítica entre el Backend (Node.js) y el Frontend (SPA / React):

1. **Backend (Runtime Injection)**: Node.js se ejecuta en un servidor. Las variables como `DATABASE_URL` y `JWT_SECRET` se leen en tiempo de ejecución (`process.env.VARIABLE`) desde la memoria del sistema operativo del contenedor. Nunca se hornean dentro de la imagen.
2. **Frontend (Build-Time Injection)**: React y Vite no se ejecutan en un servidor con Node.js en producción; son un conjunto de archivos estáticos (`.html`, `.js`, `.css`) interpretados por el **navegador web del cliente**.
    - Por tanto, las variables de entorno con prefijo `VITE_` se reemplazan como texto plano durante la ejecución de `pnpm build`.
    - **Regla de oro de seguridad**: Jamás expongas claves privadas, firmas criptográficas ni credenciales maestras en variables del frontend, puesto que cualquier usuario puede inspeccionarlas mediante las herramientas de desarrollador del navegador.

## 3. DESARROLLO PRÁCTICO (90 min)

Vamos a transformar nuestro repositorio en un sistema de despliegue automatizado.

### 3.1. Dockerizando el Backend (Node.js + Prisma + pnpm)

En la raíz del proyecto backend (`fintech-core-app`), primero crearemos un archivo `.dockerignore` para evitar que Docker copie archivos innecesarios al construir la imagen:

#### Archivo `fintech-core-app/.dockerignore`

Evita la fuga de dependencias locales y archivos de configuración del host hacia el contexto de construcción:

```plain copy
node_modules
dist
.env
.npm
pnpm-debug.log
```

#### Archivo `fintech-core-app/pnpm-workspace.yaml`

Para permitir la instalación de dependencias nativas de **Prisma**, `bcrypt` y `esbuild` en entornos de construcción aislados, debemos habilitar explícitamente la construcción de estos paquetes:

```yaml copy
allowBuilds:
  '@prisma/engines': true
  bcrypt: true
  esbuild: true
  prisma: true
```

#### Archivo `fintech-core-app/Dockerfile`

Implementaremos un Multi-stage build con Node 24 en Alpine Linux para optimizar el tamaño de la imagen final y reducir la superficie de ataque:

```Dockerfile copy
# ==========================================================
# ETAPA 1: Builder (Compilación y generación de artefactos)
# ==========================================================
FROM node:24-alpine AS builder

# Habilitar pnpm
RUN npm i -g pnpm@12.6.0

WORKDIR /app

# Copiar manifiestos de dependencias
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY prisma.config.ts ./
COPY prisma ./prisma/

# Instalar TODAS las dependencias (necesarias para build)
RUN pnpm install --frozen-lockfile

# Copiar el código fuente
COPY . .

# Generar el cliente de Prisma (Crucial antes de compilar)
RUN pnpm exec prisma generate

# Compilar TypeScript a JavaScript (/dist)
RUN pnpm build

# ==========================================================
# ETAPA 2: Runner (Entorno de Ejecución Mínimo en Producción)
# ==========================================================
FROM node:24-alpine AS runner

# Habilitar pnpm
RUN npm i -g pnpm@12.6.0

WORKDIR /app

ENV NODE_ENV=production

# Copiar manifiestos de dependencias
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY prisma.config.ts ./
COPY prisma ./prisma/

# Instalar SOLO dependencias de producción
RUN pnpm install --prod --frozen-lockfile

# Copiar el cliente de Prisma generado y el build
COPY --from=builder /app/dist ./dist

EXPOSE 3000

# SCRIPT DE ARRANQUE RESILIENTE:
# 1. Aplica migraciones pendientes sobre la base de datos (seguro para tablas vacías o incrementales)
# 2. Inicializa el proceso principal de Node.js
CMD ["sh", "-c", "pnpm dlx prisma@7.9.1 migrate deploy && node dist/server.js"]
```

Para construir la imagen del backend y etiquetarla como `fintech-backend:latest`:

```bash copy
docker build -t fintech-backend:latest .
```

### 3.2. Dockerizando el Frontend (React + Vite + Nginx)

El frontend no requiere el runtime de Node.js en producción. La etapa final utilizará **Nginx Alpine** como un servidor web estático de ultra alto rendimiento.

#### Archivo `fintech-core-front/.dockerignore`

```plain copy
node_modules
dist
.env
.env.local
pnpm-debug.log
.git
```

#### Archivo de Configuración de Servidor Web (`fintech-core-front/nginx.conf`)

En una SPA (*Single Page Application*), el enrutamiento lo maneja el cliente mediante la History API del navegador. Si un usuario refresca una ruta como `/dashboard`, el servidor Nginx debe retornar siempre el archivo `index.html` en lugar de un error `404`:

```nginx copy
server {
    listen 80;
    server_name localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        # Redirigir cualquier petición que no sea un archivo físico hacia index.html
        try_files $uri $uri/ /index.html;
    }

    # Desactivar logs innecesarios para favicon y assets para optimizar I/O
    location = /favicon.ico { 
        access_log off; 
        log_not_found off; 
    }

    # Manejo de compresión gzip para acelerar la carga de assets
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
}
```

#### Archivo `fintech-core-front/pnpm-workspace.yaml`

Para permitir la instalación de dependencias nativas de `esbuild` en entornos de construcción aislados, debemos habilitar explícitamente la construcción de este paquete:

```yaml copy
allowBuilds:
  esbuild: true
```

#### Archivo `fintech-core-front/Dockerfile`

```Dockerfile copy
# ==========================================================
# ETAPA 1: Builder (Compilación de React con Vite y pnpm)
# ==========================================================
FROM node:24-alpine AS builder

RUN npm i -g pnpm@12.6.0

WORKDIR /app

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
RUN pnpm install --frozen-lockfile

COPY . .

# Argumento de construcción: Inyección de la URL del API Backend
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL

# Transpilar y empaquetar el frontend a HTML/CSS/JS minificado (/app/dist)
RUN pnpm build

# ==========================================================
# ETAPA 2: Servidor Web de Archivos Estáticos (Nginx)
# ==========================================================
FROM nginx:alpine

# Sustituir la configuración por defecto de Nginx por nuestra configuración para SPAs
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copiar el paquete compilado desde la etapa de construcción
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Para construir la imagen del frontend y etiquetarla como `fintech-frontend:latest`:

```bash copy
docker build -t    .
```

### 3.3. Orquestación Local Completa con Docker Compose

Para orquestar la relación entre **PostgreSQL 18**, la **API Backend** y el **Cliente Frontend**, crearemos un nuevo proyecto raíz llamado `fintech-infra` con la configuración necesaria para el despliegue local. En este proyecto, creamos un manifiesto declarativo en la raíz global del proyecto.

Se inicializará un repositorio Git independiente para la infraestructura, con el siguiente comando:

```bash copy
mkdir fintech-infra && cd fintech-infra
git init
```

#### Archivo `fintech-infra/.gitignore`

```plain copy
.env
```

#### Archivo de Secretos Locales (`fintech-infra/.env` en la raíz)

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=password
POSTGRES_DB=dbname
POSTGRES_PORT=5432

JWT_SECRET=default_secret
JWT_EXPIRES_IN="1h"

BACKEND_PORT=3000
FRONTEND_PORT=8080
```

#### Archivo `fintech-infra/docker-compose.yml`

Implementa *Healthchecks* activos sobre PostgreSQL 18 para evitar arranques fallidos por falta de disponibilidad:

```yaml
services:
  # -------------------------------------------------------------
  # CAPA DE PERSISTENCIA: Motor Relacional PostgreSQL 18
  # -------------------------------------------------------------
  postgres-db:
    image: postgres:18-alpine
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "${POSTGRES_PORT}:5432"
    volumes:
      - postgres_fintech_data:/var/lib/postgresql//18/docker
    healthcheck:
      # Verificación periódica para asegurar que el motor acepte conexiones
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5

  # -------------------------------------------------------------
  # CAPA DE APLICACIÓN: Backend API REST (Node.js + Prisma)
  # -------------------------------------------------------------
  backend:
    build:
      context: ../fintech-core-app
      dockerfile: Dockerfile
    restart: always
    environment:
      # Resolución interna mediante el nombre del servicio 'postgres-db' en la red virtual
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres-db:5432/${POSTGRES_DB}?schema=public
      - JWT_SECRET=${JWT_SECRET}
      - JWT_EXPIRES_IN=${JWT_EXPIRES_IN}
      - PORT=3000
    ports:
      - "${BACKEND_PORT}:3000"
    depends_on:
      postgres-db:
        # Condición estricta: No arranca hasta que el Healthcheck de Postgres sea exitoso
        condition: service_healthy

  # -------------------------------------------------------------
  # CAPA DE PRESENTACIÓN: Frontend Cliente Web (React + Nginx)
  # -------------------------------------------------------------
  frontend:
    build:
      context: ../fintech-core-front
      dockerfile: Dockerfile
      args:
        # El navegador del cliente consume el backend a través del puerto expuesto en el host
        - VITE_API_URL=http://localhost:${BACKEND_PORT}
    restart: always
    ports:
      - "${FRONTEND_PORT}:80"
    depends_on:
      - backend

volumes:
  # Volumen persistente administrado para PostgreSQL 18
  postgres_fintech_data:
```

#### Comandos de Ejecución y Monitoreo Local

- Construir y levantar todo el ecosistema en segundo plano

  ```bash copy
  docker compose up -d --build
  ```

- Visualizar logs del backend y comprobar la ejecución de migraciones

  ```bash copy
  docker compose logs -f backend
  ```

- Verificar el estado de salud de los contenedores

  ```bash copy
  docker compose ps
  ```

- Apagar y desmontar los servicios conservando los datos persistentes

  ```bash copy
  docker compose down
  ```

### 3.4. Despliegue en la Nube (Platform as a Service)

Aprovisionaremos la infraestructura en una plataforma PaaS moderna con soporte para PostgreSQL y contenedores Docker.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Desarrollador
    participant Cloud as Plataforma en la Nube (Render/Railway)
    participant DB as Managed PostgreSQL 18
    participant API as Backend Service (Docker)
    participant Web as Frontend Service (Nginx)

    Dev->>Cloud: Provisionar PostgreSQL 18
    Cloud-->>Dev: Retorna DATABASE_URL (Internal connection string)
    
    Dev->>Cloud: Crear Backend Service desde /backend
    Dev->>Cloud: Inyectar DATABASE_URL y JWT_SECRET en Environment
    Cloud->>API: Construir Dockerfile & Ejecutar Contenedor
    API->>DB: prisma migrate deploy (Crea tablas en DB vacía)
    API-->>Cloud: Servicio saludable en puerto 3000
    Cloud-->>Dev: Retorna URL Pública del Backend (ej. https://api-fintech.onrender.com)

    Dev->>Cloud: Crear Frontend Service desde /frontend
    Dev->>Cloud: Inyectar Build Arg / Env VITE_API_URL=https://api-fintech.onrender.com/api
    Cloud->>Web: Construir con Vite & Servir con Nginx
    Cloud-->>Dev: URL Pública del Frontend (ej. https://app-fintech.onrender.com)
```

Para esta practica, utilizaremos **[Render](https://render.com/)** como proveedor de nube, aunque [Railway](https://railway.app/) o [Fly.io](https://fly.io/) son alternativas igualmente válidas.

#### Paso 1: Aprovisionamiento de la Base de Datos Relacional

1. Accede al panel de control de la nube ([Render](https://render.com/)) y crea una instancia de **PostgreSQL**.
2. Verifica que el motor corresponda a PostgreSQL 18 (o la versión superior soportada por el proveedor).
3. Localiza la sección de credenciales y copia la **Internal Database URL** (URL de conexión segura de la red interna privada). Su estructura es:
`postgresql://usuario:contraseña@dpg-xxxxxx:5432/fintech_db`

#### Paso 2: Despliegue del Backend

1. Crea un nuevo **Web Service** vinculado a tu repositorio de GitHub.
2. Configura el *Root Directory* apuntando a la subcarpeta `backend`.
3. En el campo **name**, asigna un nombre descriptivo y único (ej. `fintech-core-app`).
4. Selecciona el entorno de ejecución basado en **Docker** (detectará automáticamente el `Dockerfile`).
5. **Configuración de Variables de Entorno y Secretos**:
    - `DATABASE_URL`: Asigna el valor de la URL obtenida en el Paso 1.
    - `JWT_SECRET`: Define una cadena criptográfica robusta para producción (mínimo 64 caracteres alfanuméricos).
    - `JWT_EXPIRES_IN`: el tiempo de expiración deseado.
    - `PORT`: `3000` (o el asignado por el PaaS).
6. **Inicialización y Despliegue**:
    - Al ejecutarse el contenedor, el comando `pnpm dlx prisma@7.9.1 migrate deploy` detectará que la base de datos remota está vacía e instanciará las tablas de forma automática.
    - Copia la URL pública generada (ej. <https://fintech-core-app.onrender.com>).

#### Paso 3: Despliegue del Frontend

1. Crea un nuevo **Web Service** apuntando al repositorio, configurando el *Root Directory* en `frontend`.
2. Selecciona **Docker** como entorno de ejecución.
3. En el campo **name**, asigna un nombre descriptivo y único (ej. `fintech-core-front`).
4. **Inyección de Variable de Construcción**:
    - Define la variable de entorno o argumento de construcción `VITE_API_URL` con la URL de tu API desplegada: <https://fintech-core-app.onrender.com>
5. Al compilarse la imagen, Vite compilará el código consumiendo dicha URL para las peticiones HTTP.
6. Copia la URL pública generada (ej. <https://fintech-core-front.onrender.com>).

## 4. PUNTOS DE CONTROL ARQUITECTÓNICOS Y MEJORES PRÁCTICAS

1. **Paridad de Entornos (Dev / Prod Parity)**: Mediante Docker y `docker-compose`, garantizamos que la versión de PostgreSQL (versión 18), las dependencias de Node.js y los comportamientos de red sean idénticos tanto en local como en la nube.
2. **Inmutabilidad y Determinismo**: El uso sistemático del flag `--frozen-lockfile` con `pnpm` garantiza que la imagen construida en el pipeline o en el servidor use con exactitud matemática los mismos paquetes probados localmente, evitando el error de dependencias no deseadas.
3. **Seguridad y Control de CORS**: En producción, tu servidor Express en `backend/src/presentation/server.ts` debe rechazar peticiones no autorizadas. Configura el middleware de CORS para admitir exclusivamente el dominio público de tu Frontend:

    ```typescript copy
    import cors from 'cors';

    const allowedOrigins = [process.env.CLIENT_ORIGIN || 'http://localhost:8080'];
    app.use(cors({
      origin: (origin, callback) => {
        if (!origin || allowedOrigins.includes(origin)) {
          callback(null, true);
        } else {
          callback(new Error('Bloqueado por política CORS'));
        }
      },
      credentials: true
    }));
    ```

4. **Resiliencia en Arranque**: La cláusula `depends_on: { condition: service_healthy }` en `docker-compose.yml` previene una de las condiciones de carrera más frecuentes en microservicios: la API intentando aplicar migraciones antes de que el socket de PostgreSQL esté listo para aceptar conexiones.
5. **Mitigación de Fuga de Secretos**: Verifica que el comando `git status` no reporte ningún archivo `.env`. Todos los secretos de producción deben inyectarse exclusivamente a través de los paneles de control de la nube o GitHub Secrets.

## 5. TAREAS Y ENTREGABLE DE LA SESIÓN

**El Gran Entregable Final:**

Cada estudiante debe consolidar el repositorio del proyecto con los artefactos de infraestructura y el sistema desplegado en vivo:

1. **Estructura de Contenerización en el Repositorio**:
    - `backend/Dockerfile` multi-etapa optimizado con `pnpm` y comando `prisma migrate deploy`.
    - `frontend/Dockerfile` multi-etapa optimizado con Nginx y soporte para React Router (`nginx.conf`).
    - `docker-compose.yml` orquestando PostgreSQL 18, Backend y Frontend con healthchecks y volumen persistente.
2. **Despliegue Funcional en la Nube**:
    - Base de datos relacional operativa con las migraciones aplicadas.
    - API Backend conectada a la base de datos de producción.
    - Frontend en producción consumiendo los endpoints de la API.
3. **Sube los siguientes enlaces a la plataforma académica:**
    - Enlace al/los repositorio(s) de GitHub (público o con acceso al instructor).
    - Enlace a la Aplicación Frontend en Producción (URL pública para pruebas funcionales).
    - Enlace al Backend (Endpoint público base o Health Check).
    - Un video de 5 minutos mostrando la aplicación en funcionamiento, incluyendo la creación de un usuario, inicio de sesión y las distintas transacciones financieras.

**Nota**: ten en cuenta que este entregable es el recomendado, pero puedes optar por crear una nueva aplicación en la que apliques los mismos principios de arquitectura (Clean Architecture) tanto en el frontend como en el backend, contenerización y despliegue. El despliegue lo puedes hacer en la nube de tu preferencia o en un ambiente local con Docker Compose. *Lo importante es que demuestres la integración de todos los conceptos aprendidos*.

---

🚀**¡Misión cumplida, arquitectos!**🚀

Haber completado este proyecto significa que han integrado disciplinas complejas: desde el diseño conceptual con *Clean Architecture*, pasando por la consistencia transaccional con PostgreSQL y la reactividad con React, hasta llegar al despliegue nativo en la nube con Docker.

Este no es el final, sino la nueva línea base de sus estándares profesionales. Tienen en sus manos un **Fintech Core System** que demuestra habilidades de un perfil Full Stack Senior. Añádelo a su portafolio, compartan su conocimiento y sigan construyendo software de alto impacto.

¡Éxitos en sus carreras!
