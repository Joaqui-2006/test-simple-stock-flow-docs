# Spec — Simple Stock Flow

**Qué debe hacer el sistema y por qué. Ninguna decisión técnica vive en este documento**
(esas están en `plan.md`). Un lector de negocio debe poder leerlo entero y objetar.

- **Origen:** `enunciado/prueba-inventarios.docx` (ruta prevista en `simple-stock-flow-docs`) — prueba técnica "Construcción de Sistema End-to-End".
- **Repositorios:** `simple-stock-flow-docs`, `-api`, `-app`, `-page`, `-infra` y `-tool`. Este documento vive en `docs`; el mapa completo está en `architecture.md`.
- **Fecha:** 2026-10-02

---

## 1. Problema

Un negocio necesita saber **qué tiene para vender y qué ha vendido**. Hoy no puede responder
ninguna de las dos preguntas sin contarlo a mano, y el recuento envejece en cuanto se registra
una venta.

El sistema debe sostener tres afirmaciones, y todo lo demás es accesorio:

1. El stock que muestra el catálogo es el stock que hay.
2. Una venta registrada no se puede alterar después.
3. El reporte de un período cerrado dice hoy lo mismo que dirá dentro de un año.

---

## 2. Actores

| Actor | Quién es | Qué puede hacer |
|---|---|---|
| **Vendedor** (`seller`) | Operador de mostrador | Consultar el catálogo, registrar ventas, consultar ventas y el reporte |
| **Administrador** (`admin`) | Responsable del catálogo | Todo lo del vendedor, más crear, editar y dar de baja productos, subir imágenes y dar de alta usuarios |
| **Anónimo** | Nadie autenticado | Únicamente autenticarse y consultar el estado de salud del servicio |

**No existe el comprador como actor del sistema.** La venta registra quién la tomó, no a quién
se le vendió. Es una decisión de alcance, no un olvido: no hay requisito de cliente final, y
añadirlo ampliaría la superficie de datos personales sin que nadie lo haya pedido.

---

## 3. Historias de usuario

Cada historia lleva criterios de aceptación verificables. "Verificable" significa que se puede
escribir el test antes que el código.

### HU-01 — Consultar el catálogo

> Como **vendedor**, quiero buscar productos por nombre y filtrar por categoría, para encontrar
> rápido lo que el cliente pide.

- **CA-01.1** Dado un catálogo con productos, cuando consulto sin filtros, entonces recibo una
  página de resultados con el total de elementos y el total de páginas.
- **CA-01.2** Dado un texto de búsqueda, entonces solo recibo productos cuyo nombre lo contiene,
  **sin distinguir mayúsculas ni acentos** («cafe» encuentra «Café»; consecuencia de la
  comparación del motor, declarada en `data-model.md`).
- **CA-01.3** Dado un filtro de categoría, entonces solo recibo productos de esa categoría.
- **CA-01.4** Dado un producto dado de baja, entonces **no aparece nunca** en los resultados.
- **CA-01.5** Dado un tamaño de página mayor que el máximo permitido, entonces el sistema aplica
  el máximo en lugar de rechazar la petición.

### HU-02 — Mantener el catálogo

> Como **administrador**, quiero crear, editar y dar de baja productos, para que el catálogo
> refleje lo que el negocio vende.

- **CA-02.1** Dado un producto nuevo con nombre, precio, stock y categoría, cuando lo creo,
  entonces queda disponible para la venta y recibo su identificador.
- **CA-02.2** Dado un precio menor o igual a cero, entonces la creación **se rechaza** con un
  mensaje que explica la regla.
- **CA-02.3** Dado un stock inicial negativo, entonces la creación se rechaza.
- **CA-02.4** Dada una categoría inexistente, entonces la creación se rechaza.
- **CA-02.5** Dado un producto **que ya se vendió**, cuando lo doy de baja, entonces desaparece
  del catálogo **y las ventas que lo contienen siguen intactas y consultables**.
- **CA-02.6** Dado un producto dado de baja, entonces **no se puede vender** aunque alguien
  conozca su identificador.
- **CA-02.7** Dado un vendedor, cuando intenta crear, editar o dar de baja, entonces la
  operación se rechaza por falta de permiso.

