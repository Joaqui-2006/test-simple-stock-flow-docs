# Matriz Integral de Trazabilidad, Auditoría Técnica y Cumplimiento
> **Simple Stock Flow · Ficha ADSO 3413974**  
> **Arquitectura Onion (4 Anillos + Bootstrap) en Laravel 11 y React 18**  
> **Metodología:** *Spec-Driven Development* (SDD)

---

## 📑 Índice de Contenidos
1. [Propósito y Alcance](#1-propósito-y-alcance)
2. [Matriz Cruzada de Reglas de Negocio (RN-01 a RN-12)](#2-matriz-cruzada-de-reglas-de-negocio-rn-01-a-rn-12)
3. [Matriz de Decisiones Cerradas de Negocio (DP-01 a DP-04)](#3-matriz-de-decisiones-cerradas-de-negocio-dp-01-a-dp-04)
4. [Catálogo de Endpoints y Puertos Inbound (E-01 a E-15)](#4-catálogo-de-endpoints-y-puertos-inbound-e-01-a-e-15)
5. [Auditoría de Invariantes y Triple Barrera de Blindaje](#5-auditoría-de-invariantes-y-triple-barrera-de-blindaje)
6. [Matriz de Cuerpos de Error (Estándar RFC 7807 Problem Details)](#6-matriz-de-cuerpos-de-error-estándar-rfc-7807-problem-details)
7. [Matriz de Reglas Arquitectónicas de Dependencia (R-01 a R-06)](#7-matriz-de-reglas-arquitectónicas-de-dependencia-r-01-a-r-06)
8. [Trazabilidad de Historias de Usuario (HU-01 a HU-08)](#8-trazabilidad-de-historias-de-usuario-hu-01-a-hu-08)
9. [Catálogo de Decisiones de Arquitectura Registradas (ADR-005 a ADR-010)](#9-catálogo-de-decisiones-de-arquitectura-registradas-adr-005-a-adr-010)
10. [Evidencias de Ejecución y Suite de Pruebas](#10-evidencias-de-ejecución-y-suite-de-pruebas)

---

## 1. Propósito y Alcance
Este documento constituye la fuente de auditoría formal para la evaluación de la prueba técnica. Establece una correspondencia biyectiva entre cada requerimiento de la especificación técnica original (`test-simple-stock-flow-docs/spec-python/`), las decisiones de traducción a Arquitectura Onion en PHP/Laravel y React, y los artefactos de software construidos y verificados.

---

## 2. Matriz Cruzada de Reglas de Negocio (RN-01 a RN-12)

| # | Regla de Negocio | Capa Dominio (Anillo 1) | Capa Aplicación (Anillo 2) | Capa Infraestructura (Anillo 3) / BD | Prueba Automatizada |
|---|---|---|---|---|---|
| **RN-01** | El stock **nunca** es negativo | `Product::deductStock()` lanza `InsufficientStockException` | `PlaceSaleService` orquesta reintentos | `ck_product_stock_non_negative CHECK (stock >= 0)` | `ProductTest`, `PlaceSaleServiceTest` |
| **RN-02** | El precio es **mayor que cero** | `Money::strictlyPositive()` lanza `InvalidPriceException` | `ProductCatalogService` | `ck_product_price_positive CHECK (price > 0)` | `MoneyTest`, `ProductTest` |
| **RN-03** | La cantidad es **mayor que cero** | `Quantity` (`$val > 0`) lanza `InvalidQuantityException` | `PlaceSaleService` | `ck_sale_item_quantity_positive CHECK (quantity > 0)` | `QuantityTest` |
| **RN-04** | Venta con **al menos una línea** | Constructor de `Sale` valida `count($items) > 0` | Validador de comando | `EmptySaleException` al instanciar | `SaleTest::testCannotCreateSaleWithEmptyItems` |
| **RN-05** | Sin productos **repetidos** en la venta | `Sale` constructor valida IDs únicos en líneas | Comprobación de líneas duplicadas | `RepeatedProductException` | `SaleTest::testCannotCreateSaleWithRepeatedProducts` |
| **RN-06** | Precio y nombre se **congelan** al vender | `SaleItem` recibe copias inmutables de precio/nombre | Mapeo al persistir venta | `sale_items.product_name` y `unit_price` | `SaleTest`, `PlaceSaleServiceTest` |
| **RN-07** | Venta registrada **no se altera ni anula** | Ausencia total de mutadores en `Sale` | Sin puertos `UpdateSale` ni `DeleteSale` | Cero verbos `PUT/PATCH/DELETE` en `/api/sales` | Verificación por ausencia de rutas |
| **RN-08** | Producto con ventas **no se borra** | `Product` no conoce ventas | `ManageProducts` coordina baja lógica | `EloquentProductRepository::delete` aplica `deleted_at = now()` | `ProductCatalogService` |
| **RN-09** | Importes en la **misma moneda** | `Money` valida moneda idéntica (`COP`) en `add()` | Uso consistente de `Money` | Columna `currency VARCHAR(3) DEFAULT 'COP'` | `MoneyTest` |
| **RN-10** | `username` **único**, en minúsculas | `Username` normaliza con `strtolower(trim())` | `AuthenticationService` valida duplicidad | Índice `UNIQUE KEY uk_users_username (username)` | `UsernameTest` |
| **RN-11** | Rol en `{admin, seller}` | Enum PHP puro `Role` (`Admin`, `Seller`) | Guardia de roles en middleware | Restricción `ck_user_role CHECK (role IN ('admin', 'seller'))` | `RoleTest` |
| **RN-12** | Total es **siempre** suma de líneas | `Sale::getTotal()` calcula `unitPrice * quantity` dinámicamente | Total nunca proviene del cliente | **Cero columnas `total` o `subtotal`** en BD | `SaleTest::testTotalIsDerivedAtRuntimeFromItems` |

---

## 3. Matriz de Decisiones Cerradas de Negocio (DP-01 a DP-04)

| Decisión | Pregunta Central | Solución Implementada | Justificación Arquitectónica |
|---|---|---|---|
| **DP-01** | Producto renombrado entre ventas: ¿qué nombre muestra el reporte? | Muestra el **nombre congelado más reciente** dentro del rango. | La consulta `EloquentSalesReportQuery` realiza un agrupamiento dinámico en SQL sobre las líneas históricas `sale_items`, garantizando inmutabilidad histórica sin consultar el catálogo vivo. |
| **DP-02** | ¿El reporte desglosa por vendedor? | **No.** Desglose exclusivo por producto. | El DTO `SalesReportRow` es cerrado: contiene exclusivamente `productId`, `productName`, `unitsSold`, `totalAmount`. Ningún endpoint acepta filtros por vendedor. |
| **DP-03** | ¿El catálogo necesita código, SKU o descripción? | **No.** Exactamente 5 atributos de negocio: nombre, precio, stock, categoría, imagen. | La tabla `products` se modela con exactamente esos campos más las columnas técnicas de infraestructura (`id`, `deleted_at`, `version`, `created_at`, `updated_at`). |
| **DP-04** | ¿Un administrador puede crear otros administradores? | **No.** Los administradores solo crean vendedores (`seller`). | El puerto y caso de uso `AuthenticationService::registerSeller` fija forzosamente el rol `Role::Seller`, impidiendo la elevación de privilegios desde la capa de aplicación. |

---

## 4. Catálogo de Endpoints y Puertos Inbound (E-01 a E-15)

| # | Endpoint | Verbo | Rol / Acceso | Puerto Inbound Vinculado | DTOs de Entrada / Salida |
|:---:|---|:---:|:---:|---|---|
| **E-01** | `/api/auth/login` | POST | Anónimo | `Authenticate` | `LoginRequest` → `AuthResult` |
| **E-02** | `/api/auth/register` | POST | **admin** | `Authenticate` | `RegisterSellerRequest` → `UserView` |
| **E-03** | `/api/products` | GET | Token | `ManageProducts` | `search, categoryId, page, pageSize` → `PagedResult<ProductView>` |
| **E-04** | `/api/products/{id}` | GET | Token | `ManageProducts` | `ProductId` → `ProductView` |
| **E-05** | `/api/products` | POST | **admin** | `ManageProducts` | `CreateProductRequest` → `ProductView` |
| **E-06** | `/api/products/{id}` | PUT | **admin** | `ManageProducts` | `UpdateProductRequest` → `ProductView` |
| **E-07** | `/api/products/{id}` | DELETE | **admin** | `ManageProducts` | `ProductId` → 204 No Content |
| **E-08** | `/api/products/{id}/image` | POST | **admin** | `ManageProducts` | `UploadedFile` → `ProductView` |
| **E-09** | `/api/categories` | GET | Token | `ManageProducts` | `void` → `CategoryView[]` |
| **E-10** | `/api/sales` | POST | Token | `PlaceSale` | `PlaceSaleCommand` → `SaleView` |
| **E-11** | `/api/sales` | GET | Token | `GetSales` | `from, to, page, pageSize` → `PagedResult<SaleView>` |
| **E-12** | `/api/sales/{id}` | GET | Token | `GetSales` | `SaleId` → `SaleView` |
| **E-13** | `/api/reports/sales` | GET | Token | `GetSalesReport` | `from, to` (ISO 8601) → `SalesReport` |
| **E-14** | `/health` | GET | Anónimo | — | HealthCheck directo contra conexión PDO |
| **E-15** | `/media/{key}` | GET | Anónimo | `ManageProducts` | Key opaca → Stream binario de imagen |

---

## 5. Auditoría de Invariantes y Triple Barrera de Blindaje

El sistema garantiza que ninguna regla de negocio pueda ser vulnerada, implementando una **triple línea de defensa**:

```
[ Cliente HTTP ]
       │
       ▼
1. Capa Presentación: Validación de Forma (Request DTOs)
       │  (Tipos de datos, presencia de campos obligatorios, formato ISO fechas)
       ▼
2. Capa Dominio: Validación de Invariantes de Negocio (Modelos y Value Objects)
       │  (Stock >= pedido, precio > 0, moneda idéntica, venta con líneas, rol válido)
       ▼
3. Capa Base de Datos: Restricciones de Motor (CHECK Constraints)
          (ck_product_stock_non_negative, ck_product_price_positive, etc.)
```

### Detalle de las 9 Restricciones CHECK en MySQL
1. `ck_product_price_positive`: `price > 0`
2. `ck_product_stock_non_negative`: `stock >= 0`
3. `ck_sale_item_unit_price_positive`: `unit_price > 0`
4. `ck_sale_item_quantity_positive`: `quantity > 0`
5. `ck_user_role`: `role IN ('admin', 'seller')`
6. `ck_category_name_not_empty`: `CHAR_LENGTH(TRIM(name)) > 0`
7. `ck_product_name_not_empty`: `CHAR_LENGTH(TRIM(name)) > 0`
8. `ck_user_username_not_empty`: `CHAR_LENGTH(TRIM(username)) > 0`
9. `ck_product_currency`: `currency = 'COP'`

---

## 6. Matriz de Cuerpos de Error (Estándar RFC 7807 Problem Details)

Por mandato estricto del Artículo VI y el Contrato Congelado, la API emite exactamente tres formatos de error:

| Código HTTP | Escenario de Disparo | Formato de Respuesta | Estructura del Cuerpo |
|:---:|---|:---:|---|
| **400 Bad Request** | Error de formato o parámetro faltante | `application/problem+json` | `{"type":"about:blank","title":"Bad Request","status":400,"detail":"...","errors":{"campo":["mensaje"]}}` |
| **401 Unauthorized** | Token ausente o inválido | Texto plano | **Cuerpo vacío**, `Content-Length: 0`, cabecera `WWW-Authenticate` |
| **403 Forbidden** | Rol insuficiente (vendedor en ruta admin) | Texto plano | **Cuerpo vacío**, `Content-Length: 0` |
| **404 Not Found** | Recurso o ruta inexistente | Texto plano | **Cuerpo vacío**, `Content-Length: 0` |
| **405 Method Not Allowed**| Verbo HTTP incorrecto | Texto plano | **Cuerpo vacío**, `Content-Length: 0`, cabecera `Allow` |
| **409 Conflict** | Conflicto de concurrencia optimista | `application/problem+json` | `{"type":"about:blank","title":"Conflict","status":409,"detail":"Conflicto de concurrencia: el recurso fue modificado..."}` |
| **422 Unprocessable** | Infracción de regla de negocio de dominio | `application/problem+json` | `{"type":"about:blank","title":"Unprocessable Entity","status":422,"detail":"[Mensaje de Dominio en Español]"}` |
| **500 Server Error** | Excepción no controlada | `application/problem+json` | `{"type":"about:blank","title":"Internal Server Error","status":500,"detail":"Ha ocurrido un error interno en el servidor"}` *(Cero fugas de traza SQL)* |

---

## 7. Matriz de Reglas Arquitectónicas de Dependencia (R-01 a R-06)

```
        ┌─────────────────────────────────────────────────────────┐
        │  Bootstrap: PortBindingsServiceProvider (Ensamble)      │
        └────────────────────────────┬────────────────────────────┘
                                     │ enlaza contratos
                  ┌──────────────────┴──────────────────┐
                  ▼                                     ▼
        ┌───────────────────┐                 ┌───────────────────┐
        │   Presentation    │                 │  Infrastructure   │
        │   (Controladores, │                 │  (Eloquent, JWT,  │
        │    Middlewares)   │                 │   Mappers, MySQL) │
        └─────────┬─────────┘                 └─────────┬─────────┘
                  │                                     │
                  │        solo hacia adentro           │
                  └──────────────────┬──────────────────┘
                                     ▼
                        ┌─────────────────────────┐
                        │       Application       │
                        │ (Puertos In/Out, Casos) │
                        └────────────┬────────────┘
                                     ▼
                        ┌─────────────────────────┐
                        │         Domain          │
                        │ (Modelos, Value Objects)│
                        └─────────────────────────┘
```

* **R-01 (`Domain` puro):** Cero importaciones a `Illuminate\`, frameworks o librerías de persistencia.
* **R-02 (`Application` depende solo de `Domain`):** No contiene referencias a infraestructura, controladores ni base de datos.
* **R-03 (`Presentation` no conoce `Infrastructure`):** Desacoplamiento total; la presentación solo invoca puertos de `Application/Ports/Inbound`.
* **R-04 (Inyección en Casos de Uso):** Los constructores de `Application` solo reciben puertos `Outbound` o entidades de `Domain`.
* **R-05 (Inyección en Controladores):** Los controladores únicamente reciben puertos `Inbound`.
* **R-06 (Composition Root único):** Solo `app/Bootstrap/PortBindingsServiceProvider.php` resuelve las interfaces e instancia servicios de infraestructura.

---

## 8. Trazabilidad de Historias de Usuario (HU-01 a HU-08)

| Historia | Descripción | Componente Frontend | Controlador Backend | Caso de Uso |
|---|---|---|---|---|
| **HU-01** | Consultar catálogo de productos | `CatalogView.tsx` | `ProductController::index` | `ProductCatalogService::listProducts` |
| **HU-02** | Mantener catálogo (crear/editar/borrar) | `ProductManagement.tsx` | `ProductController::{store,update,destroy}` | `ProductCatalogService::{create,update,delete}` |
| **HU-03** | Asociar imagen a producto | `ProductImageUpload.tsx` | `ProductController::uploadImage` | `ProductCatalogService::uploadImage` |
| **HU-04** | Registrar una venta | `CartView.tsx` | `SaleController::store` | `PlaceSaleService::execute` |
| **HU-05** | Consultar ventas realizadas | `SalesHistoryView.tsx` | `SaleController::{index,show}` | `GetSalesService::{listSales,getSale}` |
| **HU-06** | Generar reporte de ventas | `SalesReportView.tsx` | `ReportController::sales` | `SalesReportService::getReport` |
| **HU-07** | Iniciar sesión y autenticarse | `LoginForm.tsx` | `AuthController::login` | `AuthenticationService::login` |
| **HU-08** | Operar el sistema por roles | `App.tsx` (Guarda de vistas) | `RequireRole` Middleware | `AuthenticateToken` Middleware |

---

## 9. Catálogo de Decisiones de Arquitectura Registradas (ADR-005 a ADR-010)

* **ADR-005: Adopción de Arquitectura Onion en 4 Anillos concéntricos + Bootstrap.**  
  Justifica la traducción formal del spec hexagonal original a anillos concéntricos con dirección estricta de dependencias hacia el centro.
* **ADR-006: Ubicación de los Puertos en la Capa de Aplicación.**  
  Resuelve la disputa teórica entre Palermo y el spec SENA: los puertos Outbound e Inbound son definidos y gobernados por `Application`.
* **ADR-007: Uso de `Brick\Math\BigDecimal` para precisión monetaria.**  
  Mitiga la ausencia de tipo decimal primitivo en PHP, impidiendo errores de redondeo de punto flotante en cálculos de ventas.
* **ADR-008: Verificación estática con Deptrac.**  
  Sustituye la herramienta `import-linter` de Python para garantizar la frontera arquitectónica en tiempo de compilación.
* **ADR-009: Desacoplamiento de Laravel (Active Record, Facades y Global Helpers).**  
  Prohíbe el uso de Eloquent en el Dominio y abstrae transacciones en `LaravelUnitOfWork`.
* **ADR-010: Orquestación reproducible con Docker Compose.**  
  Garantiza el aprovisionamiento de MySQL 8.4, Laravel y React mediante volúmenes con nombre aislados del sistema operativo anfitrión.

---

## 10. Evidencias de Ejecución y Suite de Pruebas

### 10.1. Verificación Estática con Deptrac
```text
  Report:
  Violations:           0
  Skipped violations:   0
  Uncovered:            108
  Allowed:              322
  Warnings:             0
  Errors:               0
```

### 10.2. Suite de Pruebas Unitarias y Casos de Uso (PHPUnit 11)
```text
PHPUnit 11.5.57 by Sebastian Bergmann and contributors.
Runtime:       PHP 8.2.34
Configuration: /var/www/html/phpunit.xml

.........................                                         25 / 25 (100%)

Time: 00:00.078, Memory: 10.00 MB
OK (25 tests, 173 assertions)
```

### 10.3. Pruebas Unitarias de Dominio Frontend (Node Test Runner)
```text
TAP version 13
# Subtest: Cart accumulates items and calculates total dynamically
ok 1 - Cart accumulates items and calculates total dynamically
# Subtest: Cart throws Error when adding more than available stock
ok 2 - Cart throws Error when adding more than available stock
# Subtest: Cart allows removing items
ok 3 - Cart allows removing items
1..3
# tests 3 | pass 3 | fail 0
```
