# Constitución — Simple Stock Flow

Principios innegociables del proyecto. No son estilo ni preferencia: cada artículo trae
**cómo se verifica**, porque un principio que nadie puede comprobar es una intención.

Un cambio que viole cualquiera de estos artículos se rechaza sin entrar a discutir el resto.

---

## Artículo I — El dominio no depende de nada

`src/stockflow/domain/` no importa ningún otro paquete del proyecto ni ninguna librería de
terceros. Ni SQLAlchemy, ni FastAPI, ni pydantic, ni `application/`. Solo biblioteca estándar:
`dataclass(frozen=True)` para los value objects y `decimal.Decimal` para el dinero. Ni un
decorador de framework, ni un `import` externo, ni una anotación de persistencia.

**Por qué:** es lo único que hace que las invariantes de negocio se puedan probar sin base de
datos, sin servidor y sin contenedor — y lo único que permite cambiar de ORM sin tocar reglas.

**Verificación:** un contrato de `import-linter` prohíbe que `stockflow.domain` importe
cualquier otro módulo del proyecto. Un test de arquitectura en `tests/architecture/` recorre los
`import` de `domain/` y falla si encuentra algo fuera de la biblioteca estándar.

---

## Artículo II — El núcleo declara sus puertos; el mundo los implementa

`src/stockflow/application/` solo importa el dominio. Todo lo que necesita del exterior lo
expresa como un puerto (`typing.Protocol` o ABC, sin prefijo `I`) que **él mismo define** en
`ports/outbound/`. Ningún tipo de infraestructura —`Session` de SQLAlchemy, `Request` de FastAPI,
`UploadFile`, `StaleDataError`— aparece jamás en la firma de un puerto.

**Por qué:** es la inversión de dependencia real. Si el puerto habla el lenguaje del adapter,
el núcleo ya depende de la infraestructura aunque haya una abstracción de por medio.

**Verificación:** un contrato de `import-linter` de tipo `layers` impide que `application`
importe `adapters` o `bootstrap`. Los tests de `tests/application/` corren con fakes y **sin una
sola referencia a `adapters/`**.

---

## Artículo III — Un único punto de composición

`src/stockflow/bootstrap/composition/port_bindings.py` es el único módulo donde un puerto se
encuentra con su adapter. Fuera de ahí no se instancia infraestructura.

**Por qué:** si hay dos sitios donde se decide quién implementa qué, nadie puede responder con
certeza qué corre en producción.

**Verificación:** ninguna instanciación de una clase de `adapters/` fuera de `port_bindings.py`
y de los propios adaptadores. Se comprueba con un test de arquitectura (o un `grep`), y por eso
se puede automatizar.

---

## Artículo IV — SOLID no es decorativo

| Principio | Aplicación concreta en este proyecto | Verificación |
|---|---|---|
| **SRP** | Un servicio de aplicación = **un** caso de uso. `PlaceSaleService` registra ventas y no hace nada más | Un servicio con dos razones para cambiar se parte |
| **OCP** | Añadir un adapter (S3 en vez de disco, RabbitMQ en vez de nada) no toca ni el dominio ni la aplicación | El nuevo adapter solo añade una línea en `port_bindings.py` |
| **LSP** | Los fakes de test sustituyen al adapter real sin cambiar ni una expectativa del caso de uso | El mismo test de aplicación pasa contra fake y contra adapter real |
| **ISP** | Puertos pequeños y con un motivo: `Clock`, `PasswordHasher`, `TokenGenerator`. **Nunca** un puerto `Infrastructure` con quince métodos | Ningún implementador deja métodos sin usar o lanzando |
| **DIP** | El puerto lo define quien lo **consume** (`application`), no quien lo implementa (`adapters`) | Los módulos de `application/ports/outbound/` viven en el paquete de aplicación |

---

## Artículo V — El esquema es propiedad del servicio, no de la infraestructura

El repositorio de infraestructura levanta **un motor vacío**: contenedor, volumen, usuario y
base creada. Nada más. Las tablas, índices, restricciones y datos de referencia los crea el
servicio mediante **migraciones de Alembic**, que viven en el adapter de persistencia
(`src/stockflow/adapters/outbound/persistence/migrations/`).

**Por qué:** si el DDL vive en infraestructura, el esquema y el código que lo usa se despliegan
por separado y se desincronizan. El modelo es del servicio; la migración también.

**Verificación:** el repo de infra no contiene ni un `.sql` de DDL. Una base recién creada y
vacía queda completa ejecutando únicamente el servicio (`alembic upgrade head` al arrancar).

---

## Artículo VI — Las invariantes viven en el agregado

