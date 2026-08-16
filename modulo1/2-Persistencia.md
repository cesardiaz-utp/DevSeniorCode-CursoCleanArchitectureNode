# SESIÓN 2: Persistencia Relacional con PostgreSQL y Prisma ORM

**Duración**: 2 horas (30 min Teoría / 90 min Práctica)

## 1. INTRODUCCIÓN Y CONTEXTO

En la Sesión 1, construimos el corazón inmutable de nuestra _Fintech Core App_: **La Capa de Dominio**. Definimos nuestras entidades puras (`User`, `Account`), establecimos invariantes de negocio estrictas, y estructuramos nuestro modelo de transacciones utilizando polimorfismo avanzado (donde `Transaction` actúa como clase abstracta, siendo heredada por `Deposit`, `Withdrawal` y `Transfer`). Hasta este punto, nuestro código vive exclusivamente en la memoria RAM y es completamente agnóstico de cualquier tecnología externa.

Hoy cruzaremos la frontera arquitectónica hacia la **Capa de Infraestructura**. Necesitamos que nuestras entidades cobren vida, persistan sus estados de forma segura y sobrevivan al reinicio del servidor o a posibles caídas del sistema. Para ello, integraremos **PostgreSQL** (un motor relacional robusto y probado en la industria financiera) utilizando **Prisma ORM** como nuestra herramienta de interacción con la base de datos.

El gran reto de la sesión de hoy será resolver un problema clásico en la ingeniería de software conocido como el **Object-Relational Impedance Mismatch (Desajuste de Impedancia Objeto-Relacional)**. Este concepto describe las dificultades técnicas y conceptuales que surgen al intentar guardar objetos (que tienen herencia, encapsulamiento e identidad) en tablas relacionales (que solo entienden de filas, columnas y llaves foráneas). Específicamente: ¿Cómo guardamos clases que usan herencia en una base de datos que usa tablas planas? Lo resolveremos de manera elegante utilizando patrones de diseño estructurales y Mappers dinámicos, manteniendo nuestro dominio 100% puro.

## 2. FUNDAMENTACIÓN TEÓRICA (30 min)

### 2.1. El Desajuste de Impedancia Objeto-Relacional (_Object-Relational Impedance Mismatch_)

El Paradigma de Orientación a Objetos (POO) y el Paradigma Relacional fueron concebidos para resolver problemas distintos bajo principios matemáticos y de modelado conceptualmente diferentes:

- **Orientación a Objetos (Dominio en TypeScript)**: Se basa en la encapsulación de estado y comportamiento, identidad de objetos en memoria, referencias navegables directas entre grafos de objetos, polimorfismo y jerarquías de herencia.
- **Modelo Relacional (PostgreSQL)**: Se basa en la teoría de conjuntos y la lógica de predicados de primer orden. Los datos se organizan en tablas bidimensionales compuestas por filas y columnas, donde las relaciones entre entidades se expresan mediante llaves primarias (`PRIMARY KEY`) y llaves foráneas (`FOREIGN KEY`).

Esta discordancia estructural se conoce como **Desajuste de Impedancia Objeto-Relacional** y se manifiesta en seis dimensiones principales:

| Dimensión de Impedancia | Paradigma Orientado a Objetos (TypeScript) | Paradigma Relacional (PostgreSQL) |
| --- | --- | --- |
| **Granularidad** | Modelos finos con objetos de valor (`Money`, `Email`). | Tablas con tipos de datos primitivos escalares (`VARCHAR`, `NUMERIC`). |
| **Herencia y Subtipado** | Jerarquías (`Transaction` $\rightarrow$ `Deposit`, `Transfer`). | No existe concepto nativo de herencia estándar entre tablas relacionales. |
| **Identidad** | Identidad por referencia en RAM (`obj1 === obj2`) o por valor business (`UUID`). | Identidad por Clave Primaria (`PK`) en una tabla específica. |
| **Asociaciones $1:N$** | Colecciones directas de referencias (`account.transactions`). | Llave foránea (`FK`) en la tabla del lado $N$ apuntando al lado $1$. |
| **Asociaciones $M:N$** | Listas de referencias en ambos sentidos (`user.roles` / `role.users`). | IMPOSIBLE de representar directamente; requiere **Tabla Pivote / Join Table**. |
| **Navegación de Datos** | Recorrido en memoria navegando propiedades (`account.user.email`). | Consultas declarativas uniendo conjuntos de datos mediante sentencias SQL. |

