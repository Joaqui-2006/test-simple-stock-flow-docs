# Simple Stock Flow — Arquitectura Onion · VERSIÓN FINAL

> **Documento de arranque del equipo.** Define el qué, el cómo y el orden de trabajo.
> Quien lea esto completo no necesita volver a preguntar qué construir.
>
> **Antes de empezar:** hay que tener abierta la especificación en
> `test-simple-stock-flow-docs/spec-python/`. Este documento **traduce** esa
> especificación a Laravel + React en Onion. No la reemplaza: es la segunda
> fuente de verdad si se contradicen, y por eso toda decisión de traducción
> está registrada como ADR.

---

## 📌 1 · Qué construimos

**Simple Stock Flow** — control de stock y ventas para un almacén pequeño.

El sistema tiene **3 afirmaciones** y todo se apoya en ellas:

1. El stock que muestra el catálogo **es** el stock que hay.
2. Una venta registrada **no se puede alterar** después.
3. El reporte de un período cerrado dice hoy lo mismo que dirá en un año.

**Actores:** `seller` (vende y consulta) · `admin` (todo lo anterior + catálogo, imágenes y alta de usuarios) · anónimo (solo login y health).

**Historias:** HU-01 catálogo · HU-02 mantener catálogo · HU-03 imagen · HU-04 registrar venta · HU-05 consultar ventas · HU-06 reporte · HU-07 autenticación · HU-08 operar el sistema.

**No existe el comprador como actor.** Está fuera de alcance: devoluciones, pagos, envíos, descuentos, impuestos, notificaciones y CRUD de categorías.

---

## 📌 2 · Las 12 reglas de negocio

Cada una tiene dueño, capa que la prueba y tarea asignada.

| # | Regla | Tarea |
|---|---|---|
| RN-01 | El stock **nunca** es negativo | T-10 · T-20 |
| RN-02 | El precio es **mayor que cero** | T-05 · T-21 |
| RN-03 | La cantidad es **mayor que cero** | T-21 |
| RN-04 | Una venta tiene **al menos una línea** | T-10 |
| RN-05 | Un producto **no se repite** en la misma venta | T-10 · T-20 |
| RN-06 | El precio y el nombre se **congelan** al vender | T-11 |
| RN-07 | Una venta registrada **no se modifica ni se anula** | T-22 |
| RN-08 | Un producto vendido **no se borra**: se da de baja | T-09 |
| RN-09 | Todos los importes en la **misma moneda** | T-05 |
| RN-10 | `username` **único**, normalizado a minúsculas | T-06 · T-21 |
| RN-11 | El rol está en el conjunto cerrado `{admin, seller}` | T-21 |
| RN-12 | El total **siempre** es la suma de sus líneas | T-07 |

**RN-07 se cumple por ausencia:** no hay operación de edición en el agregado, ni método en el puerto, ni verbo HTTP. T-22 lo demuestra revisando las tres superficies.

---

## 📌 3 · Las 4 decisiones cerradas de negocio

| # | Pregunta | Decisión | Consecuencia técnica |
|---|---|---|---|
| **DP-01** | Producto renombrado entre dos ventas del mismo rango: ¿qué nombre muestra el reporte? | El **congelado más reciente dentro del rango** | Ventana en la consulta agregada del motor. El catálogo vivo no se consulta nunca |
| **DP-02** | ¿El reporte desglosa por vendedor? | **No.** Solo por producto | `SalesReportRow` es cerrado. Ningún endpoint acepta filtro por vendedor |
| **DP-03** | ¿El catálogo necesita descripción, código o SKU? | **No.** Exactamente 5 atributos: nombre, precio, stock, categoría, imagen | `product` no tiene columnas más allá de esos 5 + `deleted_at` + `version` |
| **DP-04** | ¿Un admin puede crear otros admins? | **No.** Los admins crean sellers | **Estructural:** el puerto de registro no sabe crear admins |

---

## 📌 4 · Contrato de API — los 15 endpoints

`api-contract.md` **manda**. Si el OpenAPI autogenerado discrepa, el código está mal.