Ninguna regla de negocio en un router, en un servicio de aplicación ni en una cláusula SQL.
El servicio de aplicación **orquesta** puertos y delega toda decisión al dominio.

**Por qué:** una invariante duplicada en dos capas diverge. Una invariante que vive en un
`WHERE` deja de ser del negocio y pasa a ser del motor.

**Verificación:** un `if` de negocio en `application/services/` es motivo de rechazo. Toda
invariante tiene un test en `tests/domain/` que la ejerce **sin dobles de prueba**.

---

## Artículo VII — Lo derivado se calcula, nunca se almacena

`Sale.total` y `SaleItem.subtotal` se calculan. No hay columna, ni columna generada, ni trigger
que los mantenga. Lo mismo para cualquier agregado del reporte.

**Por qué:** un total almacenado es un total que algún día no coincide con sus líneas, y nadie
sabe cuál de los dos miente.

**Verificación:** el esquema no tiene columnas `total` ni `subtotal`. La suma se prueba contra
las líneas, no contra sí misma.

---

## Artículo VIII — La prueba va antes que la implementación

Cada invariante y cada caso de uso se escriben como test que **falla primero**. Tres niveles,
con fronteras que no se cruzan:

- `tests/domain/` — invariantes puras, **sin dobles de prueba**.
- `tests/application/` — casos de uso con puertos outbound en fake, **sin infraestructura**.
- `tests/integration/` — contra MySQL real vía Testcontainers (exige Docker).

Además, `tests/architecture/` hace cumplir los artículos I, II y III.

**Por qué:** un test escrito después de ver pasar el código prueba lo que el código hace, no lo
que debía hacer.

**Verificación:** el test existe en el commit que introduce el comportamiento, no en el
siguiente.

**También para la infraestructura.** Un `docker compose` es código: se escribe **después** de la
comprobación que lo juzga. Primero el guion de verificación (`verify.sh`) —que falla porque no
hay nada que levantar—, después el compose, hasta que pase. La infraestructura que nadie
comprueba es la que funciona en la máquina de quien la escribió.

**No se escribe producción sin su prueba roja delante. Sin excepciones, tampoco para "esto es
trivial".**

---

## Artículo IX — Ningún secreto en el repositorio