### HU-03 — Imagen del producto

> Como **administrador**, quiero asociar una imagen a un producto, para que el catálogo sea
> reconocible de un vistazo.

- **CA-03.1** Dada una imagen subida, entonces el producto la muestra en las consultas
  posteriores.
- **CA-03.2** Dado un producto sin imagen, entonces el catálogo lo devuelve sin imagen, **sin
  error y sin una dirección rota**.
- **CA-03.3** Dado un producto con imagen, cuando lo doy de baja, entonces **no queda una
  referencia apuntando a un binario que ya no existe**.

### HU-04 — Registrar una venta

> Como **vendedor**, quiero registrar una venta con varias líneas, para descontar el stock y
> dejar constancia de lo vendido.

- **CA-04.1** Dada una venta con líneas válidas, cuando la registro, entonces el stock de cada
  producto **baja exactamente** en la cantidad vendida y recibo el identificador de la venta.
- **CA-04.2** Dada una cantidad mayor que el stock disponible, entonces **la venta completa se
  rechaza** y **ningún** producto queda descontado.
- **CA-04.3** Dada una venta sin líneas, entonces se rechaza.
- **CA-04.4** Dado el mismo producto repetido en dos líneas, entonces se rechaza.
- **CA-04.5** Dada una cantidad menor o igual a cero, entonces se rechaza.
- **CA-04.6** Dadas **dos ventas simultáneas** del último ejemplar, entonces una de las dos se
  registra y la otra recibe un conflicto. **El stock nunca queda negativo.**
- **CA-04.7** Registrada la venta, cuando después cambia el nombre o el precio del producto,
  entonces **la venta sigue mostrando el nombre y el precio del momento en que se vendió**.

### HU-05 — Consultar ventas

> Como **vendedor**, quiero consultar una venta y listar las de un período, para resolver dudas
> sobre lo registrado.

- **CA-05.1** Dado el identificador de una venta, entonces obtengo su fecha, quién la registró,
  sus líneas y el total.
- **CA-05.2** Dado un rango de fechas, entonces obtengo una página de ventas de ese rango.
- **CA-05.3** Dado un rango cuya fecha final es anterior a la inicial, entonces se rechaza.
- **CA-05.4** El total de la venta **siempre cuadra con la suma de sus líneas**, porque se
  calcula a partir de ellas.
- **CA-05.5** Registrada una venta, **no existe forma de modificarla ni de anularla**: ni una
  operación del agregado, ni un método del repositorio, ni un verbo del API. Es la afirmación 2
  de §1 y se verifica sobre la **forma** del sistema, no sobre una llamada concreta (RN-07).

### HU-06 — Reporte de ventas por rango de fechas

> Como **responsable**, quiero saber cuántas unidades y cuánto dinero generó cada producto en un
> período, para decidir qué reponer.

- **CA-06.1** Dado un rango, entonces obtengo **una fila por producto y etiqueta de categoría
  congelada**, con unidades vendidas e importe, más el número de ventas y el total general del
  período. Sin recategorizaciones dentro del rango —el caso normal— eso es exactamente una fila por
  producto. **Con** ellas son dos, y es deliberado: colapsarlas exigiría decidir que dos etiquetas
  son la misma cosa, y esa decisión le tocaba a quien renombró, no al reporte
  ([`data-model.md`](data-model.md) §11.1).
- **CA-06.2** Dado un rango sin ventas, entonces obtengo un reporte vacío, **no un error**.
- **CA-06.3** Dado un producto que se dio de baja después de haberse vendido, entonces **sigue
  apareciendo** en el reporte del período en que se vendió.
- **CA-06.4** Dado un reporte de un período ya transcurrido, entonces **repetir la consulta
  devuelve exactamente lo mismo**, aunque el catálogo haya cambiado entretanto.
- **CA-06.5** La agregación se resuelve **en la base de datos**. Traer las ventas a memoria para
  sumarlas es un defecto, no una alternativa.

### HU-07 — Autenticación y protección

> Como **negocio**, quiero que solo el personal autorizado use el sistema, para que el
> inventario y las ventas no los toque cualquiera.