Un **ORM (Object-Relational Mapper)** es una herramienta de infraestructura diseñada para tender un puente de traducción bidireccional sobre este desajuste. Convierte automáticamente los registros relacionales devueltos por el motor SQL en grafos de objetos utilizables en la aplicación, y traduce las mutaciones de estado de los objetos en sentencias `INSERT`, `UPDATE` o `DELETE`.

### 2.2. El Problema Específico del Mapeo de Relaciones Muchos a Muchos ($M:N$)

El mapeo de relaciones **Muchos a Muchos** ($M:N$) es uno de los puntos donde la impedancia entre el paradigma de objetos y el relacional es más evidente.

#### Incompatibilidad Estructural Básica

En la Orientación a Objetos, una relación $M:N$ se representa de forma natural agregando colecciones directas de referencias en ambas entidades:

```typescript
// Paradigma Orientado a Objetos (Memoria RAM)
class User {
  id: string;
  roles: Role[]; // Referencia directa a múltiples roles
}

class Role {
  id: string;
  users: User[]; // Referencia directa a múltiples usuarios
}
```

En cambio, en el **Modelo Relacional**, una relación $M:N$ **no puede almacenarse directamente** en las tablas de las entidades involucradas sin violar la Primera Forma Normal (1NF), la cual exige que cada celda de una tabla contenga un único valor atómico (no arreglos ni listas embebidas).

Para resolver esto en SQL, es imperativo introducir una tercera tabla intermedia denominada**Tabla Pivote, Tabla de Unión o Join Table**:

```sql
-- Estructura física en PostgreSQL
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email VARCHAR UNIQUE
);

CREATE TABLE roles (
  id UUID PRIMARY KEY,
  name VARCHAR UNIQUE
);

-- Tabla Pivote / Join Table que rompe la relación M:N en dos relaciones 1:N
CREATE TABLE user_roles (
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
  PRIMARY KEY (user_id, role_id) -- Llave primaria compuesta
);
```

#### De Tabla Pivote Simple a Entidad Asociativa (_Associative Entity_)

Existen dos variantes fundamentales al mapear relaciones $M:N$:

1. **Relación $M:N$ Pura (Sin atributos adicionales)**: La tabla pivote solo contiene las dos llaves foráneas (`user_id`, `role_id`).

    ```mermaid
    erDiagram
        USERS ||--o{ USER_ROLES : "posee"
        ROLES ||--o{ USER_ROLES : "asignado_a"

        USERS {
            uuid id PK
            string email
        }
        
        USER_ROLES {
            uuid user_id FK, PK
            uuid role_id FK, PK
        }
        
        ROLES {
            uuid id PK
            string name
        }
    ```

2. **Relación $M:N$ Atribuida / Entidad Asociativa**: La relación misma posee información de negocio (por ejemplo, `grantedAt`, `assignedBy`, `status`, `permissions`). En este caso, en el diseño del sistema la tabla pivote deja de ser una simple tabla de unión y se promueve conceptualmente a una **Entidad del Dominio** (por ejemplo, `UserRole` o `AccountSubscription`).

    ```mermaid
    erDiagram
        USERS ||--o{ USER_ROLES : "posee"
        ROLES ||--o{ USER_ROLES : "asignado_a"

        USERS {
            uuid id PK
            string email
        }
        
        USER_ROLES {
            uuid user_id FK, PK
            uuid role_id FK, PK
            timestamp granted_at "Atributo de la relación"
            string granted_by "Atributo de la relación"
        }
        
        ROLES {
            uuid id PK
            string name
        }
    ```

#### Cómo maneja Prisma ORM las relaciones $M:N$

Prisma ofrece dos estrategias para gestionar el mapeo $M:N$:

