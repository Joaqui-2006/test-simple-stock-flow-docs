# Plan técnico — Simple Stock Flow

**Cómo se construye lo que `spec.md` pide.** Cada decisión trae la alternativa que se descartó y
el motivo, porque una decisión sin alternativa descartada no es una decisión: es una suposición
que nadie revisó.

- **Fecha:** 2026-10-02
- **Rige bajo:** `constitution.md` (innegociable) y `spec.md` (qué y por qué)
- **Stack:** MySQL 8.4 · Python 3.12 (FastAPI, SQLAlchemy 2.x, Alembic) · React 19 + TypeScript ·
  Docker Compose, en seis repositorios hermanos de los que se despliegan tres
  ([`architecture.md`](architecture.md) §8).
- **La forma del sistema** —topología, hexágono, interfaces, flujos— está en
  [`architecture.md`](architecture.md). **Las decisiones estructurales**, en [`adr/`](adr/).
- **El modelo de datos vive entero en [`data-model.md`](data-model.md)** y en ningún otro sitio:
  glosario, entidades e invariantes, columnas, restricciones, claves foráneas, índices, privacidad,
  auditoría y semilla. Aquí quedan las **decisiones técnicas** que lo produjeron (§1).

---

## 1. Decisiones técnicas

### D-01 · Motor: MySQL 8.4 LTS (InnoDB, utf8mb4)

El enunciado admite SQLite, SQL Server o Postgres; MySQL es el motor del stack elegido para este
proyecto y se justifica por lo que el sistema necesita, no por preferencia:

- **Decimal exacto.** `DECIMAL(12,2)` es aritmética exacta en el motor: el importe del reporte —el
  entregable de más peso— no pasa por punto flotante.
- **Transacciones y bloqueo por fila.** InnoDB da transacciones ACID y nivel de aislamiento
  `REPEATABLE READ`, suficientes para la concurrencia optimista de D-04.
- **Restricciones reales.** Desde 8.0.16 `CHECK` se aplica de verdad (`stock >= 0` es la última
  barrera de D-04), y 8.4 es versión de soporte largo.
- **Operación ligera.** Imagen oficial lista para Docker, sin licencia, coherente con un entregable
  cuyo extra declarado es Docker Compose.

**Lo que MySQL no da, y el plan sustituye en lugar de esconder:**

| Falta en MySQL | Sustituto | Dónde se decide |
|---|---|---|
| Tipo de instante con zona | Se guarda **UTC en `DATETIME(6)`**; el adaptador convierte desde y hacia `datetime` con zona UTC (D-07) | [`data-model.md`](data-model.md) §3 |
| Índices parciales | Columna generada más índice único (los `NULL` no chocan en un `UNIQUE`), o índice compuesto que arranca por `deleted_at` | [`data-model.md`](data-model.md) §6.2 |
| Extensión de búsqueda por trigramas | `LIKE '%texto%'` con la colación `ai_ci` (el patrón es «contiene»); `FULLTEXT ... WITH PARSER ngram` solo como escalada si el `EXPLAIN` lo exige | [`data-model.md`](data-model.md) §6.2 |
| Testigo de versión del motor | Columna `version` gestionada por el adaptador (D-04) | ADR-002 |
| DDL transaccional | Migraciones pequeñas y `GET_LOCK` para serializar el arranque (D-02) | ADR-001 |

**Colación:** `utf8mb4_0900_ai_ci` es insensible a mayúsculas **y a acentos**. Implicación declarada:
dos nombres que solo difieren en acento o capitalización se consideran iguales en unicidad y en
búsquedas. Es el comportamiento buscado para un catálogo, y se prueba en integración.

**Alternativas descartadas:**

- **PostgreSQL.** Técnicamente la más cómoda: tipo de instante con zona, índices parciales, testigo
  de versión y trigramas existen de serie. Se descarta porque el stack del proyecto es MySQL, y
  porque cada una de esas ventajas tiene un sustituto de coste bajo y declarado (tabla anterior).
  **Si el evaluador lo exigiera, el plan se porta cambiando tres cosas:** el tipo de instante, el
  sustituto del índice parcial y la columna `version` por el testigo del motor.
- **SQLite.** Tiene afinidad de tipos, no tipos: una columna decimal se almacena como punto
  flotante y el importe dejaría de ser exacto. No tiene tipo fecha, y su escritura es de un solo
  escritor: no permite ejercitar la venta concurrente del último ejemplar (CA-04.6) contra un
  motor con la misma semántica que el de producción.