- **CA-07.1** Dadas credenciales válidas, entonces recibo un token con su vencimiento, mi
  usuario y mi rol. **Nunca el hash de la clave.**
- **CA-07.2** Dadas credenciales inválidas, entonces recibo un rechazo que **no revela** si
  falló el usuario o la contraseña.
- **CA-07.3** Dada una petición sin token a un endpoint protegido, entonces se rechaza.
- **CA-07.4** Dado un token de vendedor en una operación de administrador, entonces se rechaza
  por permiso, no por autenticación.
- **CA-07.5** El sistema **nunca** almacena ni registra la contraseña en claro.
- **CA-07.6** Dos usuarios no pueden compartir nombre de usuario, **ni siquiera si se registran
  a la vez**, ni si sus nombres solo difieren en mayúsculas o acentos (RN-10).
- **CA-07.7** Dado un rol que no es administrador ni vendedor, entonces el alta del usuario **se
  rechaza**. No hay un tercer rol, ni un rol vacío, ni un rol por omisión que se cuele (RN-11).

### HU-08 — Operar el sistema

> Como **evaluador**, quiero levantar el sistema completo con un comando y entender qué hace,
> para poder juzgarlo sin adivinar.

- **CA-08.1** Dado el repositorio clonado, un solo comando levanta base de datos, API y app.
- **CA-08.2** En el primer arranque, el esquema se crea solo, **sin ejecutar SQL a mano**.
- **CA-08.3** Existen categorías desde el arranque: sin ellas **no se puede crear ni un
  producto**, y el CRUD no se podría ni ejercer.
- **CA-08.4** El README explica qué es cada repositorio, cómo se levanta y **qué falta**.

---

## 4. Reglas de negocio

Se numeran para poder citarlas desde un test o desde una revisión.

**Una regla que nadie prueba no es una regla: es una intención.** Por eso cada fila declara cinco
cosas y no una — dónde vive, **quién la hace cumplir**, **qué prueba debe cubrirla**, **qué tarea
de [`tasks.md`](tasks.md) la entrega** y **qué criterio de aceptación la ejerce**—. **Convención
verificable:** el nombre de todo test que ejerce una regla lleva su identificador
(`test_rn_03_…`), de modo que `pytest -k rn_03` encuentra sus pruebas y el paso 12 de §8 puede
comprobar que ninguna regla se queda sin ellas.

Las columnas se leen así:

- **Quién la hace cumplir** — *agregado* (una guarda en el dominio), *motor* (una restricción de
  MySQL), *los dos*, o **nadie**.
- **Prueba exigida** — la capa: `D` = `tests/domain`, `A` = `tests/application` (puertos en doble),
  `I` = `tests/integration` (MySQL real), `X` = `tests/architecture`.
- **Criterio que la ejerce** — el criterio de aceptación de §3 que se apoya en la regla.

