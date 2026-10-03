# Arquitectura — Simple Stock Flow

**Cómo están dispuestas las piezas y por dónde pasa cada cosa.** Las decisiones que sostienen esta
disposición viven en `plan.md` y en `adr/`; aquí está la forma, no el porqué.

- **Fecha:** 2026-10-02
- **Rige bajo:** `constitution.md`. Cada regla de dependencia de este documento es un artículo de
  esa constitución, y se verifica igual.
- **Estado del sistema:** vive solo en la tabla de progreso de [`tasks.md`](tasks.md). Este documento
  describe la forma que se va a construir.

---

## 1. Topología: tres unidades desplegables

```
                    ┌─────────────────────────────┐
   navegador ──────►│  simple-stock-flow-app      │  React (SPA) · hexágono propio
                    │  nginx sirve el build        │  único puerto publicado: 8080
                    └──────────────┬──────────────┘
                                   │ /api y /media por proxy · JSON · token en cabecera
                                   ▼
                    ┌─────────────────────────────┐
                    │  simple-stock-flow-api      │  Python 3.12 · FastAPI · hexagonal
                    │  ┌───────────────────────┐  │  puerto interno 8000
                    │  │ adaptadores entrantes │  │  REST (el único)
                    │  ├───────────────────────┤  │
                    │  │ aplicación + dominio  │  │  NÚCLEO
                    │  ├───────────────────────┤  │
                    │  │ adaptadores salientes │  │  persistencia · seguridad · binarios
                    │  └───────────────────────┘  │
                    └────────┬───────────┬────────┘
                             ▼           ▼
              ┌──────────────────┐  ┌──────────────────┐
              │ MySQL 8.4 (db)   │  │ volumen media    │
              │ lo levanta       │  │ sustituible por  │
              │ simple-stock-    │  │ almac. objetos   │
              │ flow-infra       │  │                  │
              └──────────────────┘  └──────────────────┘
```

| Repositorio | Responsable de | **No** responsable de |
|---|---|---|
| `simple-stock-flow-infra` | Contenedor del motor, red, volúmenes, orquestación del conjunto (compose) | **El esquema.** Ni una línea de DDL |
| `simple-stock-flow-api` | Reglas de negocio, API **y el esquema completo** vía migraciones (Alembic) | La orquestación del despliegue |
| `simple-stock-flow-app` | Interfaz y sus propios casos de uso | El almacenamiento. Solo habla con la API |

La propiedad del esquema es la decisión estructural del conjunto: **ADR-001**. Los otros tres
repositorios (`docs`, `page`, `tool`) no se despliegan; ver §8.

---

## 2. El hexágono del servicio

```
     adaptador entrante                             adaptadores salientes
   (quién invoca al núcleo)                    (de quién depende el núcleo)

                                                    ┌───── persistencia (SQLAlchemy + PyMySQL)
   REST ──► puertos entrantes ──► núcleo ──► puertos salientes ─┼───── binarios
            PlaceSale             dominio +   ProductRepository └───── seguridad (JWT, hash)
            ManageProducts        aplicación  FileStorage
            GetSalesReport                    Clock
```

Disposición del paquete raíz de la API:

```
src/stockflow/
├─ domain/                      entidades, value objects, políticas — solo stdlib
├─ application/
│  ├─ ports/inbound/            PlaceSale · ManageProducts · GetSales · GetSalesReport · Authenticate
│  ├─ ports/outbound/           repositorios · FileStorage · PasswordHasher · TokenGenerator · Clock · UnitOfWork · SalesReportQuery
│  └─ services/                 PlaceSaleService, etc.
├─ adapters/
│  ├─ inbound/api/              FastAPI
│  └─ outbound/{persistence,storage,security}/
└─ bootstrap/composition/port_bindings.py
```

**REST es el único adaptador entrante**, y `adapters/inbound/` contiene una sola carpeta, `api/`.
No se crean adaptadores de mensajería ni de planificación que declaren una capacidad que el sistema
no tiene (artículo X): el enunciado no los pide.

Lo que esa forma garantiza es la regla: **un adaptador entrante nuevo no llamaría a persistencia,
sino al mismo puerto entrante que usa REST.** Es lo que hace del puerto una frontera real y no un
nombre — pero hasta que exista un segundo adaptador es una propiedad de diseño, no un hecho
observable.