- **Relaciones $M:N$ Implícitas**: Prisma crea y gestiona la tabla pivote en PostgreSQL de forma transparente sin necesidad de declararla explícitamente en el esquema, siempre y cuando no se requieran atributos adicionales en la relación:

  ```prisma
  model User {
    id    String @id @default(uuid()) @db.Uuid
    roles Role[] // Prisma genera implícitamente la tabla _RoleToUser
  }
  model Role {
    id    String @id @default(uuid()) @db.Uuid
    users User[]
  }
  ```

- **Relaciones $M:N$ Explícitas (Recomendado para Arquitectura Limpia)**: Se declara explícitamente el modelo intermedio en Prisma. Esto permite capturar metadatos, auditoría y mantener un mapeo directo $100\%$ transparente hacia las entidades de dominio.

### 2.3. Decisión Arquitectónica: Evaluación Técnica ORM vs. SQL Nativo

Elegir la herramienta de persistencia en un sistema financiero es una decisión de arquitectura crítica. No existe una solución perfecta; cada enfoque presenta un conjunto de ventajas y compromisos que deben alinearse con los requisitos del sistema.

#### Enfoque 1: Consultas SQL Nativas / Query Builders (`pg`, `Knex.js`, `Kysely`)

- **Ventajas**:
  - **Control de Rendimiento Absoluto**: Permite escribir sentencias SQL optimizadas manualmente, aprovechando _window functions_, CTEs (_Common Table Expressions_) complejas y planes de ejecución tuning (`EXPLAIN ANALYZE`).
  - **Cero Sobrecarga de Abstracción (Overhead)**: No hay capas intermedias traduciendo objetos; los datos fluyen directamente del driver TCP de PostgreSQL a estructuras JSON.
- **Desventajas**:
  - **Ausencia de Tipado Estricto de Extremo a Extremo**: Las consultas escritas como cadenas de texto (`"SELECT id, amnt FROM tx"`) no son validadas por el compilador de TypeScript. Un error tipográfico en el nombre de una columna se detecta únicamente en tiempo de ejecución (_runtime_).
  - **Mantenibilidad Costosa**: Las migraciones de esquemas deben gestionarse de forma manual y artesanal mediante scripts SQL por separado.
  - **Riesgo de Inyección SQL**: Si las consultas dinámicas no se parametrizan adecuadamente, se abren vectores de ataque críticos.

#### Enfoque 2: ORM Basado en Generación de Código y Esquema Declarativo (Prisma ORM)

- **Ventajas**:
  - **Seguridad de Tipos Estricta (_Type-Safety_ de Extremo a Extremo)**: Prisma analiza el esquema declarativo (`schema.prisma`) y genera un cliente TypeScript fuertemente tipado (`@prisma/client`). Si se renombra un campo en el esquema, el compilador de TypeScript marcará un error en todo el proyecto donde se intente usar la propiedad anterior.
  - **Protección Nativa contra Inyección SQL**: El cliente de Prisma parametriza de forma automática el $100\%$ de las variables enviadas a PostgreSQL.
  - **Gestión Integrada de Migraciones (`prisma migrate`)**: Genera código SQL declarativo versionado en Git, garantizando que los entornos de Desarrollo, Staging y Producción evolucionen con esquemas perfectamente sincronizados.
  - **Excelente Experiencia de Desarrollo (DX)**: Auto-completado inteligente en el IDE, reducción drástica de código repetitivo (_boilerplate_) y trazabilidad de tipos.
- **Desventajas**:
  - **Sobrecarga Marginal de Procesamiento**: Requiere una pequeña conversión en memoria para adaptar objetos de Prisma a JavaScript nativo.
  - **Limitación en Consultas Analíticas (OLAP)**: Para reportes financieros masivos o agregaciones multi-tabla hipercomplejas, el ORM puede generar SQL subóptimo.

#### Veredicto Arquitectónico para nuestro Fintech Core

Para un sistema transaccional en tiempo real (**OLTP**) centrado en la integridad de datos, la seguridad de tipos y la mantenibilidad bajo arquitectura limpia, **Prisma ORM es la elección óptima**. La garantía de detectar errores de esquemas en tiempo de compilación cancela el riesgo de operar sobre campos inexistentes en cuentas financieras. Para casos puntuales de analítica masiva donde se requiera SQL nativo, Prisma permite ejecutar consultas parametrizadas puras mediante `$queryRaw`.