| # | Regla | Dónde vive | Quién la hace cumplir | Prueba exigida · tarea | Criterio que la ejerce |
|---|---|---|---|---|---|
| RN-01 | El stock de un producto **nunca** es negativo | Agregado `Product` | **Los dos** — `Product.withdraw` y el `CHECK ck_product_stock_non_negative` | `D` retirar más de lo que hay no deja el stock negativo · `A` la venta falla sin confirmar cuando el stock no alcanza · `I` el motor rechaza un stock negativo escrito por fuera del adaptador, y una actualización perdida no resucita stock ya vendido · T-02, T-10 | CA-04.1, CA-04.2, CA-04.6 |
| RN-02 | El precio de un producto es **mayor que cero** | Agregado `Product` | **Los dos** — `Product.change_price`, por donde pasa también la creación, y `CHECK (price > 0)` (T-20 punto 4). `Money` admite importe 0, así que sin el `CHECK` un precio 0 escrito por otra vía entraría sin resistencia | `D` un precio 0 se rechaza · `I` el motor lo rechaza · T-04, T-20 | CA-02.2 |
| RN-03 | La cantidad vendida es **mayor que cero** | Value object `Quantity` | **Los dos** — el constructor de `Quantity` y `CHECK (quantity > 0)` (T-20 punto 5) | `D` construir con 0 o con un negativo falla; con 1 conserva el valor · `I` el motor lo rechaza · **T-21**, T-20 | CA-04.5 |
| RN-04 | Una venta tiene **al menos una** línea | Agregado `Sale` | **Solo el agregado** — `Sale.ensure_confirmable`, invocado por `PlaceSaleService` | `D` rechaza una venta sin líneas · `A` el caso de uso la rechaza · T-10 | CA-04.3 |
| RN-05 | Un producto **no se repite** dentro de una misma venta | Agregado `Sale` | **Los dos** — `Sale.add_item` y el único compuesto `(sale_id, product_id)` (T-20 punto 3) | `D` añadir el mismo producto dos veces falla · `A` el caso de uso rechaza la línea repetida · `I` el motor rechaza el duplicado · T-10, T-20 | CA-04.4 |
| RN-06 | El precio y el nombre de la línea quedan **congelados** al vender | Agregado `Sale` | **Solo el agregado** — `SaleItem` copia nombre y precio al construirse y es inmutable | `D` añadir una línea toma el stock y congela el precio · `I` las líneas se reconstruyen con la cantidad y el precio de la venta · T-01, T-10 | CA-04.7 |
| RN-07 | Una venta registrada **no se modifica ni se anula**. No existe operación que lo permita | Modelo entero | **Nadie: la sostiene una ausencia.** `Sale` no expone quitar ni reemplazar líneas, su colección de líneas no se puede mutar desde fuera, el repositorio de ventas solo busca, lista y añade, y la ruta de ventas publica un `POST` y dos `GET`. Nada la defiende salvo la prueba estructural | `X` tres superficies: el agregado, el puerto de salida y la ruta HTTP (ningún `PUT`, `PATCH` ni `DELETE`). Una pasada futura que añada una de esas puertas debe ver una prueba en rojo · **T-22** | CA-05.5 |
| RN-08 | Un producto vendido **no se elimina**: se da de baja y desaparece del catálogo | Catálogo y persistencia | **Los dos** — la baja lógica (marca de baja solo en la persistencia, T-09), sin operación de borrado en el repositorio, y la clave foránea `ON DELETE RESTRICT` de `sale_item` hacia `product` (T-20 punto 2) | `I` dar de baja un producto vendido: desaparece del catálogo, no se puede vender, **la fila sigue existiendo** y la venta conserva nombre y precio congelados · `X` el repositorio no tiene borrado duro · `I` un `DELETE` a mano falla en el motor · **T-09**, T-20 | CA-02.5, CA-02.6, CA-01.4, CA-06.3 |
| RN-09 | Todos los importes de una venta están **en la misma moneda** | Value object `Money` y agregado `Product` | **Solo el agregado** — `Money` rechaza operar con dos monedas y `Product` rechaza un precio en moneda ajena por sus dos puertas (`create` y `change_price`), de modo que una venta mixta es inalcanzable | `D` operar dos monedas falla · `D` crear un producto o cambiarle el precio en otra moneda se rechaza · T-05 | Ninguno propio: §6 declara el alcance monomoneda |
| RN-10 | El nombre de usuario es **único** y se normaliza a minúsculas | Identidad y persistencia | **Los dos** — `User.normalize_username`, el índice único de `user.username` (la comparación del motor ignora mayúsculas **y acentos**: `jose` y `josé` son el mismo nombre) y el `CHECK` de forma normalizada (T-20 punto 8) | `D` `«  Ana  »` se guarda como `ana` · `A` rechaza un nombre ya tomado · `I` dos escritores que vieron libre el nombre dejan una sola fila · `I` quien pierde la carrera recibe una regla de negocio y no un error de base de datos · T-06, T-21 | CA-07.6 |
| RN-11 | El rol pertenece a un conjunto cerrado: administrador o vendedor | Agregado `User` | **Los dos** — el constructor de `User` (validación del rol) y `CHECK` sobre el rol (T-20 punto 7) | `D` un rol fuera del conjunto, vacío o por omisión falla; los dos válidos pasan · `I` el motor lo rechaza · **T-21**, T-20 | CA-07.7. CA-07.4 ejerce el rol en el borde HTTP, no el conjunto cerrado |
| RN-12 | El total de una venta es **siempre** la suma de sus líneas | Agregado `Sale` | **Solo el agregado, y por construcción:** `Sale.total` se calcula desde las líneas y **no hay columna de total** que pueda discrepar | `D` el total suma cada línea · T-10 | CA-05.4 |