Cómo se aplica SOLID aquí, principio por principio y con su verificación, está en el
**artículo IV** de [`constitution.md`](constitution.md). No se repite.

### 2.1 La app repite el hexágono

```
src/
├─ domain/            modelos, value objects, políticas — sin React
├─ application/       ports/ · use-cases/ · state/
├─ infrastructure/    http/ · mappers/ · interceptors/ — implementan los puertos
└─ features/          pantallas por dominio funcional
```

Sus casos de uso no conocen el cliente HTTP. Cambiar el transporte es sustituir una clase de
`infrastructure/`. No es simetría decorativa: es lo que permite probar la política de carrito sin
levantar un servidor.

---

## 3. Interfaces

### 3.1 API

> **La forma exacta de cada petición y de cada respuesta está en
> [`api-contract.md`](api-contract.md), y solo ahí.** Este apartado se queda con lo que es
> arquitectura —qué endpoints hay, quién puede llamarlos, dónde se traduce un error y por qué no
> hay escritura de categorías—. Códigos, cuerpos, campos, tipos, nulabilidad, valores por omisión
> y orden de las filas **no se repiten aquí**: duplicarlos crearía la segunda fuente de verdad que
> el artículo XII prohíbe.

| Método | Ruta | Autorización | Caso de uso |
|---|---|---|---|
| `POST` | `/api/auth/login` | anónimo | autenticar |
| `POST` | `/api/auth/register` | **admin** | alta de usuario |
| `GET` | `/api/products` | autenticado | listar catálogo paginado |
| `GET` | `/api/products/{id}` | autenticado | obtener producto |
| `POST` `PUT` `DELETE` | `/api/products[/{id}]` | **admin** | mantener catálogo |
| `POST` | `/api/products/{id}/image` | **admin** | adjuntar imagen |
| `GET` | `/api/categories` | autenticado | listar categorías de referencia |
| `POST` | `/api/sales` | autenticado | registrar venta |
| `GET` | `/api/sales[/{id}]` | autenticado | consultar ventas |
| `GET` | `/api/reports/sales?from=&to=` | autenticado | reporte por rango |
| `GET` | `/media/{key}` | **anónimo** | servir la imagen de un producto |
| `GET` | `/health` | anónimo | — |

**La autorización de la tabla es un requisito, no una descripción.** En particular,
`POST /api/auth/register` exige rol `admin`: sin token válido de administrador no se crea ningún
usuario. Se verifica con un test de integración por cada fila protegida (401 sin token, 403 con rol
insuficiente).

**`GET /media/{key}` es público y sin token, por diseño.** No lo sirve un endpoint enrutado: lo
sirve `StaticFiles` de FastAPI, montado sobre el volumen de binarios, bajo la misma URL que
`FileStorage` publica dentro de `ProductView`. Al no pasar por las dependencias de seguridad,
**no hay forma de protegerlo sin cambiar el mecanismo**. Las claves son UUID de 32 caracteres sin
relación con el nombre original, luego no se pueden adivinar ni enumerar, pero una URL filtrada sirve
la imagen a cualquiera para siempre. Es lo que hace falta para que la etiqueta `<img>` de la app
funcione sin cabeceras, y queda como **riesgo aceptado con dueño** en
[`api-contract.md`](api-contract.md) §5, `H-6`. Su ficha, incluido el requisito de que el proxy de
nginx reenvíe también las extensiones de imagen (`.jpg`, `.png`) al servicio, está en
[`api-contract.md`](api-contract.md) E-15.

**Traducción de errores, en un único exception handler del adaptador REST:**

| Origen | Respuesta | Motivo |
|---|---|---|
| Violación de invariante de negocio | **422** | La petición estaba bien formada; la regla es lo que falló |
| Contrato mal formado | 400 | FastAPI responde 422 por defecto a la validación de forma: se reasigna a 400 con un handler propio, y **422 queda reservado a invariantes de negocio** |
| Sin token o token inválido | 401 | Autenticación |
| Rol insuficiente | **403** | Autorización, distinguible de 401 (CA-07.4) |
| Recurso inexistente | 404 | — |
| Conflicto de concurrencia | **409** | Agotados los reintentos de ADR-002 |

