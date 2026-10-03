# ADR-003 — Baja lógica de producto con `deleted_at` solo en persistencia

**Estado:** Aceptada · **Fecha:** 2026-10-02

## Contexto

El enunciado pide CRUD de productos, la "D" incluida. Pero la línea de venta referencia al
producto, y el reporte histórico depende de que esa fila siga existiendo. Borrar físicamente un
producto vendido rompe el histórico; impedir el borrado deja al usuario con un botón que falla
justo para los productos que importan: los que se vendieron.

## Decisión

**Baja lógica.** Una columna `deleted_at DATETIME(6) NULL` (UTC), nula mientras el producto está
activo.

La columna **no existe en la entidad de dominio**: solo en el **modelo de persistencia**
SQLAlchemy, que es distinto de la entidad y se convierte con mappers en el adapter. La operación
de baja pasa a ser un `UPDATE`; el borrado físico desaparece del camino de la aplicación, y la
clave foránea de la línea hacia el producto queda como barrera de última instancia para que un
borrado manual falle ruidosamente.

**El filtro está centralizado en la base de los repositorios.** SQLAlchemy no tiene filtro global
de consulta: la seguridad por omisión se logra con **un único método base** (p. ej.
`_active_products()`, que devuelve `select(ProductModel).where(ProductModel.deleted_at.is_(None))`)
del que parten todas las consultas del repositorio. Ninguna consulta de producto construye su
`select` por fuera. Una consulta que **no** debe filtrar bajas parte de otro método base con
nombre explícito (`_all_products_including_deleted()`), de modo que la excepción tenga que
escribirse a mano.

**Contrato obligatorio de los repositorios:**

| Operación | Filtra bajas | Motivo |
|---|---|---|
| Buscar catálogo | **Sí** | Un producto retirado no se ofrece (CA-01.4) |
| Cargar varios por identificador, previo a vender | **Sí** | Sin esto **se puede vender un producto retirado** (CA-02.6) |
| Cargar uno por identificador | **No** | Una línea de venta histórica debe poder resolver su producto |

**Cómo se verifica:** tests de integración (MySQL real) por cada fila de la tabla anterior: un
producto dado de baja no aparece en el catálogo, no se puede vender y sí se resuelve por
identificador. Un test de arquitectura exige que `select(ProductModel)` aparezca únicamente en los
métodos base del repositorio.

## Alternativas consideradas

| Alternativa | Por qué no |
|---|---|
| **Borrado físico con clave foránea restrictiva** | La "D" del CRUD fallaría exactamente para los productos con ventas, y el usuario recibiría un error que no puede resolver por sí mismo |
| **Marca booleana** en vez de fecha | El booleano no responde *cuándo* se dio de baja, y se indexa mal: dos valores efectivos. La fecha nula permite además columnas generadas y unicidad condicionada (ver Consecuencias) |
| **Propiedad real en la entidad de dominio** | El dominio ganaría un concepto que es de persistencia. Es defendible verlo como regla de negocio —"un producto vendido se retira, no se elimina"—, pero añadirlo a la entidad tendría mayor radio de cambio sin beneficio observable |
| **Filtrar en cada consulta a mano** | Confiar en no olvidarlo es exactamente cómo se cuela la venta de un producto retirado |
| **Evento `do_orm_execute` con `with_loader_criteria`** como filtro global automático | Reproduce el filtro global de otros ORM, pero es magia implícita y difícil de excluir en la consulta que no debe filtrar; se prefiere el método base explícito, más legible y fácil de probar. Es la alternativa a reconsiderar si crecen las consultas |

## Consecuencias

**Positivas**

- El dominio no se entera: `Product` no gana ni un atributo.
- El método base hace que **lo seguro sea lo que ocurre por omisión** y la excepción tenga que
  escribirse a mano.

**Negativas, declaradas**

- Las filas no se recuperan nunca: la tabla solo crece. Irrelevante al volumen previsto.
- **MySQL no tiene índices parciales.** Un índice sobre el catálogo no puede llevar el predicado
  `deleted_at IS NULL`. Sustituto: un índice compuesto que arranque por `deleted_at` (los activos
  quedan agrupados) y consultas que incluyan el predicado literalmente.
- **Regla para el futuro:** cualquier restricción de unicidad sobre el producto debe aplicar solo
  a los activos, o un producto retirado reservaría su código para siempre. Como no hay índice
  único parcial, se usa una **columna generada** más un índice único, p. ej.
  `active_name VARCHAR(...) GENERATED ALWAYS AS (IF(deleted_at IS NULL, name, NULL)) STORED` con
  `UNIQUE (active_name)`: los `NULL` no chocan en un `UNIQUE`.
- La colación `utf8mb4_0900_ai_ci` es insensible a mayúsculas **y acentos**: "Café" y "cafe"
  colisionan en esa unicidad. Es el comportamiento buscado y queda declarado.