### 4.1 · Las cuatro reglas con tarea propia

RN-03, RN-07, RN-08 y RN-11 no se cierran con la prueba obvia de su capa:

| Regla | Por qué necesita tarea propia | Quién la cierra |
|---|---|---|
| **RN-03** | Es una invariante de un value object: la prueba de dominio es barata y es fácil olvidarla | **T-21** |
| **RN-11** | Misma causa: la validación del rol vive en el constructor de `User` y hay que ejercerla | **T-21** |
| **RN-07** | Es una regla **estructural**: no se prueba ejercitando una operación, sino comprobando que **no existe ninguna** que la viole, en tres superficies (agregado, puerto, ruta HTTP) | **T-22** |
| **RN-08** | Exige construir la baja lógica entera y, además, la barrera del motor | **T-09**, con la barrera del motor en **T-20** |

> **RN-09 se satisface por construcción, no por una regla activa:** el sistema es monomoneda
> por cómo se persiste el importe (`plan.md` D-05), y la guarda de `Product` (T-05) cierra la
> única puerta por la que una moneda ajena podría entrar.

> **Las reglas que solo viven en el agregado no protegen los datos, solo la aplicación.** Cinco de
> ellas —RN-02, RN-03, RN-11, la normalización de RN-10 y el nombre no vacío de la categoría— las
> salta cualquier `INSERT` hecho a mano contra la base. Su barrera en el motor se escribe a mano en `initial_schema` (T-02; Alembic no autogenera los `CHECK`) y **T-20** enumera cada una con su definición exacta y la prueba violándola.

---

## 5. Requisitos no funcionales

| # | Requisito | Criterio de aceptación |
|---|---|---|
| RNF-01 | **Integridad bajo concurrencia** | Dos ventas simultáneas del último ejemplar no dejan stock negativo (CA-04.6) |
| RNF-02 | **Fidelidad histórica** | Un reporte de un período cerrado no cambia porque el catálogo cambie (CA-06.4) |
| RNF-03 | **Paginación** | Toda lista que pueda crecer sin límite viene paginada, con un tope de tamaño de página |
| RNF-04 | **Arranque reproducible** | Desde cero a sistema usable con un comando, sin pasos manuales |
| RNF-05 | **Trazabilidad** | Registro estructurado de cada petición, con correlación, sin datos personales ni secretos |
| RNF-06 | **Portabilidad del almacenamiento de binarios** | Cambiar disco local por un servicio de objetos es un adapter nuevo y una línea de composición |
| RNF-07 | **Pruebas** | Invariantes con test unitario; casos de uso con puertos en doble; persistencia contra motor real |
| RNF-08 | **Confidencialidad y retención** · **alcance añadido** | Cada atributo tiene una clasificación declarada y un manejo exigido: qué puede salir en una respuesta, qué puede aparecer en un registro y qué no se borra nunca. Ver [`data-model.md`](data-model.md) §7. **El enunciado no lo pide:** es el único requisito del spec que no sale del reto, y se conserva a sabiendas porque el sistema guarda el nombre de quien vende. Decisión del propietario, heredada del proyecto de referencia (2026-09-19) |

---

## 6. Fuera de alcance

No se modela, **porque nadie lo pidió**: clientes o compradores, devoluciones y anulaciones,
pagos y medios de pago, envíos, descuentos, impuestos, permisos más finos que los dos roles,
notificaciones, gestión de usuarios más allá del alta, y CRUD de categorías (son datos de
referencia sembrados).

**El producto tiene cinco atributos y no seis.** Nombre, precio, stock, categoría e imagen. Quedan
fuera **descripción, código de referencia y SKU**, y no por olvido: es la decisión DP-03 de §7.0.
Los puntos suspensivos del enunciado —*«nombre, precio, stock, categoria, imagen ...»*— se leen
como «y nada más», no como una invitación.