Cadena de conexión, clave de firma del token y credenciales del usuario inicial
(`DATABASE_URL`, `JWT_SIGNING_KEY`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`) vienen de configuración de
entorno. No hay valor por defecto para un secreto: si falta, **el arranque falla ruidosamente**.

**Por qué:** un valor por defecto para un secreto es un secreto publicado, y además silencia
justo el error que querías ver.

**Verificación:** la clase de configuración (`pydantic-settings`) declara los secretos sin valor
por defecto. Un test arranca la configuración sin `JWT_SIGNING_KEY` y exige un error explícito,
no una clave de relleno. `.env` está ignorado y `.env.example` no contiene valores reales.

---

## Artículo X — El estado declarado es el estado real

Lo que no está implementado lanza y **dice qué falta**. El README no promete lo que no corre.
Un endpoint que no funciona no se documenta como si funcionara.

**Por qué:** en una entrega evaluable, la diferencia entre un pendiente declarado y una promesa
incumplida es toda la credibilidad del entregable.

**Verificación:** cada `NotImplementedError` trae el `TODO` de qué hacer, y la sección "Qué
falta" del README los lista. El estado del sistema vive solo en la tabla de progreso de
`tasks.md`. Si el README y el código discrepan, gana el código y el README se corrige.

### X.1 · Un número escrito a mano no tiene quien lo vigile

Un documento de estado se pudre más rápido cuanto mejor va el proyecto: las cifras que declara
caducan precisamente porque el trabajo avanza.

Tres reglas, en orden de coste:

1. **Una cifra que crece con el trabajo no se escribe: se publica el comando que la produce.**
   `pytest` no caduca; «264 tests» caduca al día siguiente.
2. **Un cero no es un recuento, es una invariante.** «0 warnings», «0 failed» y «0 skipped» pueden
   escribirse: no crecen, y el día que dejan de ser ciertos es justo cuando queremos enterarnos.
3. **Si una cifra tiene que estar escrita** —porque alguien la lee sin poder ejecutar nada—
   **algo la recalcula y falla si no cuadra.** El recuento de progreso de `tasks.md` está permitido
   solo porque `verify.sh` lo recompone desde la propia tabla.

Y su corolario para los defectos: **cada defecto abierto lleva al lado la sonda que lo cerraría**,
escrita antes de arreglarlo. Sin ella, «cerrado» es una opinión de quien lo tocó.

**Verificación:** `verify.sh` §8 aplica estas comprobaciones sobre los documentos. Cada
comprobación se prueba **rompiendo los documentos a propósito**: debe fallar nombrando el
documento y la línea, y volver a pasar al restaurarlos. Una comprobación que nunca ha fallado no
se sabe si funciona.

### X.2 · La sonda se escribe **antes** del arreglo

`verify.sh` §8 comprueba **la forma, no la verdad**: exige que cada defecto abierto lleve al lado la
sonda que lo cerraría, pero **una sonda inventada pasaría igual**. Ningún guion puede cerrar ese
hueco, porque para cerrarlo habría que saber de antemano qué es cierto.

Lo que sí lo cierra es **el orden**. La sonda se escribe cuando todavía no sabes qué va a devolver:
antes de tocar el defecto. Escrita después, se parece sospechosamente a lo que acabas de hacer —es
el artículo VIII, el test rojo primero, aplicado a los defectos en vez de a las funciones—.

**Verificación:** ninguna automática, y por eso está escrito aquí. Si la sonda de un defecto aparece
en el mismo cambio que su arreglo, no demuestra nada.

### X.3 · Un documento firmado no se corrige: declara su deuda

Un documento firmado no lo reescribe quien pasa por ahí, así que **envejece mintiendo**. La salida no
es romper la firma ni poner un comprobador que falle todos los días hasta que alguien decida —una
comprobación que siempre falla se aprende a ignorar—.

La salida es un **registro de deuda declarada**: una lista fechada donde cada entrada dice **qué
afirma el documento**, **qué mide la realidad** y **qué corrección se propone**, más una marca en el
sitio exacto donde el lector se topa con la afirmación falsa. El documento firmado no cambia lo que
dice; deja de decirlo **en silencio**.

**Una mentira declarada es deuda. Una mentira silenciosa es una trampa.**

**Verificación:** `verify.sh` §9. No comprueba que las afirmaciones sean ciertas —eso exige medir—:
comprueba que el registro **exista, esté fechado, esté completo** y que **marcas y entradas se
correspondan una a una**. Falla cuando alguien declara una deuda donde nadie la leerá, o marca una
afirmación sin decir qué es cierto en su lugar.

---

## Artículo XI — Dos idiomas, una frontera clara

**Una sola pregunta decide: ¿quién lo lee?**

| Quien lo lee | Idioma | Qué incluye |
|---|---|---|
| La persona que **usa** el sistema | **Español** | Textos de la interfaz · mensajes de negocio que viajan en una respuesta 422 o 409 · validaciones que llegan al cliente · las rutas de la URL |
| La persona que **mantiene o despliega** | **Inglés** | Identificadores · comentarios · **nombres de prueba** · tablas, columnas e índices · **salida de scripts** · **diagnósticos de arranque y configuración** · mensajes de log · commits y ramas |
| Quien **evalúa el entregable** | **Español** | README de cada repositorio y todo `simple-stock-flow-docs/` |

**Por qué esta frontera y no otra:** el dominio lanza sus excepciones con un mensaje pensado para
que el usuario lo lea —"El stock inicial no puede ser negativo"— y ese texto viaja tal cual hasta
la respuesta HTTP. Todo lo demás lo lee quien mantiene, y ahí el inglés evita la mezcla que obliga
a traducir mentalmente en cada lectura.

> **Una frontera con excepciones no es una frontera.** Si hay que razonar caso por caso, cada
> persona razonará distinto y el repositorio acabará mezclado. La pregunta «¿quién lo lee?» se
> responde sin discutir; «¿esto cuenta como diagnóstico?» no. Un archivo entero en un solo idioma
> es mejor que uno mezclado.

**El idioma no basta: también hay que escribirlo.** El español que lee una persona va **con sus
tildes y su eñe** —"Rol no válido", "La sesión expiró", "Fontanería"—. Es lo primero que lee quien
evalúa. Un mensaje mal escrito no deja de ser un defecto por estar en el idioma correcto.

Del mismo grupo, y por el mismo motivo: **el documento declara el idioma en el que está.** La app
sirve `<html lang="es">`, que es lo que leen los lectores de pantalla y lo que evita que el
navegador ofrezca una traducción innecesaria.

**Verificación:** un comentario en español es una violación. Un nombre de prueba en español,
también. Un mensaje de excepción de dominio en inglés, también. Y un texto de usuario sin acentos,
también — se comprueba con `grep -rnP "[áéíóúñ]"`: si no devuelve **nada** en un repositorio que
tiene texto en español, el texto está mal escrito, no limpio.

**Cómo se barre, que es donde falla la comprobación.** El barrido **enumera archivos, no
extensiones**. Una lista de extensiones deja fuera justo los archivos de configuración y de
construcción (`pyproject.toml`, `Dockerfile`, `.conf`, `.ini`, `.yml`), donde también se cuelan
comentarios en el idioma equivocado.

```bash
# Correcto: enumera todo y filtra después.
find . -type f -not -path "*/node_modules/*" -not -path "*/.git/*" \
  -not -path "*/__pycache__/*" -not -path "*/.venv/*" -not -path "*/dist/*" | while read f; do ...
```

> **Una lista de extensiones es una lista de sitios donde ya sospechabas.** El defecto vive
> justamente en el tipo de archivo que no se te ocurrió mirar.

**Y se busca en los dos idiomas, no solo en el del código.** Un barrido de referencias huérfanas
a un componente eliminado debe buscar el término inglés (`messaging|scheduler`) **y** el español
(*"mensajería y planificador"*). El término inglés está en el código; el español, en la
documentación. Quien busque uno solo se cree limpio a mitad.

**Y el patrón se escribe sin acentos.** `grep -i "submodul"` **no encuentra** "submódulo", y
`subm.dul` tampoco: en locale C la `ó` ocupa **dos bytes**, así que el `.` se queda corto. Buscar
una palabra acentuada y no encontrarla **no prueba nada**. Se busca siempre por el fragmento sin
acento más largo —`grep -i "dulo de"`— o con `subm..dul`.

---

## Artículo XII — Comentarios sutiles, y solo los necesarios

**Un comentario explica *por qué*. Nunca *qué*.** Si hace falta un comentario para entender qué
hace una línea, el problema es la línea.

**Se escriben:**

- La razón de una decisión que el código no puede expresar: por qué se congela un precio, por qué
  se anula la clave de imagen antes de borrar el binario, por qué el reintento rehace el caso de
  uso completo.
- La advertencia que evita que alguien "arregle" algo a peor.

**No se escriben:**

- Los que repiten la firma: `# Crea un producto` sobre una función que se llama `create_product`.
- Los separadores decorativos que trocean un archivo largo. **Si un archivo necesita letreros para
  orientarse, lo que necesita es partirse en funciones con nombre.**
- Los `TODO` sin acción concreta. Un `TODO` dice qué hacer, o no existe.

**Por qué:** cada comentario es una segunda fuente de verdad que nadie ejecuta y que envejece sin
avisar. El código que cambia sin su comentario deja una mentira firmada.

**Verificación:** un comentario que se puede borrar sin perder información **debe** borrarse.
`ruff` hace cumplir la parte mecánica (`ERA` para código comentado, `T20` para `print`).

---

## Artículo XIII — Cada repositorio se explica a sí mismo

Todo repositorio lleva un README que responde **cinco preguntas, en este orden**, y ninguna se da
por obvia:

| # | Pregunta | Qué tiene que decir, en concreto |
|---|---|---|
| 1 | **Qué es esto** | Una frase. Qué hace y de qué **no** se ocupa |
| 2 | **Cómo se levanta** | Los comandos exactos, copiables, desde un equipo limpio |
| 3 | **Dónde están los datos** | **Nombre de la base, usuario, puerto y de dónde salen las credenciales.** Cómo conectarse con un cliente para mirarlos con los propios ojos |
| 4 | **Cómo se prueba** | El comando, y qué necesita: Docker sí o no |
| 5 | **Qué falta** | Lo que no está hecho, dicho por su nombre (artículo X) |

**La tercera es la que casi nadie escribe y la primera que hace falta.** Quien evalúa un
entregable quiere abrir la base y ver las filas; si tiene que deducir el nombre de la base de un
`docker-compose`, el README no ha hecho su trabajo.

**Los valores viven en `.env`, nunca en el README ni en el código.** El repositorio versiona un
`.env.example` con **todas** las claves y sin un solo valor real; el `.env` está ignorado. El
README explica qué variable hace qué y cuál no tiene valor por defecto a propósito.

**Y no lleva más documentación que ese README.** La carpeta `docs/` es de `simple-stock-flow-docs`
y de ningún otro repositorio. Un repositorio de código documenta **lo suyo**, en su README, y
nada más. Un archivo que solo apunta a otro repositorio no explica nada y se rompe cuando alguien
clona ese repositorio suelto —que es exactamente lo que hará quien evalúe—.

> **La cura de una duplicación no es un puntero: es decidir de quién es el documento.** Si el
> documento habla de varios repositorios, no es de ninguno de ellos; es del repositorio de
> documentación. Si habla de uno solo, es su README.

**Verificación:** una persona que nunca vio el proyecto levanta el sistema y consulta la base
siguiendo **solo** el README. Y `find . -name docs -type d` en un repositorio de código no
devuelve nada. Si necesita preguntar algo, eso que preguntó le falta al README.