### 2.4. Consistencia Relacional y Transacciones ACID en Fintech

En el desarrollo de software financiero, la consistencia absoluta de los datos no es una característica deseable, es una obligación legal y operativa no negociable. No podemos permitirnos perder el registro de una transacción, duplicar saldos o tener "dinero fantasma" flotando en el sistema por un error de red. Para blindar nuestra aplicación, nos apoyamos en el estándar transaccional de las bases relacionales definido por el acrónimo **ACID**:

- **A - Atomicity (Atomicidad)**: La regla del "Todo o Nada". En una transferencia de fondos de la cuenta $A$ a la cuenta $B$, el sistema debe realizar dos operaciones: debitar de $A$ y acreditar en $B$. Si el servidor se apaga repentinamente justo después de debitar de $A$ pero antes de acreditar a $B$, la base de datos revertirá automáticamente el débito inicial (_Rollback_). Nunca habrá estados intermedios inconsistentes.
- **C - Consistency (Consistencia)**: La base de datos siempre debe transicionar de un estado válido a otro, respetando estrictamente las reglas e invariantes definidas (por ejemplo, el mandato $B \ge 0$, donde $B$ es el balance de una cuenta de ahorros). Si una operación intenta violar esta restricción matemática, la transacción entera es rechazada.
- **I - Isolation (Aislamiento)**: Define cómo y cuándo los cambios realizados por una operación son visibles para otras concurrentes. Si dos usuarios intentan retirar los últimos $\$100$ de una cuenta compartida exactamente en el mismo milisegundo, el aislamiento previene las _Race Conditions_ (Condiciones de Carrera). El motor encola o bloquea las transacciones para asegurar que solo el primero tenga éxito, protegiendo la integridad del saldo.
- **D - Durability (Durabilidad)**: Una vez que el sistema confirma al usuario que su transferencia fue exitosa (_Commit_), esa información está escrita permanentemente en el disco físico. Ni siquiera un corte de energía catastrófico en el centro de datos borrará ese registro.

### 2.5. Resolviendo la Herencia

Nuestro diagrama de clases UML (definido en el SRS) exige que las operaciones `Deposit`, `Withdrawal` y `Transfer` hereden atributos y comportamientos de `Transaction`.

```mermaid
classDiagram
  class Deposit {
    +UUID destinationAccountId
  }

  class Withdrawal {
    +UUID sourceAccountId
  }

  class Transfer {
    +UUID sourceAccountId
    +UUID destinationAccountId
  }

  class Transaction {
    <<abstract>>
    +UUID id
    +Decimal amount
    +TransactionStatus status
    +String description
    +Date createdAt
    +execute()
  }

    Transaction <|-- Deposit
    Transaction <|-- Withdrawal
    Transaction <|-- Transfer
```

En la arquitectura de bases de datos, tenemos dos enfoques principales para resolver esto:

1. **Class Table Inheritance (CTI)**: Crear una tabla por cada clase (`Tabla_Transaction`, `Tabla_Deposit`, etc.) y unirlas con _JOINs_. Esto es puro, pero costoso en rendimiento computacional.

    ```mermaid
    erDiagram
      TRANSACTION {
        UUID id PK
        Decimal amount
        Integer status
        String description
        Date createdAt
      }

      DEPOSIT {
        UUID transaction_id PK, FK
        UUID destinationAccountId
      }

      WITHDRAWAL {
        UUID transaction_id PK, FK
        UUID sourceAccountId
      }

      TRANSFER {
        UUID transaction_id PK, FK
        UUID sourceAccountId
        UUID destinationAccountId
      }

      TRANSACTION ||--|| DEPOSIT : is
      TRANSACTION ||--|| WITHDRAWAL : is
      TRANSACTION ||--|| TRANSFER : is
    ```

