# Tareas — Simple Stock Flow

Unidades ejecutables derivadas de `plan.md`, ordenadas por dependencia real. Cada una trae
**cómo se sabe que está hecha**, y ese criterio es una comprobación, no una opinión.

**El estado del sistema vive solo aquí, en la tabla de progreso.** Una tarea pasa a ✅ únicamente
con su evidencia escrita en su fila: el comando ejecutado y lo que devolvió.

**Repos (6):** `docs`, `api`, `app`, `page`, `infra`, `tool`, todos con prefijo `simple-stock-flow-`.
Se despliegan solo `infra`, `api` y `app`; se clonan como hermanos y el compose construye con `../`.

**Premisa de las tareas del `api`.** T-01 crea el esqueleto del repo (`pyproject.toml`, ruff,
mypy `--strict`, import-linter, pytest) y el dominio puro de `src/stockflow/domain/`. Toda tarea
del `api` depende de T-01 aunque no lo repita. Los comandos de comprobación son siempre los del
nuevo stack: `pytest`, `alembic`, `ruff check`, `mypy`, `lint-imports` (import-linter), `vitest`,
`docker compose` y `verify.sh`.

---

## Progreso

**0 de 25 hechas (0 %).** El 25 sale de contar las filas de esta tabla; `verify.sh` (repo `infra`)
recompone la cifra contando las filas de la tabla y las marcadas ✅, y falla si lo declarado en
esta línea no coincide (artículo X.1). La columna de evidencia está vacía a propósito: ninguna
tarea se ha iniciado.

| Tarea | Repo | Estado | Evidencia |
|---|---|---|---|
| **T-01** | api | ⬜ Pendiente | — |
| **T-02** | api | ⬜ Pendiente | — |
| **T-03** | infra | ⬜ Pendiente | — |
| **T-04** | api | ⬜ Pendiente | — |
| **T-05** | api | ⬜ Pendiente | — |
| **T-06** | api | ⬜ Pendiente | — |
| **T-07** | api | ⬜ Pendiente | — |
| **T-08** | api | ⬜ Pendiente | — |
| **T-09** | api | ⬜ Pendiente | — |
| **T-10** | api | ⬜ Pendiente | — |
| **T-11** | api | ⬜ Pendiente | — |
| **T-12** | api | ⬜ Pendiente | — |
| **T-13** | api | ⬜ Pendiente | — |
| **T-14** | api | ⬜ Pendiente | — |
| **T-15** | app | ⬜ Pendiente | — |
| **T-16** | api · app · infra | ⬜ Pendiente | — |
| **T-17** | los seis | ⬜ Pendiente | — |
| **T-18** | infra | ⬜ Pendiente | — |
| **T-19** | api · app · tool | ⬜ Pendiente | — |
| **T-20** | api | ⬜ Pendiente | — |
| **T-21** | api | ⬜ Pendiente | — |
| **T-22** | api | ⬜ Pendiente | — |
| **T-23** | page | ⬜ Pendiente | — |
| **T-24** | tool | ⬜ Pendiente | — |
| **T-25** | docs | ⬜ Pendiente | — |

**Cobertura de repos:** `api` T-01, T-02, T-04…T-14, T-20…T-22 · `app` T-15 · `infra` T-03, T-18 ·
`page` T-23 · `tool` T-24 · `docs` T-25. T-16, T-17 y T-19 cruzan varios. T-23, T-24 y T-25 existen
solo para que cada repo tenga al menos una tarea propia.

**Las cuatro reglas de negocio con tarea propia:** RN-03 y RN-11 son **T-21**; RN-07 es **T-22**;
RN-08 es **T-09**, con la barrera del motor en **T-20**. La trazabilidad regla por regla —quién la
hace cumplir, qué capa de prueba la cubre, qué criterio la ejerce— vive en [`spec.md` §4](spec.md),
y es el único sitio donde vive.

---

## Fase 0 — Cimientos. Bloquean todo lo demás

### T-01 · Probar que los value objects se materializan

**Antes que cualquier repositorio.** Esqueleto del repo `api` (ver premisa), dominio puro
(`Money`, `Quantity`, `Product`, `Sale`, `SaleItem`, `Category`, `User`) y un test de integración
contra MySQL real (Testcontainers) que persiste y relee un producto y una venta a través del
**mapper SQLAlchemy ↔ dominio**.

- **Por qué primero:** los value objects validan en su constructor (`dataclass(frozen=True)`,
  `decimal.Decimal`) y el modelo de persistencia es otro tipo, no el del dominio. Si el mapper
  materializa mal —un `float` donde iba `Decimal`, un `datetime` sin zona donde iba UTC, un
  `Quantity` que se salta el constructor—, falla toda la persistencia, y descubrirlo después de
  escribir cuatro repositorios cuesta cuatro veces más (D-07).
- **Qué se afirma:** el importe `DECIMAL(12,2)` vuelve como `Decimal` exacto; la cantidad vuelve
  como `Quantity`; el precio y el nombre congelados de la línea vuelven iguales; `sold_at` se guarda
  en `DATETIME(6)` como UTC y vuelve como `datetime` con tz UTC; la columna `version` **no** aparece
  en la entidad de dominio. En esta tarea las tablas salen de `metadata.create_all`; T-02 las
  sustituye por la migración.
- **Hecho cuando:**
  - `pytest tests/integration -k materializ` pasa contra el contenedor, y **falla** si se cambia el
    mapper para que devuelva `float` o un `datetime` sin zona.
  - `ruff check`, `mypy --strict` y `lint-imports` pasan sobre el esqueleto.
  - `tests/architecture/` contiene el test que recorre los `import` de `src/stockflow/domain/` y
    falla ante cualquier módulo fuera de la stdlib (artículo I).

### T-02 · Esquema inicial y categorías semilla

