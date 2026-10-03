# ADR-004 — El reporte agrega en MySQL sobre atributos congelados

**Estado:** Aceptada (la pregunta DP-01 que destapa está decidida: ver el final) · **Fecha:** 2026-10-02

## Contexto

El reporte de ventas por rango es el entregable funcional de más peso del enunciado. Necesita,
por producto: unidades, importe, nombre y categoría. Dos problemas:

1. Un puerto de ventas que devuelva **agregados de venta completos** para un rango, sin paginar,
   obliga a cargar todas las ventas del período en memoria — exactamente lo que CA-06.5 declara
   defecto.
2. El nombre de la categoría **no es alcanzable** desde la venta si la línea solo congela el
   nombre del producto. Y el servicio del reporte no debe inyectar ningún repositorio de
   catálogo, así que **no podría resolver ese campo de ninguna manera**.

## Decisión

**Dos cosas, y las dos son necesarias:**

1. **Un puerto outbound de solo lectura** propio del reporte, `SalesReportQuery`, que devuelve las
   filas ya agregadas. El adapter lo implementa con una consulta SQLAlchemy que agrupa **en
   MySQL** (`GROUP BY` por producto y etiqueta de categoría congelada, con `SUM` de unidades y de `quantity * unit_price`). El
   servicio de aplicación no ve una sola entidad de venta. Los importes son `DECIMAL(12,2)`
   exactos y llegan como `decimal.Decimal`; el rango de fechas se compara contra `DATETIME(6)`
   en UTC.
2. **La línea de venta congela también el nombre de la categoría**, igual que congela nombre y
   precio del producto.

## Alternativas consideradas

| Alternativa | Por qué no |
|---|---|
| **Unir contra el catálogo vivo** para resolver la categoría | Recategorizar un producto **reescribiría reportes ya emitidos** de períodos cerrados. Viola CA-06.4 y RNF-02 |
| **Inyectar repositorios de catálogo en el servicio del reporte** | Le daría dos razones para cambiar y reintroduciría la lectura viva del punto anterior |
| **Guardar también el identificador de categoría** en la línea | Obligaría a unir contra la categoría para mostrar el nombre, que es la lectura viva que se quiere evitar. El reporte agrupa por la **etiqueta congelada** de la línea, no por un identificador de categoría vivo |
| **Tabla de reporte precalculada** | Un agregado almacenado que puede desincronizarse (artículo VII), para un volumen que no lo necesita |
| **Agregar en Python** sobre las ventas cargadas | Carga todo el período en memoria (CA-06.5) y mueve el cálculo lejos del motor, que lo hace mejor |

## Consecuencias

**Positivas**

- El reporte de un período cerrado es **estable por construcción**, no por disciplina.
- La consulta toca dos tablas (venta y línea) en vez de cuatro.
- El servicio del reporte mantiene una sola responsabilidad.

**Negativas, declaradas**

- **Cambia la firma de un método del agregado**: añadir una línea pasa a requerir la categoría
  además del producto, y el caso de uso de venta gana una dependencia para resolverla. El coste
  no desaparece: se mueve del camino de lectura al de escritura, que es donde el dato se conoce.
- El nombre de la categoría se repite en cada línea. **No es desnormalización**: dos líneas del
  mismo producto pueden legítimamente tener categorías distintas si se recategorizó entre ambas
  ventas, luego no existe la dependencia funcional que haría transitiva la repetición.
- Con `utf8mb4_0900_ai_ci`, un `GROUP BY` sobre un nombre agrupa "Café" y "cafe" como iguales.
  Por eso el reporte agrupa por **identificador de producto** (y por la etiqueta de categoría congelada, CA-06.1), no por el nombre del producto.

## Pregunta que esta decisión hizo visible (DP-01, decidida)

**DP-01.** Con los nombres congelados, un producto **renombrado** entre dos ventas del mismo rango
produce dos valores distintos. Agrupar por producto y nombre devuelve **dos filas del mismo
producto**; agrupar solo por producto obliga a **elegir** un nombre. Ninguna de las dos opciones es
neutral, y la elección es de negocio.

**Decidida** ([`spec.md`](../spec.md) §7.0): la implementación toma el nombre congelado de la
venta más reciente del rango y lo deja escrito en el código como la decisión que es — no
disimulada dentro de una consulta.

**La pregunta es más pequeña de lo que parece.** De las dos salidas que el párrafo anterior
presenta como equivalentes, **una está prohibida**: agrupar por producto y nombre devuelve dos
filas del mismo producto, y CA-06.1 pide *una fila por producto y etiqueta congelada*. Unir contra el catálogo vivo
tampoco es una salida — CA-06.4 y RNF-02 lo cierran, que es justamente lo que este ADR decide.
Queda por elegir **cuál** de los nombres congelados del rango, y eso no bloquea la
implementación: es una cláusula de ordenación. Análisis completo en
[`spec.md` §7.1](../spec.md).
