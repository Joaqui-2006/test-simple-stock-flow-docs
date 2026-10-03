# ADR-001 — El esquema pertenece al servicio; la infraestructura entrega un motor vacío

**Estado:** Aceptada · **Fecha:** 2026-10-02

## Contexto

El sistema se despliega desde tres repositorios: infraestructura (`simple-stock-flow-infra`),
servicio (`simple-stock-flow-api`) y app (`simple-stock-flow-app`). Alguien tiene que crear las
tablas, los índices y los datos de referencia, y hay dos sitios plausibles donde alojar esa
responsabilidad: el repositorio de infraestructura, junto al contenedor del motor, o el
repositorio del servicio, junto al modelo que las usa.

La decisión no es de comodidad: determina **qué commit explica el estado del esquema**.

## Decisión

Las migraciones son de **Alembic**, se **autogeneran desde el modelo de persistencia** de
SQLAlchemy y viven en el adapter de persistencia del servicio
(`src/stockflow/adapters/outbound/persistence/migrations/`). El servicio las aplica al arrancar.

```bash
alembic revision --autogenerate -m "<nombre>"   # genera la migración desde el modelo
alembic upgrade head                            # la aplica (también al arrancar la API)
```

El repositorio de infraestructura levanta MySQL 8.4 (InnoDB, `utf8mb4`), la base `stockflow`
vacía, el usuario, la red y el volumen. **No contiene ni una línea de DDL.**

## Alternativas consideradas

| Alternativa | Por qué no |
|---|---|
| **SQL versionado con pares `up`/`down`**, ejecutado por un runner independiente en el repo de infraestructura | Es la opción correcta cuando varios servicios comparten la misma base o cuando el rol de administrador de base es distinto del de desarrollo. Aquí hay **un solo consumidor**, y el coste real sería mantener a mano un DDL que el ORM ya deriva del modelo, con la garantía añadida de que ambos diverjan |
| **Esquema creado por el contenedor** con scripts de inicialización (`/docker-entrypoint-initdb.d`) | Solo se ejecutan cuando el volumen está vacío. La segunda versión del esquema no se aplicaría nunca, y el fallo sería silencioso |
| **`Base.metadata.create_all()`** sin migraciones | No deja historia, no es reversible y no se puede revisar en un cambio |

## Consecuencias

**Positivas**

- El commit que cambia una entidad trae la migración que la soporta. No se pueden separar.
- La infraestructura es sustituible: cualquier MySQL 8.4 accesible sirve, incluido uno gestionado.
- Un entorno limpio queda completo ejecutando únicamente el servicio.

**Negativas, declaradas**

- **El DDL de MySQL no es transaccional** (cada sentencia hace commit implícito). Una migración
  que falla a la mitad deja el esquema **parcialmente aplicado**, sin vuelta atrás automática.
  Mitigación: migraciones pequeñas (una idea por revisión) y revisar siempre el SQL autogenerado
  antes de aceptarlo.
- Aplicar migraciones en el arranque **no es seguro con varias réplicas** si no se serializa. El
  arranque toma un `GET_LOCK('stockflow_migrations', timeout)` de MySQL antes de
  `alembic upgrade head` y lo libera al terminar; con réplicas, lo preferible es moverlo a un paso
  previo del despliegue. Declarado como riesgo en `plan.md`.
- El arranque del servicio depende de que el motor esté listo, no solo levantado. Requiere el
  healthcheck (`mysqladmin ping`) con `depends_on: condition: service_healthy` **y** reintento de
  conexión en la API.
- Quien revise el esquema tiene que leer el modelo SQLAlchemy o la migración generada, no un
  `.sql` escrito a mano.
- La autogeneración de Alembic **no genera los `CHECK`** (ni detecta sus cambios): se escriben a
  mano en la revisión `initial_schema` y se comprueba que cada `ck_*` de
  [`data-model.md`](../data-model.md) §3.1 aparezca. Los datos de referencia van en una revisión
  aparte, `seed_categories` ([`data-model.md`](../data-model.md) §3.2).