2. **Concrete Table Inheritance (TPC - Table-per-Concrete-Type)**: Se crea una tabla por cada clase concreta sin tabla base. Dificulta las consultas polimórficas globales (por ejemplo, listar todas las transacciones de un usuario independientemente del tipo).

    ```mermaid
    erDiagram
      DEPOSIT {
        UUID transaction_id PK
        Decimal amount
        Integer status
        String description
        Date createdAt
        UUID destinationAccountId
      }

      WITHDRAWAL {
        UUID transaction_id PK
        Decimal amount
        Integer status
        String description
        Date createdAt
        UUID sourceAccountId
      }

      TRANSFER {
        UUID transaction_id PK
        Decimal amount
        Integer status
        String description
        Date createdAt
        UUID sourceAccountId
        UUID destinationAccountId
      }
    ```

3. **Single Table Inheritance (STI)**: El enfoque que adoptaremos. Consiste en crear una única **tabla ancha** llamada `Transaction` en la base de datos, la cual contendrá la unión de todos los campos de todas las subclases.

    ```mermaid
    erDiagram
      TRANSACTION {
        UUID id PK
        Decimal amount
        String type "DEPOSIT, WITHDRAWAL or TRANSFER"
        Integer status
        String description
        Date createdAt
        String sourceAccountId "Opcional para Retiros y Transferencias"
        string destinationAccountId "Opcional para Depósitos y Transferencias"
      }
    ```

**Reglas de Implementación del Patrón STI**:

- **Columna Discriminadora (`type`)**: Se introduce una columna de tipo `ENUM` que almacena de forma explícita qué subclase concreta representa cada fila.
- **Campos Opcionales en DB (`NULLABLE`)**: Las columnas asociadas a subclases específicas (como `sourceAccountId` para retiros/transferencias y `destinationAccountId` para depósitos/transferencias) se declaran como opcionales (`?`) en la base de datos relacional.
- **Validación de Integridad en el Mapper**: La responsabilidad de garantizar que un `Deposit` contenga un `destinationAccountId` no nulo deja de ser de PostgreSQL y se traslada a la **Capa de Dominio** y a sus **Data Mappers**, garantizando consistencia estricta en tiempo de ejecución.

### 2.6. El Patrón Repository (Repositorio)

En _Clean Architecture_, el principio de Inversión de Dependencias (DIP) de SOLID dicta que las capas internas no deben depender de las externas.

¿Cómo guardamos un usuario en la base de datos sin importar Prisma en nuestro caso de uso? **A través del Patrón Repository**.

1. **El Puerto (Interfaz)**: El Dominio define una interfaz `AccountRepository` (ej. "Necesito un método `save(account: Account)`").
2. **El Adaptador (Implementación)**: La Infraestructura implementa esa interfaz en `PrismaAccountRepository`, utilizando Prisma por debajo.

De esta forma, si mañana cambiamos Prisma por TypeORM o raw SQL, la lógica de negocio permanece intacta.

## 3. DESARROLLO PRÁCTICO (90 min)

### Paso 0: Configuración de la base de datos (PostgreSQL) usando Docker Compose (Opcional)

1. Creamos el archivo `docker-compose.yml`:

    ```yml copy
    services:
      postgres:
        image: postgres:18
        environment:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: tu_password_seguro
          POSTGRES_DB: fintech_core
        ports:
          - "5432:5432"
        volumes:
          - postgres_data:/var/lib/postgresql/data

      volumes:
        postgres_data:
    ```

2. Levanta el contenedor de PostgreSQL en segundo plano:

    ```bash copy
    docker compose up -d
    ```

### Paso 1: Configuración de Infraestructura de Datos

Primero, instalaremos las dependencias necesarias. Asegúrate de estar en la raíz de tu proyecto Backend.

Instalamos la CLI de Prisma como dependencia de desarrollo

```bash copy
pnpm add -D prisma dotenv
```

Instalamos el Cliente de Prisma para ejecución

```bash copy
pnpm add @prisma/client
```

Inicializamos Prisma (Esto creará la carpeta prisma y el archivo .env)

```bash copy
pnpm exec prisma init
```

El comando `init` habrá creado un archivo `.env` en la raíz. Debemos configurarlo apuntando a nuestra instancia de PostgreSQL (ya sea instalada localmente o corriendo en un contenedor de Docker). Asegúrate de reemplazar las credenciales por las tuyas:

Configura tu archivo `.env` en la raíz del proyecto para apuntar a tu instancia local de PostgreSQL (puedes usar Docker si lo prefieres):

`.env`

```bash copy
DATABASE_URL="postgresql://postgres:tu_password_seguro@localhost:5432/fintech_core?schema=public"
```

### Paso 2: Modelado del Esquema (`schema.prisma`)

Ahora, traduciremos nuestro modelo conceptual (Dominio) al modelo físico de base de datos utilizando el DSL (Domain Specific Language) de Prisma.

Para cumplir estrictamente con el requisito de **Modelado Decimal Seguro**, es imperativo evitar los tipos de dato numéricos de coma flotante (`Float` o `Double`). Estos tipos sufren de imprecisión en los redondeos binarios, lo cual es inaceptable en un sistema financiero. Utilizaremos el tipo `Decimal` nativo de PostgreSQL, definiendo una precisión de 19 dígitos totales y 4 posiciones decimales.

Abre el archivo `prisma/schema.prisma` e introduce la siguiente estructura. Nota cómo implementamos el patrón STI en el modelo `Transaction`.

```prisma copy
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

// Enums para mantener integridad de dominio en la DB
enum AccountStatus {
  ACTIVE
  FROZEN
}

enum TransactionType {
  DEPOSIT
  WITHDRAWAL
  TRANSFER
}

enum TransactionStatus {
  PENDING
  COMPLETED
  FAILED
}

model User {
  id           String    @id @default(uuid()) @db.Uuid
  email        String    @unique
  passwordHash String
  fullName     String
  createdAt    DateTime  @default(now())
  
  accounts     Account[]
}

model Account {
  id            String        @id @default(uuid()) @db.Uuid
  accountNumber String        @unique
  balance       Decimal       @default(0.00) @db.Decimal(19, 4) // Prevención de errores de flotante
  status        AccountStatus @default(ACTIVE)
  createdAt     DateTime      @default(now())
  userId        String        @db.Uuid
  
  // Relación con el usuario propietario
  user          User          @relation(fields: [userId], references: [id], onDelete: Restrict)

  // Relaciones inversas hacia las transacciones
  sentTransactions     Transaction[] @relation("SourceAccount")
  receivedTransactions Transaction[] @relation("DestinationAccount")
}

// -------------------------------------------------------------------
// Implementación de Single Table Inheritance (STI)
// -------------------------------------------------------------------
model Transaction {
  id                   String            @id @default(uuid()) @db.Uuid
  amount               Decimal           @db.Decimal(19, 4)
  type                 TransactionType   // <-- Columna Discriminadora clave para STI
  status               TransactionStatus @default(PENDING)
  description          String?
  createdAt            DateTime          @default(now())

  // Campos específicos de Withdrawal / Transfer (Deben ser opcionales en DB)
  sourceAccountId      String?           @db.Uuid
  sourceAccount        Account?          @relation("SourceAccount", fields: [sourceAccountId], references: [id])
  
  // Campos específicos de Deposit / Transfer (Deben ser opcionales en DB)
  destinationAccountId String?           @db.Uuid
  destinationAccount   Account?          @relation("DestinationAccount", fields: [destinationAccountId], references: [id])
}
```

Para aplicar este esquema a tu base de datos física, ejecuta la primera migración. Esto generará las tablas y actualizará el cliente de TypeScript:

```bash copy
npx prisma migrate dev --name init_fintech_schema
```

Ahora, generar el cliente Prisma:

```bash copy
npx prisma generate
```

### Paso 3: Data Mappers y Resolución del Polimorfismo

El **Mapper** (Mapeador) es una pieza de software vital donde ocurre la verdadera magia arquitectónica. Su responsabilidad única es traducir la representación plana de la base de datos (Data Transfer Objects de Prisma) en ricas Entidades de Dominio, y viceversa. Actúan como traductores bilingües en las fronteras de nuestro sistema.

Primero, crearemos un mapper básico para las Cuentas. Crea el archivo `src/infrastructure/mappers/AccountMapper.ts`:

```typescript copy
import { Decimal } from 'decimal.js';
import { Account as PrismaAccount, Prisma } from '../../generated/prisma/client';
import { Account } from '../../domain/entities/Account';

export class AccountMapper {
  /**
   * Convierte un registro de infraestructura a una entidad de dominio puro
   */
  public static toDomain(prismaAccount: PrismaAccount): Account {
    return new Account({
      id: prismaAccount.id,
      accountNumber: prismaAccount.accountNumber,
      balance: new Decimal(prismaAccount.balance), // Transformación segura de Decimal a valor manejable en TS
      userId: prismaAccount.userId,
      status: prismaAccount.status,
      createdAt: prismaAccount.createdAt
    });
  }

  /**
   * Convierte una entidad de dominio a la estructura requerida por Prisma
   */
  public static toPersistence(account: Account): Prisma.AccountUncheckedCreateInput {
    return {
      id: account.id,
      accountNumber: account.accountNumber,
      balance: account.balance,
      userId: account.userId,
      status: account.status,
      createdAt: account.createdAt,
    };
  }
}
```

Ahora, resolveremos la complejidad de la herencia. Crea el archivo `src/infrastructure/mappers/TransactionMapper.ts`. Nota cómo el método `toDomain` evalúa la columna discriminadora `type` para instanciar dinámicamente la subclase correcta, restaurando el polimorfismo que habíamos perdido en la base de datos relacional.

```typescript copy
import { Decimal } from 'decimal.js';
import { Transaction as PrismaTransaction, TransactionType, Prisma } from '../../generated/prisma/client';
import { Transaction, Deposit, Withdrawal, Transfer} from '../../domain/entities/Transaction';

export class TransactionMapper {
  
  public static toDomain(prismaTx: PrismaTransaction): Transaction {
    const amount = new Decimal(prismaTx.amount.toNumber());
    const txProps = {
      id: prismaTx.id,
      amount,
      status: prismaTx.status,
      description: prismaTx.description ?? '',
      createdAt: prismaTx.createdAt
    };

    // Reconstrucción polimórfica basada en el discriminador STI
    switch (prismaTx.type) {
      case TransactionType.DEPOSIT:
        if (!prismaTx.destinationAccountId) throw new Error("Inconsistencia en DB: Deposit requiere destinationAccountId");
        return new Deposit(txProps, prismaTx.destinationAccountId);

      case TransactionType.WITHDRAWAL:
        if (!prismaTx.sourceAccountId) throw new Error("Inconsistencia en DB: Withdrawal requiere sourceAccountId");
        return new Withdrawal(txProps, prismaTx.sourceAccountId);

      case TransactionType.TRANSFER:
        if (!prismaTx.sourceAccountId || !prismaTx.destinationAccountId) {
            throw new Error("Inconsistencia en DB: Transfer requiere ambas cuentas (origen y destino)");
        }
        return new Transfer(txProps, prismaTx.sourceAccountId, prismaTx.destinationAccountId);

      default:
        throw new Error(`Tipo de transacción desconocido o corrupto en DB: ${prismaTx.type}`);
    }
  }

  public static toPersistence(transaction: Transaction): Prisma.TransactionUncheckedCreateInput {
    let type: TransactionType;
    let sourceAccountId: string | null = null;
    let destinationAccountId: string | null = null;

    // Dependiendo de la instancia en memoria, determinamos la forma del registro plano
    if (transaction instanceof Deposit) {
      type = TransactionType.DEPOSIT;
      destinationAccountId = transaction.destinationAccount;
    } else if (transaction instanceof Withdrawal) {
      type = TransactionType.WITHDRAWAL;
      sourceAccountId = transaction.sourceAccount;
    } else if (transaction instanceof Transfer) {
      type = TransactionType.TRANSFER;
      sourceAccountId = transaction.sourceAccount;
      destinationAccountId = transaction.destinationAccount;
    } else {
      throw new Error("Instancia de transacción inválida proporcionada al Mapper");
    }

    return {
      id: transaction.id,
      amount: transaction.amount,
      type,
      status: transaction.status,
      sourceAccountId,
      destinationAccountId,
      createdAt: transaction.createdAt,
    };
  }
}
```