> El handler es **el único sitio** donde una excepción de dominio se convierte en código HTTP. Si
> aparece un `try/except` de negocio en una ruta, esa traducción ya está en dos sitios.

**El cuerpo del error tiene más de una forma, y el contrato debe reconciliarlas.** El handler propio
produce una; las que pone el framework (404 de ruta, 405, 401 de seguridad) llegan como
`{"detail": ...}`. [`api-contract.md`](api-contract.md) §2 fija el cuerpo único que ve el cliente y
cómo se adaptan las demás, de modo que la app pueda mostrar el campo que falló y no un mensaje
genérico.

**Categorías: lectura, y nada más.** `GET /api/categories` devuelve las categorías de referencia
**completas y sin paginar** —son datos semilla de la migración `seed_categories`, no un catálogo que crezca
(D-10)—. Su ficha completa, con el orden de las filas como parte del contrato, está en
[`api-contract.md`](api-contract.md) E-09.

**No hay `POST`, `PUT` ni `DELETE` sobre categorías, y no es un olvido.** El repositorio de
categorías es de solo lectura por decisión de D-10, y la línea de venta congela el nombre de la
categoría (ADR-004). Permitir renombrarlas obligaría a responder qué pasa con los nombres ya
congelados — una pregunta que el enunciado no plantea porque no pide mantenerlas.

**No añade un puerto.** El método entra en el puerto de catálogo —el mismo que mantiene
productos— y se resuelve con `CategoryRepository.list_all`. Hace falta el método del puerto
entrante y la ruta que lo expone: el recuento de §3.2 no cambia.

**Dónde vive la forma exacta de cada respuesta.** En [`api-contract.md`](api-contract.md), una ficha
por endpoint. Su **fuente ejecutable** son los dataclasses de `application/ports/inbound/`
—`ProductView`, `SaleView`, `SaleItemView`, `SalesReport`, `SalesReportRow`, `AuthResult`,
`PagedResult`—, tipados y verificados con `mypy --strict`, que deben coincidir campo a campo con
`src/infrastructure/http/dto/api.dto.ts` de la app, el único consumidor. El contrato **no los
copia**: declara lo que los tipos no pueden declarar por sí solos —obligatoriedad, valores por
omisión, nulabilidad, códigos de error, orden de las filas— y cita el tipo como origen. La
documentación OpenAPI autogenerada (`/docs`) existe, pero **`api-contract.md` es la fuente de
verdad**.

### 3.1.1 Los huecos del contrato: decididos antes de construir

Las decisiones de comportamiento que la forma de los DTO no resuelve están decididas, con su motivo
y su consecuencia, en [`api-contract.md`](api-contract.md) §3 —`D-C1` a `D-C8`—, junto con el cuerpo
del 400, la moneda del reporte vacío y la fragilidad de `totalPages`. Ninguna queda abierta.

Lo que el contrato mantiene como **tareas sin dueño** son los huecos nombrados uno a uno en
[`api-contract.md`](api-contract.md) §5, `H-1` a `H-6`, cada uno con su criterio de cierre. El
progreso de cada una vive en [`tasks.md`](tasks.md).

### 3.2 Puertos

**Entrantes (5):** `PlaceSale` · `ManageProducts` · `GetSales` · `GetSalesReport` · `Authenticate`.

**Salientes (10):** `ProductRepository`, `SaleRepository`, `CategoryRepository` y `UserRepository` ·
`FileStorage` · `PasswordHasher` · `TokenGenerator` · `Clock` · `UnitOfWork` · **el puerto de lectura
del reporte**, `SalesReportQuery`, que implementa el adaptador de persistencia con una consulta
agregada (T-08).

Los puertos son `typing.Protocol` (o ABC) **sin prefijo `I`**. Ningún puerto habla el lenguaje de su
adaptador: no aparece en sus firmas ningún tipo del ORM, del servidor web ni del sistema de
archivos. Es el artículo II de la constitución, y se comprueba con una búsqueda de texto y con
`import-linter`.

---

## 4. Flujos

### 4.1 Registrar una venta — donde se concentra el riesgo