**Multimoneda operativa** queda fuera: la estructura la admite, pero el sistema opera en una
sola moneda y RN-09 lo exige por venta.

### El «Delete» del enunciado se cumple como baja lógica

El enunciado pide un CRUD, y la **D** de CRUD es *Delete*. Este sistema **no borra productos**:
los da de baja (ADR-003, T-09). Es una reinterpretación consciente y se declara aquí para que
nadie la lea como un requisito incumplido.

**Por qué.** Un producto vendido está referenciado por líneas de venta que congelan su nombre y su
precio. Borrarlo de verdad deja dos únicas salidas, y las dos son peores: romper el histórico de
ventas —lo que RN-07 prohíbe—, o fallar con un error que la persona usuaria **no puede resolver**,
porque no puede deshacer una venta ya registrada.

**Qué ve quien usa el sistema.** Exactamente un *Delete*: la acción se llama **Eliminar**, el
producto desaparece del catálogo, deja de poder venderse y no vuelve a aparecer en ninguna
búsqueda. Lo que cambia es lo que ocurre por debajo: la fila sobrevive para que su histórico
siga siendo consultable.

**Dónde se explica al evaluador:** en el README del servicio, no solo aquí. Una decisión que
contradice la letra del enunciado y solo vive en el spec es una decisión que nadie encontrará.

---

## 7. Decisiones de negocio

Marcadas explícitamente porque un spec que las esconde produce código que las decide en
silencio. Las cuatro están **cerradas**.

### 7.0 · Decididas

Las tomó el propietario del proyecto de referencia el **2026-09-19** y este proyecto las adopta
tal cual. **No se vuelven a abrir.** Cada una trae lo único que le importa a quien implementa:
**qué cambia en el código**.

| # | Pregunta | Decisión | Consecuencia para el código |
|---|---|---|---|
| **DP-01** | Si un producto se **renombra** entre dos ventas del mismo rango, ¿qué nombre muestra el reporte? | **El nombre congelado más reciente del rango.** El razonamiento que descartó las otras salidas está en §7.1, ya no como pregunta sino como el **porqué** de esta decisión | **T-08.** La consulta agregada agrupa por identificador de producto y toma el `product_name` de la **línea de la venta más reciente del rango**, con una función de ventana resuelta **en el motor**. La cláusula de ordenación se escribe con nombre, no disimulada: es el punto exacto donde se leería la decisión. El nombre vivo del catálogo **no se consulta nunca** para el reporte |
| **DP-02** | ¿El reporte debe poder desglosarse **por vendedor**? | **No.** Solo por producto. Evita cruzar datos personales del operador sin que nadie lo haya pedido | `SalesReportRow` queda **cerrada** en producto, categoría, unidades e importe. Ningún endpoint de reporte acepta filtro ni agrupación por vendedor. El índice `idx_sale_sold_by_user_id` existe **solo** porque la clave foránea FK-4 lo exige ([`data-model.md`](data-model.md) §6.3), y ese es el motivo de que no sirva a ninguna consulta. La autoría se sigue guardando: se registra, no se explota |
| **DP-03** | ¿El catálogo necesita descripción, código o referencia? | **No.** Solo los cinco atributos que el enunciado enumera: **nombre, precio, stock, categoría, imagen.** Los puntos suspensivos del enunciado se leen como *«y nada más»* | Los comandos de creación y edición y la vista de producto quedan **cerrados** con esos atributos. `product` no tiene más columnas que esos cinco atributos más `deleted_at` (T-09) y `version` (T-10), que no son atributos de negocio sino marcas de gestión solo de persistencia; ambas nacen en `initial_schema` (T-02). **El formulario de la app no crece.** Si alguien echa de menos un campo, la respuesta está escrita aquí |
| **DP-04** | ¿Un administrador puede crear otros administradores? | **No.** Un administrador da de alta **vendedores**; el rol `admin` lo provisiona el despliegue al arrancar, desde el entorno. Una sola vía de escalada de privilegios, y la controla quien despliega | **La restricción es estructural, no una comprobación.** El registro de usuarios rechaza el rol `admin` como violación de invariante de negocio; crear el primer administrador vive en una operación aparte que **el adaptador HTTP no expone**. La app ofrece `/vendedores/nuevo` solo al administrador, y **sin selector de rol**: no hay nada que elegir |