- **SQL Server.** Cumple todo técnicamente y se descarta solo por peso operativo: imagen de
  1,5–2 GB y licencia en un entregable cuyo extra declarado es Docker Compose.

**Cómo se verifica:** el test de integración arranca MySQL 8.4 con Testcontainers y comprueba el
`CHECK`, la colación y el redondeo de `DECIMAL(12,2)`.

### D-02 · Migraciones: Alembic, desde el adaptador de persistencia

```bash
alembic revision --autogenerate -m "<nombre>"
alembic upgrade head
```

Las migraciones viven en `src/stockflow/adapters/outbound/persistence/migrations/`, se autogeneran
desde los modelos de persistencia y se aplican al arrancar. **Decisión completa y alternativa
descartada: [ADR-001](adr/adr-001-propiedad-del-esquema.md).**

**Reglas que impone MySQL:** el DDL hace commit implícito, así que una migración que falla a la
mitad deja el esquema parcial. Por eso cada revisión es **pequeña y hace una sola cosa**, se revisa
a mano lo que autogenera Alembic, y el arranque toma un `GET_LOCK` para que varias réplicas no
migren a la vez.

**Por qué esto no viola la constitución:** los modelos de persistencia y las migraciones viven en el
adaptador de persistencia. El dominio no los conoce, la aplicación tampoco, y ningún tipo de
SQLAlchemy cruza un puerto. La dependencia sigue apuntando hacia dentro.

### D-03 · Baja lógica en los repositorios

La marca de baja, `deleted_at`, **no existe en la entidad de dominio**: es una columna del modelo de
persistencia. SQLAlchemy no tiene filtro global, así que el filtro se **centraliza en un único
método de consulta base** de los repositorios, y todas las lecturas parten de él. Consecuencias, que
son requisitos y no sugerencias:

| Operación | Filtra bajas | Por qué |
|---|---|---|
| Buscar catálogo | **Sí** | Un producto dado de baja no se ofrece |
| Cargar varios por identificador, previo a vender | **Sí** | Sin esto **se puede vender un producto dado de baja** |
| Cargar uno por identificador | **No** | Una línea de venta histórica debe poder resolver su producto |

**Alternativa descartada:** que cada consulta añada su propio `WHERE deleted_at IS NULL`. Lo seguro
tiene que ser lo que ocurre por omisión; un filtro repartido en cada consulta se olvida en la
primera nueva. **Cómo se verifica:** test de integración por cada operación de la tabla.

**Decisión completa: [ADR-003](adr/adr-003-baja-logica.md).**

### D-04 · Concurrencia de stock: optimista, con columna `version`

Entre leer el producto y confirmar la transacción cabe otra venta. Dos ventas simultáneas del
último ejemplar leerían el mismo stock y pasarían las dos.

MySQL no tiene testigo de versión de fila, así que la tabla `product` lleva una columna
**`version INT NOT NULL`** que gestiona el adaptador de persistencia (`version_id_col` de
SQLAlchemy). Si la fila cambió entre la lectura y el `UPDATE`, SQLAlchemy lanza `StaleDataError` y
el adaptador lo traduce a un `ConcurrencyConflict` definido por la aplicación. **La columna no
existe en la entidad de dominio**, solo en el modelo de persistencia.

Se reintenta **el caso de uso completo desde la lectura** (máximo 3 intentos, luego 409), no solo la
escritura: reintentar la escritura reaplicaría un descuento calculado sobre un stock viejo. La
restricción `CHECK (stock >= 0)` en la base es la **última barrera**, no el mecanismo: si salta,
algo escribió fuera del adaptador y debe verse, no capturarse en silencio.

**Decisión completa, con sus alternativas descartadas:
[ADR-002](adr/adr-002-concurrencia-optimista.md).**

### D-05 · Moneda: el sistema es monomoneda, y queda decidido en el mapeo

La persistencia guarda el importe en **una sola columna `DECIMAL(12,2)`** y reconstruye el value
object con la moneda por defecto del sistema. No hay columna de moneda en ninguna tabla, y ningún
contrato de entrada permite elegir una. **El sistema es monomoneda por construcción.**

El importe es `decimal.Decimal` en el dominio y nunca `float`: la conversión a la columna es exacta
y el mapper rechaza un valor con más de dos decimales en lugar de redondearlo en silencio.