```
REST ─► puerto de venta ─► servicio de venta
                              │
                              ├─ cargar productos por lote      ◄── filtra bajas (ADR-003)
                              ├─ cargar categorías               ◄── para congelar (ADR-004)
                              │
                              ├─ abrir venta con el reloj inyectado
                              ├─ por cada línea: añadir item
                              │      └─ retirar del stock  ◄── invariante stock ≥ 0
                              ├─ exigir al menos una línea
                              │
                              └─ confirmar unidad de trabajo
                                     ├─ éxito
                                     └─ conflicto de versión ─► reintento del caso de uso
                                                                completo (máx. 3) ─► 409
```

**Tres propiedades que este flujo debe garantizar**, y que los tests deben ejercer por separado:

1. **Atomicidad** — el descuento y la línea se confirman juntos o ninguno (CA-04.2).
2. **Aislamiento** — dos ventas simultáneas del último ejemplar no dejan stock negativo (CA-04.6).
3. **Congelación** — nombre, precio y categoría de la línea no siguen al catálogo (CA-04.7).

El conflicto de versión lo detecta el adaptador de persistencia (columna `version`, ADR-002), lo
traduce a un `ConcurrencyConflict` definido por la aplicación, y ningún tipo de SQLAlchemy cruza el
puerto.

### 4.2 Reporte por rango

```
REST ─► puerto del reporte ─► servicio ─► puerto de lectura ─► agregación en el motor
                                                                agrupa por producto
                                                                suma unidades e importe
```

El servicio **no carga ventas**: recibe filas ya agregadas. Cargar el rango en memoria para sumarlo
es el defecto que CA-06.5 declara bloqueante.

### 4.3 Arranque

```
docker compose up
   ├─ db         MySQL 8.4: crea base `stockflow` vacía y usuario; healthcheck con `mysqladmin ping`
   ├─ api        espera a que el motor responda (depends_on: service_healthy) y además reintenta
   │               ├─ exige las variables de entorno obligatorias; si faltan, falla con error explícito
   │               ├─ aplica migraciones pendientes (Alembic, GET_LOCK) ◄── `initial_schema` crea el esquema (CA-08.2)
   │               │     y `seed_categories` siembra las cinco categorías: sin ellas no se crea un producto
   │               ├─ crea el administrador inicial si no existe, con credenciales de entorno
   │               └─ escucha en :8000
   └─ app        nginx sirve el build y hace de proxy de /api y /media hacia api:8000
```

**"Espera a que el motor responda"** no es un detalle: un orden de arranque solo garantiza que el
contenedor exista, no que la base acepte conexiones. Es la causa más común de un primer arranque
que falla y un segundo que funciona.

---

## 5. Dónde se satisface cada requisito no funcional

| Requisito | Mecanismo | Dónde vive |
|---|---|---|
| **RNF-01** integridad bajo concurrencia | Control optimista con reintento acotado | Adaptador de persistencia — ADR-002 |
| **RNF-02** fidelidad histórica | Atributos congelados en la línea; sin operación de edición de venta | Dominio — ADR-004 |
| **RNF-03** paginación | Tamaño de página acotado, aplicado en el repositorio | Aplicación |
| **RNF-04** arranque reproducible | Migraciones y semilla en el arranque del servicio | Arranque — ADR-001 |
| **RNF-05** trazabilidad | Registro estructurado con correlación | Adaptador REST |
| **RNF-06** binarios intercambiables | Puerto con clave opaca; el dominio nunca ve rutas ni bytes | Aplicación |
| **RNF-07** pruebas en tres niveles | Dominio sin dobles · aplicación con fakes · integración contra MySQL real (Testcontainers) | `tests/` |

---

