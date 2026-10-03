# ADR-002 — Concurrencia optimista sobre el stock, con columna `version` solo en persistencia

**Estado:** Aceptada · **Fecha:** 2026-10-02

## Contexto

`Product.withdraw` garantiza que el stock no quede negativo **en memoria**. Entre que el caso de
uso lee los productos y confirma la transacción hay una ventana en la que otra venta puede
retirar del mismo producto: **ambas leen el mismo stock, ambas pasan la invariante, y la base
queda con stock negativo** aunque el dominio se haya cumplido en las dos.

CA-04.6 lo declara inaceptable: dos ventas simultáneas del último ejemplar deben resolverse con
una confirmación y un conflicto.

## Decisión

**Control optimista** con una columna **`version INT NOT NULL`** en la tabla de productos.
MySQL/InnoDB no ofrece un token de versión por fila que el motor mantenga solo (no hay
equivalente a `xmin`), así que la versión es una columna explícita, **gestionada por el adaptador
de persistencia**. Aislamiento por defecto del motor (`REPEATABLE READ`).

- La columna existe **solo en el modelo de persistencia** SQLAlchemy, declarada con
  `version_id_col`. **No existe en la entidad de dominio** `Product`: el dominio no sabe que hay
  concurrencia.
- SQLAlchemy emite `UPDATE ... SET ..., version = version + 1 WHERE id = ? AND version = ?`. Si
  no afecta a ninguna fila, lanza `StaleDataError`.
- **El adaptador traduce `StaleDataError` a `ConcurrencyConflict`**, una excepción **definida por
  la capa de aplicación**. El tipo del ORM no sale del adapter: es infraestructura, y si llegara a
  la aplicación la regla de dependencia estaría rota.
- Ante `ConcurrencyConflict` se **reintenta el caso de uso completo desde la lectura**, máximo
  **tres** intentos con espera irregular. Agotados, la API responde **409**.
- Restricción `CHECK (stock >= 0)` en la base (real desde MySQL 8.0.16) como **última barrera**:
  si salta, algo escribió fuera del adapter, y debe verse, no capturarse en silencio.

## Alternativas consideradas

| Alternativa | Por qué no |
|---|---|
| **Versión incrementada a mano por cada ruta de escritura** (sin `version_id_col`) | Exige que **todas** las rutas la incrementen. El día que una migración, una semilla o un adapter nuevo la olvide, el mecanismo queda desactivado en silencio. Con `version_id_col` la incrementa el ORM en toda actualización hecha a través de él; las escrituras SQL directas siguen siendo responsabilidad de quien las escribe |
| **Versión en la entidad de dominio** | Filtra un detalle de persistencia al dominio y rompe el artículo I en espíritu: el agregado ganaría un campo que no es del negocio |
| **Bloqueo pesimista** (`SELECT ... FOR UPDATE`) al leer los productos | El caso de uso lee N productos por identificador: bloquearlos durante toda la venta **serializa ventas que no comparten ningún producto**. Además convertiría un puerto de lectura en una lectura que bloquea, algo que su firma no expresa: el bloqueo se filtraría al contrato |
| **Aislamiento `SERIALIZABLE`** | Convierte el mismo conflicto en un deadlock o fallo de serialización que también hay que reintentar — mismo coste — pero imponiendo bloqueo a **todas** las lecturas, incluidas las del reporte, que no tienen nada que evitar |
| **Decremento condicional en una sola sentencia** (`UPDATE ... SET stock = stock - ? WHERE stock >= ?`) | Correcto y más barato, pero **mueve la decisión de negocio al SQL**: el dominio dejaría de ser la autoridad sobre su invariante. Se descarta por frontera, no por rendimiento. Es la alternativa a reconsiderar si el reintento resultara caro |

## Consecuencias

**Positivas**

- El dominio queda intacto: la versión es un detalle del modelo de persistencia.
- El mecanismo es explícito y legible en un cliente SQL (`SELECT id, stock, version FROM ...`).

**Negativas, declaradas**

- Una columna más en el esquema, que migraciones y semillas deben inicializar (`version = 1`).
- `version` cambia ante **cualquier** actualización de la fila, incluido un renombrado o un cambio
  de imagen: produce conflictos que una versión ligada solo al stock no produciría. El reintento
  los absorbe.
- Una escritura SQL fuera del ORM que no incremente `version` pasa desapercibida para el
  mecanismo; por eso el `CHECK (stock >= 0)` queda como última barrera.
- Reintentar **solo la escritura** en vez del caso de uso completo reaplicaría un descuento
  calculado sobre un stock viejo. Es el error más probable al implementarlo, y por eso queda
  escrito aquí.