Se mantiene así porque es coherente con el alcance: la estructura del value object admite
multimoneda, pero nadie la pide. El día que se pida, el cambio es **una columna extra en el mapper,
no un cambio de dominio** — que es exactamente el motivo de que el importe lleve su moneda dentro.

**Deuda barata que se evita desde el principio:** el agregado debe comprobar la moneda al añadir una
línea. La protección no puede vivir solo en el mapeo: la guarda cuesta tres líneas y cierra la
puerta donde corresponde (T-05).

**Alternativa descartada:** columna de moneda en la venta. Introduciría una dependencia funcional
de la venta sobre un atributo no clave —violación de tercera forma normal— para proteger algo que
la guarda protege mejor y más arriba.

### D-06 · Reporte: agregación en el motor por un puerto de lectura propio

El servicio del reporte no carga ventas: recibe filas ya agregadas de un puerto de solo lectura,
`SalesReportQuery`. Y la línea de venta **congela también el nombre de la categoría**, para que
recategorizar un producto no reescriba reportes de períodos cerrados.

**Decisión completa, con su coste y la pregunta que destapa (DP-01, ya decidida):
[ADR-004](adr/adr-004-reporte-agregado-y-congelado.md).**

### D-07 · Value objects y mapeo entre dominio y persistencia

El importe y la cantidad son `dataclass(frozen=True)` inmutables: no son entidades y no tienen
tabla. Los modelos de persistencia SQLAlchemy son **clases separadas** de las entidades de dominio,
con mappers en el adaptador: el dominio no se anota con nada del ORM.

- El importe se convierte a **una sola columna** `DECIMAL(12,2)` (D-05).
- La cantidad se convierte a un entero.
- **El instante se guarda sin zona.** Las columnas de fecha son `DATETIME(6)` en UTC; el mapper
  convierte un `datetime` con zona UTC del dominio a ingenuo al escribir, y vuelve a asignarle UTC al
  leer. Un `datetime` ingenuo que llegue al mapper desde el dominio es un error, no un caso a
  adivinar. El `Clock` entrega siempre instantes con zona.
- El total de la venta y el subtotal de la línea **no se persisten**: son cálculos (artículo VII).
- La colección de líneas del agregado se mapea de modo que el agregado conserve el control de su
  propia lista; la relación de SQLAlchemy no se expone al dominio.

**Riesgo declarado:** los constructores del dominio lanzan ante datos inválidos, y el mapper los
invoca al reconstruir desde la fila. **Es el punto de mayor riesgo técnico del mapeo y se prueba
primero** (T-01), antes de escribir los repositorios.

### D-08 · Binarios fuera de la base

La base guarda una **clave opaca**, nunca la ruta ni los bytes.

**Orden obligatorio al dar de baja un producto con imagen:** anular la clave y confirmar la
transacción **primero**, borrar el binario **después**. Un binario huérfano es inofensivo; una
clave que apunta a un binario borrado es una imagen rota permanente. El almacenamiento no participa
en la transacción de la base, así que "en la misma transacción" no es alcanzable y no se promete.

### D-09 · Autenticación

Token firmado (PyJWT, HS256) y autocontenido; sin estado de sesión, por tanto sin almacén de
sesiones. El hash (`argon2-cffi`) lo produce el puerto `PasswordHasher`: el dominio **nunca ve la
clave en claro** y el resultado de autenticar **nunca incluye el hash**. La clave de firma viene de
`JWT_SIGNING_KEY` y no tiene valor por defecto (artículo IX): si falta, el arranque falla con un
error explícito.

### D-10 · Datos semilla

Las categorías **no son datos de ejemplo: son una dependencia funcional dura.** El repositorio de
categorías es de solo lectura y la categoría del producto es obligatoria, así que **sin categorías
sembradas no se puede crear ni un producto** y el CRUD del enunciado no se podría ejercer. Van en
la revisión `seed_categories` (solo datos, separada de `initial_schema`), con identificadores fijos
(UUID en `CHAR(36)`) para que las pruebas puedan referenciarlas ([`data-model.md`](data-model.md) §9.1).

El **administrador inicial no se siembra desde SQL**: su hash solo puede producirlo el puerto de
hash. Sembrarlo desde SQL exigiría reimplementar una primitiva de seguridad —una segunda
implementación que puede divergir sin que nadie lo note— o versionar una credencial en el
repositorio. Se crea en el arranque, con `ADMIN_EMAIL` y `ADMIN_PASSWORD` del entorno.

---

## 2. Modelo físico del esquema