## 6. Riesgos arquitectónicos abiertos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Migraciones aplicadas en el arranque con varias réplicas | Carrera entre instancias | `GET_LOCK` de MySQL para serializar el arranque (ADR-001). Con muchas réplicas, mover la migración a un paso previo del despliegue |
| **El DDL de MySQL no es transaccional** (commit implícito) | Una migración que falla a la mitad deja el esquema parcial | Migraciones pequeñas, una sola operación de esquema por revisión; restaurar el volumen en desarrollo |
| El filtro de bajas se olvida en la carga previa a vender | **Se vende un producto retirado** | Filtro centralizado en la base de los repositorios: lo seguro por omisión (ADR-003). Test de integración (T-09) |
| `GET /media/{key}` es público | Una URL filtrada sirve la imagen a cualquiera | **Aceptado** por diseño (§3.1, `H-6`): claves UUID no enumerables, imágenes de catálogo sin dato personal |
| La colación `utf8mb4_0900_ai_ci` ignora mayúsculas **y acentos** | "Café" y "cafe" chocan en una restricción de unicidad y coinciden en una búsqueda | Declarado y decidido en [`data-model.md`](data-model.md); se prueba en integración |

---

## 7. Decisiones registradas

| ADR | Decisión |
|---|---|
| [**ADR-001**](adr/adr-001-propiedad-del-esquema.md) | El esquema pertenece al servicio; la infraestructura entrega un motor vacío |
| [**ADR-002**](adr/adr-002-concurrencia-optimista.md) | Concurrencia optimista con una columna `version` que solo existe en el modelo de persistencia |
| [**ADR-003**](adr/adr-003-baja-logica.md) | Baja lógica con `deleted_at` en el modelo de persistencia y filtro centralizado en los repositorios |
| [**ADR-004**](adr/adr-004-reporte-agregado-y-congelado.md) | El reporte agrega en el motor sobre atributos congelados |

---

## 8. Cómo se clonan y se levantan los seis repositorios

`simple-stock-flow-infra` se explica a sí mismo (artículo XIII): su README dice cómo se levanta, qué
variable hace qué y dónde están los datos. Aquí queda solo la decisión estructural que ese
repositorio no puede tomar por su cuenta, y los dos límites que nunca debe cruzar.

El conjunto son **seis directorios hermanos, y solo tres se despliegan**:

| Repositorio | Qué es | ¿Se despliega? |
|---|---|---|
| `simple-stock-flow-docs` | Documentación: `enunciado/`, `spec/`, `diagramas/` | no |
| `simple-stock-flow-api` | Backend Python hexagonal | sí (`api`) |
| `simple-stock-flow-app` | Frontend React; nginx sirve el build y hace proxy | sí (`app`) |
| `simple-stock-flow-page` | Sitio público estático de presentación; no habla con la API | no |
| `simple-stock-flow-infra` | docker-compose, red, volúmenes, motor MySQL vacío, `.env.example`, `verify.sh` | es el orquestador |
| `simple-stock-flow-tool` | Utilidades Python: sembrador de datos de demostración (usa la API) e imágenes de demo | no |

`infra`, `api` y `app` levantan el sistema; `docs`, `page` y `tool` no entran en el compose ni hacen
falta para arrancar. `tool` existe como repositorio aparte para que `infra` no necesite Python:
la promesa de `infra` es que basta con Docker.

**Los tres repositorios desplegables se clonan como hermanos**, y el compose apunta a `../` para
construir (`../simple-stock-flow-api`, `../simple-stock-flow-app`). Clonar uno por separado, o
anidarlo, deja la construcción sin contexto. Convertirlos en submódulos de `infra` es una **mejora
futura opcional** (T-17): aportaría el clonado único que el enunciado sugiere al hablar de
*"repositorio clonable y ejecutable"* en singular, a costa de exigir `--recurse-submodules` y un
commit concreto por repositorio.

**Lo que `infra` no lleva nunca:**

- **Ni una línea de DDL, ni datos semilla.** El esquema es del servicio (ADR-001). Un script de
  inicialización del contenedor además solo se ejecuta con el volumen vacío: la segunda versión
  del esquema no se aplicaría jamás, y el fallo sería silencioso.
- **Ningún secreto real.** Un `.env.example` con todas las claves y sin valores; el `.env` real,
  ignorado por git. Tampoco el puerto de la base (3306) publicado al host por defecto: se publica
  con `docker-compose.dev.yml` (perfil dev), para inspeccionar con un cliente cuando haga falta.

**Configuración que consume la API**, por variables de entorno y **sin valor por defecto para
secretos**: `DATABASE_URL`, `JWT_SIGNING_KEY`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`, `MEDIA_ROOT`. Si
falta alguna, el arranque falla con un error que la nombra.