**El criterio que gobierna las cuatro es el mismo:** *entendimiento de negocio alineado únicamente
al reto*. Lo que el enunciado no pide, no entra — ni aunque sea barato, ni aunque sea habitual.

### 7.1 · DP-01 — por qué esa y no otra

**Este apartado no plantea una pregunta: justifica la decisión de §7.0.** Se conserva entero
porque el razonamiento es lo que hace que la decisión no se reabra por costumbre.

De las tres salidas posibles, **dos están prohibidas** por criterios escritos antes:

- **El nombre vivo del catálogo** queda descartado por CA-06.4 y RNF-02 —renombrar cambiaría un
  reporte ya cerrado— y por CA-06.3, porque obligaría a saltarse el filtro de bajas.
- **Una fila por cada nombre** queda descartada por CA-06.1, que pide *una fila por producto* (y por etiqueta de categoría congelada, §3).

Sobrevive **un nombre congelado del rango**, y el desempate es lo único que faltaba:

| Desempate | A favor | Coste |
|---|---|---|
| **El más reciente del rango** ✅ **elegido** | Es el nombre con el que el responsable reconoce el producto hoy | El nombre depende del corte de fechas: el mismo producto puede salir con nombres distintos en dos rangos que se solapan. **Coste aceptado a sabiendas** |
| El primero del rango | Estable hacia atrás: ampliar el rango por la derecha nunca cambia el nombre de una fila ya vista | Es el nombre más viejo — justo el que alguien quiso cambiar |
| El de más unidades vendidas | El nombre bajo el que el producto de verdad se movió en el período | Puede empatar, y entonces hay que desempatar el desempate |

**Las tres cuestan lo mismo de implementar** —una función de ventana en la consulta agregada—, así
que la decisión no se tomó por precio sino por a quién sirve el reporte: al responsable que decide
qué reponer, y que reconoce el producto por su nombre de hoy. **Y por lo mismo, cambiarla algún día
sería una cláusula de ordenación, no una migración.**

---

## 8. Criterio de terminado

**Un criterio de terminado que nadie ejecutó no es un criterio: es una lista de deseos.** Estos
pasos se ejecutan en orden, sobre un equipo limpio con Docker y nada más, y se deja constancia de
lo que cada uno devolvió. Es literalmente el guion de **T-18**.

Cada paso dice **qué se observa**, no qué se espera sentir.