| # | Método | Ruta | Auth | Caso de uso |
|---|---|---|---|---|
| E-01 | POST | `/api/auth/login` | anónimo | `Authenticate` |
| E-02 | POST | `/api/auth/register` | **admin** | `Authenticate` |
| E-03 | GET | `/api/products` | autenticado | `ManageProducts` |
| E-04 | GET | `/api/products/{id}` | autenticado | `ManageProducts` |
| E-05 | POST | `/api/products` | **admin** | `ManageProducts` |
| E-06 | PUT | `/api/products/{id}` | **admin** | `ManageProducts` |
| E-07 | DELETE | `/api/products/{id}` | **admin** | `ManageProducts` |
| E-08 | POST | `/api/products/{id}/image` | **admin** | `ManageProducts` |
| E-09 | GET | `/api/categories` | autenticado | `ManageProducts` |
| E-10 | POST | `/api/sales` | autenticado | `PlaceSale` |
| E-11 | GET | `/api/sales` | autenticado | `GetSales` |
| E-12 | GET | `/api/sales/{id}` | autenticado | `GetSales` |
| E-13 | GET | `/api/reports/sales?from=&to=` | autenticado | `GetSalesReport` |
| E-14 | GET | `/health` | anónimo | — |
| E-15 | GET | `/media/{key}` | anónimo | `ManageProducts` |

**5 puertos entrantes, no 6:** el alta de usuario (E-02) cae dentro de `Authenticate`. El recuento lo fija el spec y no cambia.

---

## 📌 5 · Los 3 cuerpos de error (no negociable)

Solo pueden salir estas tres formas:

| Forma | Cuándo | Cuerpo |
|---|---|---|
| **422 / 409 / 500** | regla de negocio, conflicto, excepción | `application/problem+json` con `detail` en **español** |
| **400** | forma mal formada | `{title, status, detail, errors:{campo:[msg]}}`. El 500 filtra nada: ni tipo, ni traza, ni SQL |
| **401 / 403 / 404 / 405** | auth, permiso, inexistente, método | **Cuerpo vacío.** `Content-Length: 0`. Se conservan `WWW-Authenticate` y `Allow` |

**Regla que lo gobierna:** el esquema valida **solo la forma** (presencia, tipo, formato). **Toda invariante de negocio la valida el dominio** y sale como **422**. En los DTO está **prohibido** `gt`, `ge`, `min_length` sobre campos de negocio: moverían un 422 a un 400.

---

## 📌 6 · ARQUITECTURA ONION — las 5 capas

**4 anillos + 1 punto de ensamblaje.**

```
        ┌──────────────────────────────────────┐
        │  Bootstrap                           │  ← imports todo.
        │  app/Bootstrap/                      │    NADIE lo importa.
        │  PortBindingsServiceProvider.php     │    NO ES UN ANILLO.
        └───────────────┬──────────────────────┘
                        │ conecta implementaciones con sus interfaces
     ┌──────────────────┴──────────────────┐
     ▼                                     ▼
┌─────────────────┐            ┌──────────────────────┐
│ Presentation    │            │ Infrastructure       │
│ Anillo 4        │            │ Anillo 3             │
│ controllers     │            │ Eloquent · mappers   │
│ requests        │            │ repos · JWT · binarios│
│ resources       │            │ migraciones · config │
│ errores         │            └──────────┬───────────┘
│ ═══ PROHIBIDO ══│                       │
│ importar Infra  │                       │
└────────┬────────┘                       │
         │        imports hacia adentro   │
         ▼                                ▼
       ┌───────────────────────────────────────┐
       │ Application   Anillo 2                │
       │ casos de uso · 5 puertos in           │
       │ 10 puertos out · SIN lógica negocio   │
       └──────────────────┬────────────────────┘
                          │ solo importa Domain
                          ▼
       ┌───────────────────────────────────────┐
       │ Domain   Anillo 1                    │
       │ entidades · value objects · invariantes│
       │ IMPORTA NADA. Ni Laravel. Ni la BD.   │
       └───────────────────────────────────────┘
```