### Paso 4: Implementación de Repositorios (Adaptadores)

Finalmente, implementamos la interfaz definida en nuestro dominio utilizando nuestro ORM y los Mappers recién construidos. Esta clase será el punto de acceso exclusivo a la tabla `Account`.

Crea `src/infrastructure/repositories/PrismaAccountRepository.ts`:

```typescript copy
import { PrismaClient } from '@prisma/client';
import { AccountRepository } from '../../domain/repositories/AccountRepository';
import { Account } from '../../domain/entities/Account';
import { AccountMapper } from '../mappers/AccountMapper';

export class PrismaAccountRepository implements AccountRepository {
  // Inyección de dependencias a través del constructor
  constructor(private readonly prisma: PrismaClient) {}

  async save(account: Account): Promise<void> {
    const data = AccountMapper.toPersistence(account);

    // Utilizamos 'upsert' que funciona como "Insertar si no existe, Actualizar si ya existe"
    // Esto centraliza la lógica de persistencia en un solo método robusto.
    await this.prisma.account.upsert({
      where: { id: account.id },
      update: { 
        balance: data.balance, 
        status: data.status 
      },
      create: data,
    });
  }

  async findById(id: string): Promise<Account | null> {
    const prismaAccount = await this.prisma.account.findUnique({
      where: { id }
    });

    if (!prismaAccount) return null;
    
    // Inmediatamente traducimos la respuesta de Prisma a nuestro lenguaje de Dominio
    return AccountMapper.toDomain(prismaAccount);
  }
}
```

## 4. PUNTOS DE CONTROL ARQUITECTÓNICOS Y MEJORES PRÁCTICAS

- **Inyección de Dependencias (IoC)**: Nota que el repositorio `PrismaAccountRepository` no instancia su propia conexión a la base de datos (`new PrismaClient()`). En cambio, la recibe a través del constructor. Esto es crucial para la **Testabilidad**. Cuando escribamos pruebas unitarias, podremos inyectar una versión _Mock_ (falsa) de PrismaClient, probando nuestra lógica sin tocar una base de datos real.
- **Cumplimiento de Fronteras (DoD #1)**: Revisa el código del repositorio. El método `findById` retorna una instancia puramente estructurada bajo las reglas del dominio (`Account`). Si el Caso de Uso invoca este método, jamás se enterará de que los datos provinieron de PostgreSQL o fueron procesados por Prisma. La abstracción es total y hermética.
- **Encapsulamiento del Polimorfismo**: Hemos superado el _Impedance Mismatch_. Aunque la base de datos es plana e ignora las reglas de diseño orientado a objetos (por usar STI), nuestro sistema sigue siendo rico, seguro y respeta las jerarquías de herencia del dominio.

## 5. TAREAS Y ENTREGABLE DE LA SESIÓN

1. **Completar la Capa de Mappers**: Basándote en el `AccountMapper`, construye el archivo faltante `UserMapper.ts` asegurando un retorno tipado con `Prisma.UserUncheckedCreateInput`.
2. **Completar la Capa de Repositorios**: Implementa las clases `PrismaUserRepository` y `PrismaTransactionRepository`. Para este último, utiliza `TransactionMapper` y define los métodos para guardar transacciones y listar el historial de una cuenta.
3. **Gestión de Versiones (Git Flow)**: Realiza un commit con un mensaje semántico: `feat: implement prisma infrastructure with STI and strictly typed mappers` y súbelo a tu rama de desarrollo `step-02-prisma`.

---

Ahora que nuestras entidades tienen la capacidad de sobrevivir en el tiempo gracias al disco duro, en la próxima sesión orquestaremos el comportamiento del sistema.

Diseñaremos los **Casos de Uso** e implementaremos la lógica más crítica y peligrosa de una aplicación financiera: la transferencia atómica de fondos.

Aprenderemos a agrupar operaciones utilizando el método `$transaction` de Prisma para proteger férreamente las invariantes matemáticas (como evitar que se cree dinero de la nada si falla una conexión a mitad de proceso).