Dos revisiones de Alembic en `src/stockflow/adapters/outbound/persistence/migrations/`, y **solo
dos** ([`data-model.md` §3.2](data-model.md#32-migraciones-previstas)):

| Revisión (mensaje) | Qué hace |
|---|---|
| `initial_schema` | Autogenerada desde el modelo de persistencia (`alembic revision --autogenerate -m "initial_schema"`) y **completada a mano**. Crea las cinco tablas (`category`, `product`, `sale`, `sale_item`, `user`, en singular), sus **25 columnas**, claves primarias, claves foráneas, únicas, los **13 índices** y los **nueve `CHECK`**. Los nombres son los de [`data-model.md` §3.1](data-model.md#31-convención-de-nombres-de-restricciones-e-índices) |
| `seed_categories` | **Solo datos:** las cinco categorías con los identificadores literales de [`data-model.md` §9.1](data-model.md#91-las-cinco-categorías-van-en-una-migración-de-semilla). Separada del esquema porque el DDL de MySQL no es transaccional |

**Los `CHECK` se escriben a mano.** `alembic revision --autogenerate` no compara restricciones
`CHECK`: aunque el modelo las declare con nombre, la revisión autogenerada puede omitirlas. Se
añaden con `op.create_check_constraint` y quien revisa **comprueba que cada `ck_*` de
`data-model.md` §4 aparezca**. La `MetaData` lleva un `naming_convention` para `idx_`, `uq_`,
`fk_` y `ck_`; los nombres que la convención no produce por sí sola (`uq_sale_item_sale_product`,
`idx_product_category_active_name`, `idx_product_active_name`) se declaran explícitos en el modelo.
La clave primaria no lleva nombre: en InnoDB es siempre `PRIMARY`.

- **Depende de:** T-01.
- **Por qué las categorías no son opcionales:** el repositorio de categorías es de solo lectura y
  la categoría del producto es obligatoria. **Sin semilla no se puede crear ni un producto** y el
  CRUD del enunciado no se puede ni ejercer (CA-08.3).
- **Qué trae el esquema desde el principio**, porque las demás tareas lo usan y no lo añaden:
  `product.deleted_at` y `product.version` (T-09, T-10), `sale_item.category_name` (T-11),
  `sale.sold_by_user_id` (T-12) y todos los índices (T-13).
- **Riesgo declarado:** el DDL de MySQL no es transaccional (commit implícito). Una migración que
  falla a la mitad deja el esquema parcial. Mitigación: revisiones pequeñas, y `GET_LOCK` para
  serializar `alembic upgrade head` cuando arrancan varias réplicas de `api`.
- **Hecho cuando:**
  - sobre una base vacía, arrancar el servicio crea el esquema y deja las cinco categorías
    consultables, sin ejecutar SQL a mano (CA-08.2);
  - las sondas de [`data-model.md` §10.1 a §10.3](data-model.md#10-cómo-se-comprueba-que-este-documento-no-miente)
    devuelven **25 columnas, 21 restricciones y 13 índices**, y los nueve `ck_*` figuran con
    `ENFORCED = 'YES'`;
  - un test de integración confirma que `alembic upgrade head` dos veces no duplica las categorías;
  - `alembic check` no detecta diferencias entre el modelo y la migración.

### T-03 · Repositorio de infraestructura

`simple-stock-flow-infra` con el compose que levanta MySQL (servicio `db`), la API (`api`) y la app
(`app`), el override `docker-compose.dev.yml` (perfil dev, publica 3306 al host), `.env.example` con
todas las claves, README y `verify.sh`.

- **Depende de:** T-02.
- **Límite duro:** **ni una línea de DDL** (artículo V). El contenedor crea la base `stockflow`
  vacía (usuario `stockflow`); el esquema lo pone el servicio con Alembic. Lo que el repositorio no
  lleva nunca, en [`architecture.md` §8](architecture.md).
- **Qué impone el compose:** `db` con healthcheck `mysqladmin ping`; `api` con
  `depends_on: condition: service_healthy` y reintento de conexión propio; puerto 3306 solo
  interno; **único puerto publicado en producción: 8080 (`app`)**; `.env` ignorado por git. La
  imagen de `app` sale aquí de la plantilla Vite mínima servida por nginx (con proxy de `/api` y
  `/media` hacia `api:8000`); T-15 la llena.
- **Contrato que esta tarea impone a T-06:** las credenciales del primer administrador llegan por
  `ADMIN_EMAIL` y `ADMIN_PASSWORD`, y la configuración por `DATABASE_URL`, `JWT_SIGNING_KEY` y
  `MEDIA_ROOT`. **Ningún secreto tiene valor por defecto.**
- **`verify.sh` se escribe antes que el compose** y debe fallar primero. Comprueba contenedores
  sanos, 401 sin token, el flujo de logs (T-14) y recompone la cifra de la tabla de progreso
  (artículo X.1).
- **Hecho cuando:** sobre un equipo limpio con Docker y nada más, `docker compose up` levanta los
  tres servicios; `app` responde en 8080; la API crea el esquema sola en el primer arranque;
  reiniciar **conserva** datos y binarios; **arrancar sin `JWT_SIGNING_KEY` falla con un error
  explícito**, no con una clave de relleno; y `./verify.sh` termina en verde.

---

## Fase 1 — Cerrar los casos de uso. Test primero, siempre

### T-04 · `ProductCatalogService`

Implementa el puerto entrante `ManageProducts` con los **siete** métodos: crear, actualizar, dar de
baja, obtener, listar paginado, adjuntar imagen y **listar las categorías de referencia**. Con su
primer endpoint instala el **único** exception handler que traduce excepciones de dominio a HTTP y
la reasignación de la validación de forma de 422 a 400 (`api-contract.md`).

- **Depende de:** T-02.
- **Cubre:** CA-02.1 a CA-02.4, CA-01.1 a CA-01.3, CA-01.5, CA-03.1, CA-03.2.
- **El séptimo método es el que se olvida.** `GET /api/categories` lo invoca la app, y sin él no se
  puede pintar el selector de categoría de ningún formulario de producto — y sin categoría no se
  crea un producto (D-10). Sale de `CategoryRepository.list_all`: falta el método en el puerto entrante
  y un router de tres líneas. Forma y alcance en [`architecture.md` §3.1](architecture.md) —
  solo lectura, sin paginar.
- **Cuidado con el orden al dar de baja:** anular la clave de imagen y confirmar **antes** de
  borrar el binario (D-08). Al revés queda una imagen rota permanente (CA-03.3).
- **Hecho cuando:** cada criterio de aceptación tiene su test en `tests/application/` con los
  puertos en doble (fakes), `pytest tests/application` pasa, y el servicio no contiene ni un `if`
  de negocio (Artículo VI): las reglas están en `domain/`.

### T-05 · Guarda de moneda en `Product`

Rechazar un precio cuya moneda no sea la del sistema, por las dos puertas por las que un precio
entra al producto: `Product.change_price(Money)` y `Product.create(name, Money, …)`.

- **Depende de:** nada (más allá de T-01).
- **Cubre:** RN-09.
- **Por qué en `Product` y no en `Sale.add_item`:** ningún producto puede tener otra moneda, así
  que una venta con monedas mezcladas **no se puede construir**; probarla sería una puerta sin
  salida. `Product` es la única puerta por la que una moneda ajena puede entrar, y debe rechazarla
  en voz alta: descartar la moneda en silencio (tomar solo el importe) guardaría un precio en USD
  como si fuera de la moneda del sistema.
- **Hecho cuando:** existen pruebas de dominio que construyen un producto en otra moneda y esperan
  el rechazo **por las dos puertas**, con `rn_09` en el nombre, y `pytest tests/domain -k rn_09`
  pasa. No se toca columna, persistencia ni D-05.

### T-06 · `AuthenticationService` y arranque del administrador

Autenticar, registrar vendedores, y crear el usuario administrador inicial al arrancar si no
existe. Hash con `argon2-cffi`; token JWT HS256 con `PyJWT`.

- **Depende de:** T-02.
- **Cubre:** CA-07.1, CA-07.2, CA-07.5, CA-07.6, y la restricción de DP-04.
- **Reglas duras:** credenciales del administrador desde `ADMIN_EMAIL` y `ADMIN_PASSWORD` (el
  correo es el nombre de usuario del administrador), **nunca versionadas** (Artículo IX). El rechazo
  de credenciales **no distingue** usuario inexistente de contraseña incorrecta. El resultado de
  autenticar **jamás** incluye el hash. El registro rechaza `admin` con 422; el alta del primer
  administrador vive en un puerto aparte que **no se inyecta en el adaptador HTTP**.
- **Hecho cuando:** los tests de `tests/application` cubren los criterios, y un test de integración
  confirma que dos altas concurrentes del mismo usuario dejan una sola fila y que quien pierde la
  carrera recibe una regla de negocio, no un error de base de datos (RN-10).
- **Lo que esta tarea NO cubre:** **CA-07.7** —el rol en conjunto cerrado, RN-11—. Es de **T-21**.

### T-07 · `GetSalesService`

Obtener una venta con sus líneas y listar por rango paginado (puerto entrante `GetSales`).

- **Depende de:** T-02.
- **Cubre:** CA-05.1 a CA-05.4. **CA-05.5 no**: la inmutabilidad de la venta es estructural y la
  prueba **T-22**.
- **Hecho cuando:** el total devuelto se calcula desde las líneas y hay un test que lo prueba
  contra la suma, no contra un valor fijo (`rn_12`); un rango con fecha final anterior a la inicial
  se rechaza.
- **Límite que esta tarea no cruza:** el puerto `GetSales` es de **solo lectura** y se queda así.
  Ni un método que actualice, ni uno que anule — RN-07 (`spec.md` §4).

### T-08 · `SalesReportService` y su puerto de lectura

Puerto de salida de solo lectura `SalesReportQuery` que devuelve filas ya agregadas, e
implementación que agrupa en el motor (SQLAlchemy Core sobre MySQL 8.4).

- **Depende de:** T-02. **DP-01 ya está decidida**, no pendiente ([`spec.md` §7.0](spec.md)).
- **Cubre:** CA-06.1, CA-06.2, CA-06.5.
- **Prohibido:** cargar las ventas del rango y sumarlas en memoria. Es lo que el enunciado evalúa
  con más peso.
- **DP-01:** agrupar por identificador de producto y tomar el nombre de la **línea de la venta más
  reciente del rango**, con función de ventana (`FIRST_VALUE(product_name) OVER (PARTITION BY
  product_id ORDER BY sold_at DESC, …)`) en la consulta agregada. **Escrito como tal en el código**,
  no disimulado: la cláusula de ordenación, con su desempate, es el punto donde se lee la decisión.
  El nombre vivo del catálogo no se consulta nunca aquí. El porqué está en
  [`spec.md` §7.1](spec.md).
- **DP-02:** el reporte **no** se desglosa por vendedor. La firma del puerto de lectura no acepta
  vendedor ni como filtro ni como agrupación, y `SalesReportRow` no lleva ese campo.
- **Salvedad declarada:** `sale_item.category_name` es `NOT NULL` desde T-02 pero lo puebla el caso
  de uso en T-11; hasta entonces los tests de esta tarea insertan las líneas con la etiqueta
  explícita. El plan de la consulta (`EXPLAIN` de Q9, [`data-model.md` §10.3](data-model.md#103-índices-y-planes))
  debe usar `idx_sale_sold_at` y `uq_sale_item_sale_product`.
- **Hecho cuando:** un test de integración compara el resultado contra un conjunto de ventas
  conocido, otro confirma que el reporte vacío devuelve vacío y no error, y **un tercero vende,
  renombra el producto y comprueba que el reporte muestra el nombre más reciente del rango** —que
  es la única forma de que DP-01 deje de ser una frase en un documento.

---

## Fase 2 — Robustez. Lo que separa una prueba aprobada de una prueba buena

### T-20 · Las barreras del motor, probadas

**Depende solo de T-02 y no bloquea a nadie.** Es la tarea que convierte «el motor las tiene» en
algo comprobado.

[`data-model.md` §4](data-model.md#4-restricciones-e-índices-dónde-debe-vivir-cada-regla) prevé
**21 restricciones** en el motor y fija el criterio: ADR-002 establece para `stock >= 0` que *"si
la restricción salta, algo escribió fuera del adapter"*, y ese criterio se aplica a **todas** las
invariantes que MySQL puede expresar. Una invariante que solo vive en Python protege a la
aplicación, no a los datos: cualquier cliente SQL, cualquier script y cualquier servicio futuro la
saltan sin enterarse. T-02 las **escribe** (`initial_schema`); esta tarea **no añade ninguna:
demuestra que cada una existe y rechaza**. MySQL aplica `CHECK` desde 8.0.16 y es válido en 8.4.

**Por qué es tarea propia.** Alembic no autogenera los `CHECK`: una migración revisada con prisa
puede perder uno sin que ningún test de dominio lo note, y una restricción que nadie intentó violar
no está probada, está escrita.

**Los once puntos, sin margen de interpretación** (ocho de los nueve `CHECK` más la nulabilidad, la
clave foránea y la única que el dominio no puede dar por sí solo):

| # | Objeto | Definición exacta |
|---|---|---|
| 1 | `sale_item.sale_id` | **`NOT NULL`**. Es precondición del punto 3: con una columna nulable el único compuesto no protegería nada (en un índice único cada `NULL` es distinto). `fk_sale_item_sale_id` conserva su `ON DELETE CASCADE` |
| 2 | `fk_sale_item_product_id` | `sale_item.product_id → product.id` · **`ON DELETE RESTRICT`** · `ON UPDATE NO ACTION`. Lo sirve `idx_sale_item_product_id`: no se declara otro índice |
| 3 | `uq_sale_item_sale_product` | **Único** sobre `(sale_id, product_id)`. Sirve también a la FK de `sale_id`: **no hay índice aparte sobre `sale_id`** |
| 4 | `ck_product_price_positive` | `CHECK (price > 0)` |
| 5 | `ck_sale_item_quantity_positive` | `CHECK (quantity > 0)` |
| 6 | `ck_category_name_not_blank` | `CHECK (CHAR_LENGTH(TRIM(name)) > 0)` — «no vacío» incluye la cadena de solo espacios |
| 7 | `ck_user_role_allowed` | `CHECK (role IN ('admin','seller'))`, **y la columna `role` es `VARCHAR(40) CHARACTER SET ascii COLLATE ascii_bin`**. La colación binaria es lo que hace exacto el `IN`: con `utf8mb4_0900_ai_ci` (insensible a mayúsculas y acentos) `'ADMIN'` pasaría un `IN` normal |
| 8 | `ck_user_username_normalized` | `CHECK (CHAR_LENGTH(username) > 0 AND CAST(username AS BINARY) = CAST(LOWER(TRIM(username)) AS BINARY))` — **minúsculas y sin espacios en los extremos**, lo que hace `User.normalize_username`. `username` conserva `ai_ci` (la unicidad insensible a acentos es deseada), por eso **la comparación debe ser binaria**: sin `CAST ... AS BINARY` el `CHECK` jamás fallaría |
| 9 | `ck_product_name_not_blank` | `CHECK (CHAR_LENGTH(TRIM(name)) > 0)` |
| 10 | `ck_sale_item_unit_price_positive` | `CHECK (unit_price > 0)` |
| 11 | `ck_user_password_hash_not_blank` | `CHECK (CHAR_LENGTH(password_hash) > 0)` |

Los puntos 4 a 11 son ocho `CHECK`; el noveno, `ck_product_stock_non_negative`, lo prueba **T-10**
(es la última barrera de ADR-002). Las demás claves foráneas y únicas (`fk_product_category_id`,
`fk_sale_sold_by_user_id`, `uq_category_name`, `uq_user_username`) las ejercen T-04, T-12 y T-06.

**La clave foránea del punto 2 no es una decisión nueva.** **ADR-003 ya la decidió** y se apoya en
ella de forma explícita: la baja lógica funciona porque *"la clave foránea hacia el producto queda
como barrera de última instancia para que un borrado manual falle ruidosamente"*.

**Nota fina sobre `price > 0`:** `Money` admite importe 0. La guarda del agregado vive en
`Product.change_price` —por la que pasa también la creación—, así que un precio 0 escrito por
cualquier otra vía entraría sin resistencia si faltara el `CHECK`.

- **Depende de:** T-02. Nada más.
- **Cubre:** la brecha entre el diseño de `data-model.md` §4 y el motor real. No añade ningún
  criterio de aceptación nuevo.
- **Hecho cuando:** por cada uno de los once puntos, un `INSERT` o `UPDATE` escrito **a mano en el
  cliente `mysql`** (`docker compose exec db mysql -u stockflow -p stockflow`) que lo viole **falla**,
  y se deja constancia del comando y del mensaje de error (3819 para `CHECK`, 1451/1452 para claves
  foráneas, 1062 para el único, 1048 para `NOT NULL`). Para el punto 7, el caso concreto es
  `role = 'ADMIN'`. Cada caso queda además como test en `tests/integration/`.

### T-09 · Baja lógica

Filtro **centralizado** en la base de los repositorios (SQLAlchemy no tiene filtro global: se
centraliza en un único método de consulta base y se prueba) y los tres contratos de repositorio de
D-03. La baja pasa a ser un `UPDATE` de `deleted_at`; el borrado duro desaparece del puerto.

- **Depende de:** T-04.
- **Cubre:** CA-02.5, CA-02.6, CA-01.4, RN-08.

**Lo que el esquema ya trae (T-02), y esta tarea solo usa:**

| Qué | Definición exacta |
|---|---|
| Columna | `product.deleted_at` · **`DATETIME(6)`, nulable** · **sin `DEFAULT`** · solo en el modelo de persistencia, nunca en la entidad |
| Significado | `NULL` = producto activo. No nula = instante de la baja, **en UTC** |
| Claves foráneas | `fk_sale_item_product_id` y `fk_product_category_id`, ambas `ON DELETE RESTRICT` |
| Índices | `idx_product_category_active_name` e `idx_product_active_name`; su comprobación con `EXPLAIN` es T-13 |

El tipo no es una elección de esta tarea: MySQL no tiene «instante con zona», así que **toda marca
de tiempo del sistema es `DATETIME(6)` en UTC** y el adaptador convierte desde y hacia `datetime`
con tz UTC. La nulabilidad tampoco: una columna nula como «activo» permite el índice compuesto y la
unicidad condicional de más abajo, y un valor centinela no.

**Unicidad de `product.name`: no se crea, y es una decisión, no un olvido.** ADR-003 exige que
*"cualquier restricción de unicidad que se añada sobre el producto debe ser parcial sobre los
activos"*. Es una regla **condicional**, y no hay ninguna unicidad sobre `product.name`. Ningún
criterio de aceptación pide que dos productos no puedan llamarse igual, así que crearla sería
inventar una regla de negocio que nadie pidió.

- **T-09 no crea unicidad sobre `product.name`.** La exigencia de ADR-003 no aplica porque no hay
  nada a lo que aplicarse.
- **Si alguna vez se añade**, MySQL no tiene índices parciales: la forma obligada es una columna
  generada `active_name = IF(deleted_at IS NULL, name, NULL)` con un índice **único** sobre ella
  (los `NULL` no chocan en un `UNIQUE`; [`data-model.md` §6.2](data-model.md#62-índices-del-diseño)),
  nunca un único sobre `name` a secas: si no, un producto retirado reservaría su nombre para
  siempre. Recordar que la comparación del motor ignora mayúsculas y acentos.

- **El detalle que decide si funciona:** el filtro debe aplicarse **también** al cargar productos
  para vender. Sin eso **se puede vender un producto retirado**, que es el defecto que la baja
  lógica viene a evitar. Todo método del repositorio de productos parte de la consulta base filtrada.
- **Hecho cuando:** hay un test de integración que da de baja un producto vendido y comprueba las
  tres cosas: desaparece del catálogo, no se puede vender, y su venta sigue consultable; y otro por
  cada método del repositorio (`find`, `search`, la carga para vender) que confirma que un producto
  dado de baja no se devuelve.
- **Y una cuarta, que es la que convierte RN-08 en una regla y no en una intención.** El test debe
  afirmar que **el producto vendido sigue existiendo como fila** después de la baja —no solo que no
  se lista—, porque «se da de baja» y «se borra y la consulta no falla» son indistinguibles desde el
  catálogo y radicalmente distintos para el histórico. La forma concreta: dar de baja, consultar la
  venta, y comprobar que la línea conserva el nombre y el precio congelados. Nombre del test con
  `rn_08`.
- **Lo que esta tarea no debe tener, y hay que comprobar que no existe:** `ProductRepository` no
  tiene `remove`. Mientras ese método exista, el borrado duro es alcanzable desde el caso de uso y
  RN-08 dependería de que nadie lo invoque. **Una regla que depende de la buena voluntad del que
  escribe el siguiente método no está cerrada.** Un test de `tests/architecture/` lo afirma. La
  barrera del motor, la clave foránea `ON DELETE RESTRICT` de `sale_item` hacia `product`, se
  prueba en **T-20 punto 2**: las dos son necesarias y ninguna sustituye a la otra.

### T-10 · `PlaceSaleService` y concurrencia optimista sobre el stock

Implementa el caso de uso `PlaceSale` (CA-04.1 a CA-04.5, test primero) y le añade: **`version_id_col`
de SQLAlchemy** sobre la columna `product.version INT NOT NULL` que ya trae `initial_schema` (T-02;
**no existe en la entidad de dominio**), traducción de `StaleDataError` a un `ConcurrencyConflict`
definido por la aplicación dentro del adaptador, reintento acotado (**máximo 3**) del caso de uso
completo, y **409** hacia el cliente cuando se agotan. `CHECK (stock >= 0)` en la base como última
barrera (ya creado en T-02).

- **Depende de:** T-02.
- **Cubre:** CA-04.1 a CA-04.6, RNF-01, RN-01, RN-04, RN-05, RN-06, RN-12.
- **Dos errores que anulan la tarea:** reintentar solo la escritura en vez del caso de uso
  entero, y dejar que `StaleDataError` salga del adaptador de persistencia (D-04).
- **Regla de diseño del reintento:** el reintento vive en el caso de uso, no en un decorador de
  composición. El puerto `UnitOfWork` expone `discard_changes()`, y **cada intento empieza
  descartándose**. Sin eso el segundo intento arrastra el descuento del primero y descuenta dos
  veces: la forma exacta del error que ADR-002 advierte.
- **Hecho cuando:**
  - un test de integración lanza dos ventas simultáneas del último ejemplar (dos sesiones/hilos
    contra MySQL real): una confirma, la otra recibe conflicto, y el stock final no es negativo;
  - un test del servicio con puertos en doble prueba la política de reintento (cuenta de intentos,
    descarte previo, 409 al agotar);
  - `pytest tests/application tests/integration` pasa.

### T-11 · Congelar la categoría en la línea de venta

Añadir el nombre de categoría a `SaleItem`, pasar la categoría a `Sale.add_item`, y resolverla en
el caso de uso cargando las categorías sembradas.

- **Depende de:** T-08.
- **Cubre:** CA-06.1, CA-06.3, CA-06.4, RNF-02.

**Lo que el esquema ya trae (T-02), y esta tarea solo puebla:**

| Qué | Definición exacta |
|---|---|
| Columna | `sale_item.category_name` · **`VARCHAR(120)`, `NOT NULL`** · **sin `DEFAULT`** |
| Por qué 120 | Es una **copia congelada** de `category.name`, que es `VARCHAR(120)`. Un límite menor truncaría al copiar; uno mayor admitiría lo que el origen no puede producir. Si una se ensancha, la otra en la misma migración |
| Claves foráneas | **Ninguna, y es el punto entero de la tarea.** Si apuntara a `category`, renombrar una categoría reescribiría el histórico, que es justo lo que esto evita |
| Índices | Ninguno. Ningún patrón de acceso de [`data-model.md` §6.1](data-model.md#61-patrones-de-acceso) filtra ni ordena por el nombre de categoría de la línea |

**`NOT NULL` desde el diseño.** Como la columna nace no nula, sin relleno y sin valor por defecto,
**ningún código puede insertar una línea sin categoría**: hasta que esta tarea la resuelve en el
caso de uso, los tests de T-08 insertan las líneas con `category_name` explícito. Por eso el
sembrador de T-24 se ejecuta **después** de T-11: la API tiene que congelar la categoría para poder
registrar una venta.

- **La agrupación del reporte pasa a ser por la etiqueta congelada** (CA-06.1): una fila por
  producto y etiqueta; ni ventana ni desempate para la categoría. La ventana de DP-01 se queda
  solo para el nombre del producto.
- **Coste declarado:** cambia la firma de un método del agregado y añade una dependencia al
  camino de escritura. Se paga porque sin ello **recategorizar un producto reescribe reportes de
  períodos ya cerrados**.
- **Hecho cuando:** el test de fidelidad pasa: vender, renombrar, recategorizar, dar de baja, y
  el reporte del período no cambia; y un test confirma que recategorizar entre dos ventas produce
  **dos filas** en el reporte, no una.

### T-12 · Autoría de la venta con integridad referencial

Guardar con cada venta el identificador del usuario que la registra, con clave foránea
restrictiva, conservando el nombre de usuario como atributo congelado del evento.

- **Depende de:** T-06.

**Lo que el esquema ya trae (T-02), y esta tarea solo puebla:**

| Qué | Definición exacta |
|---|---|
| **Columna del nombre** | `sale.sold_by_username` · `VARCHAR(120)`, `NOT NULL` — el nombre congelado |
| **Columna del identificador** | `sale.sold_by_user_id` · **`CHAR(36)`, `NOT NULL`** · **sin `DEFAULT`** |
| **Clave foránea** | `fk_sale_sold_by_user_id`: `sold_by_user_id → user(id)` · **`ON DELETE RESTRICT`** · `ON UPDATE NO ACTION` |
| **Índice sobre la columna** | `idx_sale_sold_by_user_id`, declarado **solo porque la clave foránea FK-4 lo exige**. No sirve a ninguna consulta de negocio: la única que lo justificaría —ventas por operador— cruza datos personales y está cerrada por DP-02 ([`data-model.md` §6.3](data-model.md#63-índices-descartados-y-por-qué)) |

**El nombre largo es el correcto en cuanto `sold_by_user_id` está al lado:** con los dos campos
juntos, `sold_by` no diría cuál de los dos es. El dominio lo llama `Sale.sold_by_username`; la API
lo expone como `soldBy`.

**Esto no toca el contrato de API.** El campo de autoría de la vista de venta es uno solo
(`api-contract.md`). El cambio se agota entre la entidad y la tabla.

**`NOT NULL` desde el diseño.** Un autor inventado en un registro contable es peor que no tener el
campo, y admitir nulos «por si acaso» dejaría ventas sin responsable para siempre.

**Por qué `RESTRICT` y no otra cosa:** *"la autoría de una venta no puede quedar huérfana"*
([`data-model.md` §5](data-model.md#5-política-de-claves-foráneas), FK-4). `CASCADE` borraría
ventas al borrar un usuario, que contradice la retención indefinida de
[`data-model.md` §7.1](data-model.md#71-retención), y `SET NULL` es incompatible con `NOT NULL`. No
existe ningún puerto que borre usuarios; `RESTRICT` es la barrera que hace que, el día que se
añada uno, falle ruidosamente en vez de callar.

- **Por qué los dos campos:** el identificador da integridad y sobrevive a un renombrado; el
  nombre da fidelidad a lo que se registró. Ninguno sustituye al otro.
- **Coste:** toca el agregado, el comando y el router. **Es la tarea más prescindible de esta
  fase**: si hay que recortar alcance, se recorta esta y se dice por qué.
- **Hecho cuando:** no se puede insertar una venta con un autor inexistente (error 1452 del motor),
  y existe el test de integración que lo comprueba.

### T-13 · Búsqueda por nombre e índices según los patrones de acceso

Los índices de acceso **ya los crea `initial_schema` (T-02)**; esta tarea implementa la consulta de
búsqueda que los usa y **comprueba con `EXPLAIN` que sirven**. MySQL no tiene índices parciales ni
`pg_trgm`; las decisiones están en [`data-model.md` §6.2](data-model.md#62-índices-del-diseño):

| Patrón de acceso | Índice / técnica | Por qué esta forma |
|---|---|---|
| Catálogo activo filtrado por categoría y ordenado por nombre (Q1) | `idx_product_category_active_name` sobre `(category_id, deleted_at, name)` | Sin parcialidad, el predicado de baja `deleted_at IS NULL` se resuelve como columna del índice, **en segunda posición** tras la columna de igualdad. Empieza por `category_id` para que **sirva también a la clave foránea** FK-1 y InnoDB no cree otro índice implícito |
| Catálogo activo sin categoría, ordenado por nombre (Q1) | `idx_product_active_name` sobre `(deleted_at, name)` | Sirve el orden por nombre del catálogo completo sin ordenar en memoria |
| Búsqueda por nombre que **contiene** el texto (CA-01.2) | **`LIKE CONCAT('%', :q, '%')`** con la colación `utf8mb4_0900_ai_ci`, sin índice de texto | Ningún árbol B sirve a un comodín a la izquierda; el filtro por categoría y `deleted_at` ya reduce el conjunto y el comodín solo recorre ese subconjunto. Es insensible a mayúsculas y acentos sin extensión alguna. El adaptador **escapa `%`, `_` y `\`** del texto del usuario |

**Escalada condicionada, no inicial.** `FULLTEXT ... WITH PARSER ngram` solo se adopta si la sonda
`EXPLAIN` de Q1 ([`data-model.md` §10.3](data-model.md#103-índices-y-planes)) muestra que el
recorrido deja de ser aceptable **al volumen real**. Si se adopta: `ngram_token_size` es variable de
arranque del servidor (por defecto 2), un texto de un solo carácter no usa el índice, y la decisión
se anota en `data-model.md` §6.2 con su sonda **antes** de tocar el esquema (nueva revisión de
Alembic, una sola intención). **No se crea ahora.**

**No se crean** índices de una sola columna sobre `product.name` ni `product.category_id`: se
sustituyen, no se suman, porque cada índice de más encarece toda escritura, para siempre. Los demás
índices del diseño (`uq_sale_item_sale_product`, `idx_sale_sold_at`, `idx_sale_item_product_id`,
`idx_sale_sold_by_user_id` y los únicos de `category.name` y `user.username`) tampoco se tocan aquí.
Como la comparación del motor ignora mayúsculas y acentos, la búsqueda y el orden heredan ese
comportamiento (CA-01.2).

- **Depende de:** T-09 (el predicado necesita el filtro de baja). Conviene **T-20 antes**, para
  que las sondas de restricciones e índices se ejecuten sobre un esquema ya comprobado.
- **Requisito que los hace servir de algo:** el predicado `deleted_at IS NULL` debe aparecer
  **literalmente** en la consulta, o el motor no lo usará como condición de acceso.
- **Hecho cuando:** un test de integración prueba que «cafe» encuentra «Café» y que un `%` escrito
  por el usuario se trata como texto; tras `alembic upgrade head`, la sonda de índices de
  `data-model.md` §10.3 devuelve los 13 índices, **ninguno `FULLTEXT`** y ningún índice de una columna
  sobre `name` ni sobre `category_id`; y `EXPLAIN` de la consulta de catálogo por categoría usa
  `idx_product_category_active_name`. Ningún índice indexa un booleano ni una enumeración de dos
  valores.

### T-21 · Las invariantes de dominio que no se pueden olvidar

**Quien escribe el dominio olvida tres cosas:** que `Quantity` rechaza 0 y los negativos, que `User`
rechaza un rol fuera del conjunto cerrado, y que `normalize_username` hace lo que dice. En ellas
viven **RN-03 y RN-11**. Esta tarea es su dueño. Son dos ficheros en `tests/domain/`:

| Fichero | Qué afirma | Regla |
|---|---|---|
| `test_quantity.py` | Construir `Quantity` con **0** lanza. Construirla con un **negativo** lanza. Construirla con 1 no lanza y conserva el valor | **RN-03** |
| `test_user.py` | `User.create` con un rol que no es `admin` ni `seller` lanza —**ese es el único punto donde la validación del rol se ejerce**—, igual que el rol vacío o por omisión. Y que `normalize_username` guarda `«  Ana  »` como `ana` | **RN-11**, y el lado de minúsculas de **RN-10** |

- **Depende de:** nada (más allá de T-01). Es la tarea más barata del plan entero y cierra dos de
  las cuatro reglas con tarea propia.
- **Cubre:** RN-03, RN-11, y directamente CA-04.5 y CA-07.7.
- **Qué no hace:** **no toca el motor.** `ck_sale_item_quantity_positive` y
  `ck_user_role_allowed` se escriben en T-02 y se prueban en **T-20**, puntos 5 y 7. Las dos capas hacen falta: el test de
  dominio prueba que la aplicación respeta la regla, el `CHECK` impide que un cliente SQL la salte.
- **`Category` no tiene fichero propio, y es una decisión.** Su única guarda —nombre no vacío— no es
  una regla de §4: las categorías son datos de referencia sembrados y su CRUD está **fuera de
  alcance** ([`spec.md` §6](spec.md)). Su barrera en el motor se prueba en T-20 punto 6, que es donde
  importa, porque el riesgo real es un `INSERT` a mano y no un formulario que no existe.
- **Hecho cuando:** los dos ficheros existen, `pytest tests/domain -k "rn_03 or rn_11"` pasa, y
  **cada test falla si se le quita la guarda que afirma**. Un test que pasa con y sin la guarda no
  prueba la guarda: prueba el lenguaje.

### T-22 · Probar que una venta no se puede tocar

**RN-07 es la afirmación 2 de `spec.md` §1** —*«una venta registrada no se puede alterar
después»*—. Se cumplirá por **ausencia**, y esa es justamente la trampa: una ausencia no protesta
cuando desaparece.

La venta debe ser inmutable porque:

- `Sale` no expone quitar ni reemplazar líneas —solo `open`, `add_item` y `ensure_confirmable`— y
  entrega sus líneas como una colección de solo lectura.
- `SaleItem` es un `dataclass(frozen=True)`: no admite asignación de atributos.
- `SaleRepository` tiene exactamente cuatro métodos: `find`, `search`, `list_by_range` y `add`.
  **Ninguno actualiza ni borra** — y el contraste está a la vista: ningún repositorio de productos
  debe tener `remove` (T-09).
- `/api/sales` publica **un `POST` y dos `GET`**. No hay `PUT`, ni `PATCH`, ni `DELETE`.

**Nada de eso está afirmado hasta que esta tarea lo hace.** Una pasada futura puede añadir un
`Sale.remove_item` «solo para un caso», un `SaleRepository.update` que el ORM agradezca, o un
`DELETE /api/sales/{id}` para limpiar pruebas, **y el conjunto de tests seguiría entero en verde**.

**Cómo se prueba una regla que se cumple por ausencia.** No ejercitando una operación, sino
comprobando la **forma** del sistema, en `tests/architecture/`. Tres afirmaciones:

| # | Qué se comprueba | Cómo |
|---|---|---|
| 1 | El agregado no deja mutar una venta ya formada | La colección que devuelve `Sale.items` **rechaza** que le añadan o le quiten un elemento, y asignar un atributo de `SaleItem` lanza `FrozenInstanceError`. Es una afirmación de comportamiento, no de introspección |
| 2 | El agregado no expone una puerta nueva | Por **introspección** (`inspect`, `dataclasses.fields`) sobre `Sale` y `SaleItem`: el conjunto de métodos públicos que modifican estado es exactamente el declarado —`open`, `add_item`, `ensure_confirmable`— y **ningún atributo es asignable**. Si alguien añade un método, el test falla y **obliga a justificarlo aquí**, que es el punto entero |
| 3 | La venta no se puede alterar desde fuera del agregado | Por introspección sobre el puerto `SaleRepository`: sus miembros públicos son los cuatro nombrados y ninguno más. Y sobre las rutas de la aplicación FastAPI: ninguna ruta bajo `/api/sales` admite `PUT`, `PATCH` ni `DELETE` |

- **Depende de:** nada (más allá de T-01 y de que existan `Sale`, el puerto y la ruta, de T-10 y
  T-07). Se escribe contra lo que hay y **debe pasar en verde nada más escribirse** — que es lo
  correcto: esta tarea no arregla un defecto, **levanta una alarma sobre una propiedad que debe
  ser cierta** para que deje de serlo con ruido en vez de en silencio.
- **Cubre:** RN-07, CA-05.5.
- **El riesgo de escribirlo mal, dicho una vez:** un test de introspección que solo cuente métodos
  es frágil sin ser útil —se rompe al renombrar—. Los tres puntos afirman **capacidad de mutación**,
  no nombres: un método de lectura nuevo no debe romper nada; uno que escriba, sí.
- **Hecho cuando:** los tres pasan (`pytest tests/architecture -k rn_07`), y **se ha comprobado que
  fallan**: añadiendo temporalmente un `remove_item` al agregado y un `DELETE` a la ruta, y viéndolos
  ponerse en rojo antes de deshacerlo. Un test estructural que nadie vio fallar no prueba la
  estructura.

---

## Fase 3 — Entrega

### T-14 · Registro estructurado

Registro JSON a stdout con la biblioteca estándar `logging` y un formateador JSON propio, con una
línea por petición y el **identificador de correlación** en cada línea (cabecera `X-Request-ID`
aceptada o generada, y devuelta en la respuesta).

- **Auditar que no se filtra nada**: ni token, ni hash, ni contraseña, ni nombre de usuario en
  claro donde no corresponda ([`data-model.md` §7](data-model.md#7-privacidad-y-retención)).
- **Cubre:** RNF-05.
- **Hecho cuando:** una petición fallida se sigue de punta a punta por su correlación, y **`verify.sh`
  lo comprueba contra el flujo real de logs** (`docker compose logs api`): cuenta las líneas
  correlacionadas y busca la contraseña enviada, el prefijo `$argon2` y un JWT (`eyJ`), sin
  encontrar nada. La comprobación se prueba también contra un flujo sucio sintético, para ver que
  detecta.

### T-15 · App: CRUD y reporte contra la API real

React 19 + TypeScript + Vite en `simple-stock-flow-app`. Repite el hexágono: `src/domain/`,
`src/application/{ports,use-cases,state}/`, `src/infrastructure/{http,mappers,interceptors}/` y
`src/features/`; DTO en `src/infrastructure/http/dto/api.dto.ts`. Pantallas de catálogo con alta,
edición y baja; carrito de venta; reporte por rango; `/vendedores/nuevo` solo para el
administrador y sin selector de rol (DP-04). `<html lang="es">`. Servida por nginx (8080) con proxy
de `/api` y `/media` a `api:8000`; es el único consumidor de la API.

- **Depende de:** T-04, T-06, T-07, T-08.
- **Hecho cuando:** un evaluador entra, crea un producto, vende, ve el stock bajar y ve la venta
  en el reporte, **sin abrir la documentación autogenerada (`/docs`)**; `vitest run`, el linter y
  `tsc --noEmit` pasan.

### T-16 · Los README de `api`, `app` e `infra`

Uno por repositorio desplegable: qué es, cómo se levanta, cómo se prueba, **y qué falta**. El de
`api` incluye la explicación de la baja lógica al evaluador ([`spec.md` §6](spec.md)).

- **Cubre:** CA-08.4, Artículo X.
- **Hecho cuando:** responden a las cinco preguntas del artículo XIII, **cada comando** se ejecutó
  antes de escribirlo, y ninguna afirmación del README es falsa. Si algo no corre, el README lo dice
  en vez de omitirlo. Los README de `docs`, `page` y `tool` son de T-25, T-23 y T-24.

### T-17 · Git y publicación

Commits con historia legible en los seis repositorios, y publicación en GitHub.

- **Depende de:** **T-18**, T-23, T-24 y T-25. Regla del propietario del proyecto: **no se commitea
  nada hasta tener una versión funcional probada.** El orden es verificar primero y commitear
  después, no al revés.
- **Consecuencia asumida:** hasta entonces el trabajo vive sin red. Un borrado accidental no tiene
  vuelta atrás, y eso es una decisión tomada a sabiendas, no un olvido.
- **Mejora futura opcional, no requisito:** convertir `api` y `app` en submódulos de `infra` para
  clonar con un solo comando. No se hace en esta tarea; mientras tanto se clonan como hermanos.
- **Hecho cuando:** los tres repositorios que se despliegan (`infra`, `api`, `app`) clonan como
  hermanos y levantan desde cero siguiendo su README, que es literalmente lo que el enunciado pide
  entregar; y `docs`, `page` y `tool` están publicados con su README.

### T-18 · Verificación de punta a punta

Ejecutar el criterio de terminado de [`spec.md` §8](spec.md), los catorce pasos, sobre un equipo
limpio.

- **Depende de:** T-01 a T-16 y T-19 a T-22 (el sembrador T-24 es opcional para el guion; el paso 4
  a 9 se puede ejecutar a mano).
- **Hecho cuando:** los catorce pasos pasan, y se deja constancia de qué se ejecutó y qué devolvió
  cada uno en la fila de T-18 de la tabla de progreso. `./verify.sh` en verde y `docker compose up`
  desde cero. Un criterio de terminado que nadie ejecutó no es un criterio: es una lista de deseos.

### T-19 · Idioma y comentarios

Alinear el código con los artículos XI y XII. **Lo que lee el código** —identificadores,
comentarios, nombres de test— va en inglés; **lo que lee la persona que opera o usa el sistema**
—interfaz, mensajes de `verify.sh`, README, documentación— va en español. Una sola pregunta,
«¿quién lo lee?», decide sin discutir: una frontera con excepciones no es una frontera.

- **Depende de:** nada. **Se aplica a cada archivo que se toque**: la tarea se cierra una vez, pero
  la regla es permanente.
- **Hecho cuando:** `ruff check` pasa con las reglas de comentarios y docstrings activadas, no queda
  un comentario en español en `api`, `app` ni `tool` (un chequeo de `verify.sh` busca tildes y `ñ`
  en comentarios de `src/` y `tests/`), ni un separador decorativo (`# ----- JWT -----`), ni un
  comentario que se pueda borrar sin perder información.
- **Reglas de estilo que la tarea fija:**
  1. **Lo que más vale es no escribir comentarios que repitan la firma.** Un comentario inútil
     traducido solo es inútil en otro idioma.
  2. **Los letreros decorativos no se escriben: se sustituyen por funciones con nombre.**
     `bootstrap/` expone `configure_jwt`, `configure_cors`, `apply_migrations`, `mount_media` y
     `create_app`, y `create_app` queda en pocas llamadas encadenadas.
  3. **Los nombres de test son identificadores:** en inglés y, si ejercen una regla, con `rn_xx`.

---

### T-23 · Sitio público de presentación (`page`)

`simple-stock-flow-page`: sitio estático HTML/CSS —sin framework y sin llamadas a la API— que
presenta el producto: qué es Simple Stock Flow, las tres afirmaciones de [`spec.md` §1](spec.md)
y los enlaces a los repositorios. Servido por nginx o GitHub Pages; **ningún otro repo depende de él**
y no está en el compose.

- **Depende de:** nada.
- **Hecho cuando:** `docker run --rm -p 8081:80 -v ./:/usr/share/nginx/html:ro nginx` (o abrir
  `index.html`) sirve la página sin errores; `<html lang="es">`; un `grep` no encuentra `fetch(`,
  `XMLHttpRequest` ni la ruta `/api` (no habla con la API); los enlaces del README y de la página
  resuelven; y el README dice qué es, cómo se sirve y **qué falta**.

### T-24 · Sembrador de datos de demostración (`tool`)

`simple-stock-flow-tool`: CLI Python 3.12 (`python -m ssf_tool seed`) que crea productos, imágenes
de demostración y ventas **usando la API pública**, con credenciales del entorno (`API_BASE_URL`,
`ADMIN_EMAIL`, `ADMIN_PASSWORD`). **No toca la base de datos**: así `infra` no necesita Python.

- **Depende de:** T-04, T-06, T-10, y se ejecuta **después de T-11 y T-12** (la API debe congelar la
  categoría y registrar el autor para poder guardar una venta: ambas columnas son `NOT NULL`) y,
  conviene, de T-20 (esquema ya comprobado).
- **Hecho cuando:**
  - `python -m ssf_tool seed` contra un compose levantado deja productos con imagen visibles en la
    app y ventas visibles en el reporte;
  - ejecutarlo dos veces **no duplica** productos (es idempotente);
  - `pytest` pasa con la API simulada, y `ruff check` y `mypy --strict` en verde;
  - sus dependencias no incluyen `sqlalchemy` ni `pymysql` (no accede a la BD);
  - el README dice cómo se usa y qué falta.

### T-25 · Documentación (`docs`)

`simple-stock-flow-docs`: `enunciado/` (con `prueba-inventarios.docx`), `spec/` (estos documentos) y
`diagramas/` (fuentes Mermaid, `.mmd`, o bloques ```mermaid``` en los propios documentos). Su README
explica los **seis** repositorios, el orden de clonado como hermanos y cuáles se despliegan.

- **Depende de:** nada para empezar; se cierra al final, porque el paso 14 de `spec.md` §8 exige
  que ningún documento contradiga al código.
- **Hecho cuando:** todos los enlaces relativos entre los documentos resuelven (comprobación por
  script en `verify.sh` cuando `docs` es hermano); cada `T-xx` citado en `spec/` y en `plan.md` existe
  en la tabla de progreso y la cifra «N de 25» coincide con ella; y `grep` no encuentra referencias a
  rutas que no existen (`../traspaso`, `diagramas/*.svg`).

---

## Orden recomendado

```
T-01 ─► T-02 ─► T-03
         │
         ├─► T-20  (las barreras del motor, probadas — cuanto antes, ver abajo)
         ├─► T-04 ─► T-09 ─► T-13
         ├─► T-06 ─► T-12
         ├─► T-07
         ├─► T-10  (PlaceSale + concurrencia)
         └─► T-08 ─► T-11
T-05  (independiente, en cualquier momento tras T-01)
T-21  (independiente — RN-03 y RN-11. La más barata del plan)
T-22  (tras T-07 y T-10 — RN-07. Debe pasar en verde al escribirse: esa es la idea)

Fase 3: T-14 · T-15 · T-16 ─► T-18 ─► T-17
                                      └─ verificar primero, commitear después

Repos satélite (independientes del camino crítico):
  T-23 (page) · T-25 (docs, se cierra al final) · T-24 (tool, tras T-04, T-06, T-10, T-11, T-12, T-20)
  Los tres, antes de T-17.

T-19  se cierra una vez; la regla sigue vigente en cada archivo que se toque
```

**Sobre T-20 y por qué va primera de su rama sin bloquear a nadie.** Las restricciones las escribe
T-02; T-20 las prueba violándolas, y cuanto antes se haga antes se descubre un `CHECK` que la
migración autogenerada haya perdido (Alembic no los autogenera). T-20 y T-13 son ramas
**independientes**: ninguna crea índices ni restricciones, las dos comprueban lo que T-02 escribió.
Si ambas se hacen, **T-20 antes**.

**Si hay que recortar alcance**, el orden de sacrificio es: T-12, luego T-14, luego T-13, luego T-23.
**No se recortan** T-09, T-10 ni T-11: son las tres que sostienen las tres afirmaciones del problema
en `spec.md` §1, y sin ellas el sistema parece funcionar y no funciona. **Tampoco se recorta
T-20**: es lo único que demuestra que los `CHECK` escritos a mano en T-02 existen y rechazan.

**T-21 y T-22 tampoco se recortan, y cuestan un rato.** Entre las dos son unos pocos ficheros de
test y **ni una línea de producción**. Sin ellas, cuatro de las doce reglas de `spec.md` §4 siguen
siendo intenciones: RN-03 y RN-11 porque nadie las afirma, RN-07 porque se cumple por ausencia y
nadie vigila esa ausencia. **Recortar la prueba de una regla no recorta trabajo: recorta la regla.**