---

## 📌 7 · Las 6 reglas de dependencia y cómo se verifican

| # | Regla | Verificación |
|---|---|---|
| **R-01** | `Domain` no importa nada del proyecto ni de terceros | `grep -R "Illuminate\\\\" app/Domain` → **0**. Un test carga las clases **sin bootear Laravel** |
| **R-02** | `Application` solo importa `Domain` | Deptrac |
| **R-03** | `Presentation` **no** importa `Infrastructure` | Deptrac |
| **R-04** | Todo constructor de `Application\` recibe solo clases de `Domain` o puertos de `Outbound` | test de reflexión |
| **R-05** | Todo constructor de `Presentation\` recibe solo puertos de `Inbound` — **nunca** de `Outbound` | test de reflexión |
| **R-06** | Solo `app/Bootstrap/` instancia `Infrastructure\*` | test de reflexión |

---

## 📌 8 · Por qué los puertos viven en Application y no en Domain

La Onion clásica de Palermo pone las interfaces de repositorio dentro de `Domain`. Nuestra constitución no:
- **Artículo II** — «Todo lo que necesita del exterior lo expresa como un puerto que **él mismo define** en `ports/outbound/`.»
- **Artículo IV (DIP)** — «Los módulos de `application/ports/outbound/` viven en el paquete de aplicación.»

Se quedan en `Application/Ports/`. Registrado como **ADR-006**.

---

## 📌 9 · Backend — Domain (Anillo 1)

```
app/Domain/
├── Model/
│   ├── Product.php        raíz de agregado · SIN `version`
│   ├── Sale.php           raíz de agregado · SIN `total`
│   ├── SaleItem.php       entidad interna · SIN `subtotal`
│   ├── Category.php       entidad de referencia, solo lectura
│   └── User.php           raíz de agregado
├── ValueObject/
│   ├── Money.php          Brick\Math\BigDecimal · escala 2 · HALF_UP
│   ├── Quantity.php       entero > 0
│   ├── ProductId.php · SaleId.php · CategoryId.php · UserId.php
│   ├── Username.php       normaliza a minúsculas (RN-10)
│   └── Role.php           admin | seller (RN-11)
├── Exception/             mensajes en ESPAÑOL · viajan tal cual al 422
│   ├── BusinessRuleViolation.php        ← base abstracta
│   ├── InsufficientStockException.php · InvalidPriceException.php
│   ├── InvalidQuantityException.php · EmptySaleException.php
│   ├── RepeatedProductException.php · ProductNotFoundException.php
│   ├── UnknownCategoryException.php · DuplicateUsernameException.php
│   └── InvalidRoleException.php · InvalidCredentialsException.php
└── Service/               VACÍA hasta que una regla se gane el lugar
    └── .gitkeep
```

---

## 📌 10 · Backend — Application (Anillo 2)

```
app/Application/
├── Ports/
│   ├── Inbound/           ← 5 puertos + DTOs (fuente del contrato)
│   │   ├── PlaceSale.php · ManageProducts.php · GetSales.php
│   │   ├── GetSalesReport.php · Authenticate.php
│   │   ├── PlaceSaleCommand.php
│   │   ├── ProductView.php · SaleView.php · SaleItemView.php
│   │   └── SalesReport.php · SalesReportRow.php
│   │   └── AuthResult.php · PagedResult.php
│   └── Outbound/          ← 10 puertos
│       ├── ProductRepository.php · SaleRepository.php
│       ├── CategoryRepository.php · UserRepository.php
│       ├── FileStorage.php · PasswordHasher.php · TokenGenerator.php
│       └── Clock.php · UnitOfWork.php · SalesReportQuery.php
├── UseCase/               un caso de uso = una clase
│   ├── PlaceSaleService.php       (T-10)
│   ├── ProductCatalogService.php  (T-04)
│   ├── GetSalesService.php        (T-07)
│   ├── SalesReportService.php     (T-08)
│   └── AuthenticationService.php  (T-06)
├── Model/
│   ├── PageRequest.php    el recorte a 100 es regla de aplicación
│   └── DateRange.php       `from` inclusivo · `to` exclusivo
└── Exception/
    └── ConcurrencyConflict.php   ← la define Application (ADR-002)