| # | Paso | Se da por bueno cuando |
|---|---|---|
| **1** | **Clonar desde GitHub** los tres repositorios que se despliegan (`infra`, `api`, `app`) como hermanos y seguir su README | El README basta: no hay ningún paso que esté solo en la cabeza de quien lo escribió. El enunciado pide *repositorio clonable y ejecutable*, así que el punto de partida es el clon, no la carpeta de trabajo |
| **2** | **Un comando levanta todo** y la app responde | `docker compose up` deja los tres contenedores sanos y la app sirve su primera pantalla. El esquema se crea solo, sin SQL a mano (CA-08.1, CA-08.2) |
| **3** | **Un endpoint protegido rechaza a quien no presenta token** | `GET /api/products` sin token devuelve 401 **antes** de que nadie se autentique, y el alta de usuarios sin token también se rechaza: §2 dice que el anónimo solo puede autenticarse (CA-07.3) |
| **4** | **Un administrador entra**, crea un producto, le sube una imagen y lo ve en el catálogo | El producto aparece en la lista con su imagen, y solo con los cinco atributos de DP-03 (CA-02.1, CA-03.1) |
| **5** | **Un vendedor entra** y el catálogo lo deja mirar pero no tocar | Con token de vendedor, listar funciona y crear devuelve un rechazo **por permiso, no por autenticación** (CA-02.7, CA-07.4) |
| **6** | **El vendedor registra una venta** de ese producto y el stock baja exactamente | El stock del catálogo baja en la cantidad vendida, ni una unidad más (CA-04.1) |
| **7** | **El reporte del rango muestra esa venta** con sus unidades e importe | Una fila por producto, con unidades e importe, más el número de ventas y el total del período (CA-06.1) |
| **8** | **Dar de baja el producto lo saca del catálogo y el reporte lo sigue mostrando** | El producto desaparece de la lista, no se puede vender aunque se conozca su identificador, y su venta **sigue consultable** con el nombre y el precio del momento (CA-02.5, CA-02.6, CA-01.4, CA-06.3, CA-04.7) |
| **9** | **Repetir el paso 7 sobre el mismo rango devuelve exactamente lo mismo** | El reporte de un período cerrado no cambió porque el catálogo cambiara (CA-06.4, RNF-02) |
| **10** | **Las pruebas del servicio pasan enteras**, incluidas las de integración contra motor real, y también el análisis estático y la regla de dependencias | La suite completa, el linter, el chequeo de tipos y la regla de dependencias en verde, sin excluir capas y sin advertencias. Los comandos exactos están en el README de `simple-stock-flow-api` |
| **11** | **Las pruebas de la app pasan enteras** | La suite completa en verde, más linter y chequeo de tipos. El enunciado pide *testing*, y la app es la mitad de lo que entrega |
| **12** | **Las doce reglas de §4 tienen prueba**, y la tabla de §4 lo dice sin una sola celda que diga «ninguna» | Para cada RN-xx existe al menos un test cuyo nombre contiene su identificador (`pytest -k rn_xx`). Es el paso que ata la tabla de §4 con el código |
| **13** | **El README dice la verdad sobre todo lo anterior** | Ninguna afirmación del README es falsa, y lo que no corre está dicho en vez de omitido (CA-08.4, Artículo X) |
| **14** | **El análisis está entregable** | El enunciado pide *análisis/planteamiento lógico de la solución* como parte de la entrega, y eso son estos documentos: este spec, `plan.md`, `architecture.md`, `data-model.md`, `api-contract.md` y los ADR. Se da por bueno cuando ninguno contradice al código |

**Una cosa que este criterio no puede comprobar solo, y por eso se nombra:**

- **La sustentación** que el enunciado pide es una defensa oral. No se verifica con un comando:
  se prepara. Lo que estos catorce pasos garantizan es que hay algo que sustentar y que decir la
  verdad sobre ello no obliga a ningún rodeo.

---

## 9. Firma

**Qué se acepta.** El entendimiento de negocio de Simple Stock Flow: el problema (§1), los tres actores
(§2), las ocho historias con sus criterios de aceptación (§3), las doce reglas de negocio con su
trazabilidad completa (§4), los ocho requisitos no funcionales (§5), el alcance excluido (§6), las
cuatro decisiones de negocio cerradas (§7) y el criterio de terminado de catorce pasos (§8).

**Contra qué se verificará.** Contra el enunciado `enunciado/prueba-inventarios.docx` y contra el
código que corra, no contra la memoria de nadie:

- Cada regla de §4 se da por cubierta cuando existe una prueba cuyo nombre lleva su identificador
  (paso 12 de §8).
- «Nadie la hace cumplir» en RN-07 se sostiene con la prueba estructural de T-22 sobre las tres
  superficies: agregado, puerto de salida y ruta HTTP.
- RN-12 es cierta por construcción: no hay columna de total que pueda discrepar de la suma.

**Fecha.** 2026-10-02.

**Qué queda explícitamente fuera de esta firma.**

- **El modelo de datos** — vive en [`data-model.md`](data-model.md) y se firma aparte.
- **El contrato de API** — vive en [`api-contract.md`](api-contract.md) y se firma aparte. Este
  documento no describe ni una ruta ni un cuerpo de respuesta, a propósito.
- **Las decisiones técnicas** — viven en [`plan.md`](plan.md) y en [`architecture.md`](architecture.md).
- **El estado de avance.** Este documento dice qué debe hacer el sistema, no cuánto está hecho.
  Eso vive solo en la tabla de progreso de [`tasks.md`](tasks.md).