**Vive íntegro en [`data-model.md`](data-model.md) §3 y §4**, y solo allí: cada columna con tipo,
nulabilidad y defecto, y cada restricción con la marca de **dónde vive** —motor, dominio o
ambos—. Las consultas que lo verifican contra el motor están en su §10.

---

## 3. Índices y patrones de acceso

**Vive íntegro en [`data-model.md`](data-model.md) §6.** Los patrones de acceso derivados de los
puertos, los índices previstos, los descartados con su motivo, y el sustituto de los índices
parciales y de la búsqueda por texto en MySQL.

---

## 4. Datos personales y retención

**Vive íntegro en [`data-model.md`](data-model.md) §7**, clasificado atributo por atributo y con su
política de retención. La decisión sobre columnas de auditoría —**el proyecto no las lleva**— está
en su §8, y la estrategia de semilla en su §9.

---

## 5. Estrategia de pruebas

Los tres niveles y lo que cada uno **no** puede usar son el artículo VIII de
[`constitution.md`](constitution.md). Lo que añade este plan es **qué toca cubrir en cada uno**:

- `tests/domain/` — cada regla de `spec.md` §4, sin dobles.
- `tests/application/` — cada caso de uso, con fakes de los puertos salientes.
- `tests/integration/` — el mapeo, los filtros de baja lógica, la unicidad, la concurrencia y la
  consulta del reporte, contra **MySQL real vía Testcontainers** (exige Docker).
- `tests/architecture/` — `import-linter` y el test que recorre los `import` de `domain/`.

El comando es `pytest`.

**Tres pruebas que no pueden faltar**, porque son las que atrapan los defectos caros de este
diseño:

1. **Materialización de los value objects** (D-07): leer de la base un producto y una venta y
   comprobar que se reconstruyen sin lanzar, con el instante de vuelta en UTC con zona y el importe
   sin pérdida de precisión.
2. **Venta concurrente del último ejemplar** (CA-04.6): dos transacciones sobre el mismo producto;
   una confirma, la otra recibe conflicto, el stock no queda negativo.
3. **Fidelidad del reporte** (CA-06.4): vender, renombrar y recategorizar el producto, darlo de
   baja, y comprobar que el reporte del período **no cambia**.

---

## 6. Riesgos de implementación

> Los riesgos de la **forma** del sistema están en [`architecture.md`](architecture.md) §6. Aquí
> están los de construirlo.

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Se olvida filtrar bajas al cargar para vender | Se vende un producto retirado | Filtro centralizado en la base de los repositorios por omisión (D-03), más test de integración |
| El reporte se implementa cargando ventas en memoria | El entregable de más peso queda mal resuelto | CA-06.5 es criterio bloqueante, no preferencia |
| Una consulta no usa el índice previsto porque no lleva el predicado de `deleted_at` | Se paga el índice y no sirve | El predicado debe aparecer **literalmente** en el filtro ([`data-model.md`](data-model.md) §6.2); se comprueba con `EXPLAIN` |
| Una migración falla a la mitad (el DDL de MySQL no es transaccional) | Esquema parcial; el siguiente arranque no puede reaplicarla | Revisiones pequeñas de una sola operación, revisión manual del autogenerado, `GET_LOCK` en el arranque |
| La colación ignora acentos y mayúsculas | Dos nombres "distintos" chocan en una restricción de unicidad | Declarado en D-01; test de integración con "Café" y "cafe" |
| El instante se guarda sin zona y alguien escribe hora local | Ventas en horas equivocadas, rangos del reporte desplazados | Solo UTC en `DATETIME(6)`; el mapper rechaza `datetime` ingenuos (D-07) |
| Se ensancha un nombre sin ensanchar su copia congelada | La venta trunca el nombre **al registrarse**, y el histórico no se puede reparar | Regla de cambio: ambas columnas en la misma migración |
| El reporte se implementa sin aplicar DP-01 | Duplica filas tras un renombrado, en vez de mostrar el nombre congelado más reciente del rango | **DP-01 ya está decidida** (`spec.md` §7). La regla del desempate va en la consulta agregada, no en el servicio |
| La categoría congelada de T-11 hereda el desempate de DP-01 y nadie lo decide | La agregación elige en silencio | **H-1 está cerrado** ([`data-model.md`](data-model.md) §11.1): se agrupa por la etiqueta congelada, sin desempate para la categoría |
| El código avanza sin actualizar estos documentos | El spec deja de describir el sistema | Artículo X: si discrepan, gana el código y el documento se corrige |