```

---

## 📌 11 · El puerto `UnitOfWork`

```php
interface UnitOfWork {
    public function run(callable $operation): mixed;
}
```

La implementación `LaravelUnitOfWork` es la única que contiene `DB::transaction()`.

---

## 📌 12 · Backend — Infrastructure (Anillo 3)

```
app/Infrastructure/
├── Persistence/
│   ├── Model/             Eloquent · NUNCA en Domain
│   │   └── ProductModel.php · SaleModel.php · SaleItemModel.php
│   │       CategoryModel.php · UserModel.php
│   ├── Mapper/            ProductMapper.php · SaleMapper.php · …
│   │                      ← aquí vive `version`, que el dominio NO ve
│   ├── Repository/
│   │   ├── EloquentProductRepository.php · EloquentSaleRepository.php
│   │   ├── EloquentCategoryRepository.php · EloquentUserRepository.php
│   │   └── EloquentSalesReportQuery.php   ← agrupa EN EL MOTOR (ADR-004)
│   ├── LaravelUnitOfWork.php           ← DB::transaction() vive AQUÍ
│   └── Concurrency/       0 filas afectadas → ConcurrencyConflict
├── Security/
│   └── JwtTokenGenerator.php · Argon2PasswordHasher.php
├── Storage/
│   └── LocalFileStorage.php   clave opaca · el dominio nunca ve rutas
├── Configuration/
│   └── Settings.php     readonly · SIN default para secretos (Art. IX)
└── Logging/
    └── CorrelationId.php
```

---

## 📌 13 · Backend — Presentation (Anillo 4)

```
app/Presentation/
├── Http/
│   ├── Controller/        8 controladores
│   ├── Request/           SOLO forma. Prohibido gt/min en campos de negocio
│   ├── Resource/          ProductResource · SaleResource · …
│   ├── Serialization/     CamelCase · Money(→number) · Utc(→+00:00)
│   └── ProblemDetails/    los 3 cuerpos de error del contrato
│       ├── ProblemDetailsRenderer.php    422 / 409 / 500
│       ├── ValidationErrorRenderer.php   400 con `errors`
│       └── EmptyErrorRenderer.php        401/403/404/405 · Content-Length: 0
└── Middleware/
    └── AuthenticateToken.php · RequireRole.php · CorrelationIdMiddleware.php

app/Bootstrap/                             ← NO ES UN ANILLO
└── PortBindingsServiceProvider.php         ← único archivo que amarra un puerto
```

---

## 📌 14 · Frontend — los mismos 4 anillos

```
app/src/
├── domain/              Anillo 1 · sin React
│   ├── model/           Product · Cart · SaleItem · Money · Quantity
│   └── error/
├── application/         Anillo 2
│   ├── ports/           ProductRepository · CartRepository · SessionRepository
│   ├── use-cases/       BrowseCatalog · AddToCart · Checkout · Login · ViewSalesReport
│   └── state/
├── infrastructure/      Anillo 3
│   ├── http/
│   │   ├── client.ts · interceptors/
│   │   └── dto/api.dto.ts
│   ├── mappers/
│   └── providers.ts
└── features/            Anillo 4 = PRESENTACIÓN
    └── auth/ catalog/ cart/ sales/ reports/
```

---

## 📌 15 · Repositorios que NO llevan Onion

* `infra`: Orquestación Docker, MySQL 8.4 vacío. Cero lógica.
* `tool`: Cliente HTTP para sembrado de datos demo.
* `page`: HTML/CSS estático público.
* `docs`: Especificaciones, ADRs y contratos.

---

## 📌 16 · Docker y Entorno

* Red unificada `stockflow-network`.
* Carpetas hermanas en el espacio de trabajo.
* Sin binarios globales requeridos en la máquina anfitriona.
