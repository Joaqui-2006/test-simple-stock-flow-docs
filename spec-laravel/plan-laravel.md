# Plan de Implementación — Simple Stock Flow (Laravel + React en Onion)

**Fecha:** 2026-10-03  
**Stack de destino:** PHP 8.2+ (Laravel 11) · React 18+ (TypeScript) · MySQL 8.4 · Docker Compose  
**Metodología:** SDD (*Spec-Driven Development*) + Onion Architecture (4 anillos + Bootstrap)

---

## 1. Mapeo de Fases

| Fase | Alcance | Entregable / Verificación |
|---|---|---|
| **P0 · Docs** | Documentación de arquitectura Onion, ADR-005 a 010, `plan-laravel.md` y contrato DTO congelado. | Commit y merge a `main` en `test-simple-stock-flow-docs`. |
| **P1 · Dominio** | Modelos puros, Value Objects, Excepciones de negocio en español, verificación sin framework. | `app/Domain/`, suite `tests/Domain/` verde sin bootear Laravel. |
| **P2 · Esquema e Infraestructura** | Migraciones de base de datos con los 9 `CHECK`, seed de categorías fijas, mappers y repositorios Eloquent. | `app/Infrastructure/`, `database/migrations/`, Docker Compose en `infra`. |
| **P3 · Casos de Uso (Aplicación)** | 5 casos de uso orquestadores, 5 puertos Inbound, 10 puertos Outbound, UnitOfWork con callable. | `app/Application/`, suite `tests/Application/` con fakes. |
| **P4 · Presentación y Robustez** | Controladores HTTP, FormRequests de forma, renderers de error (422 problem+json, 400 validation, 401/403/404 vacío), concurrencia optimista y JWT. | 15 endpoints del contrato verificados, `tests/Feature/` y `tests/Architecture/` verde. |
| **P5 · Frontend React** | App React en Onion (`domain`, `application`, `infrastructure`, `features`), carrito in-memory, catálogo, ventas y reporte. | Build de producción en Nginx, proxy `/api` y `/media`. |
| **P6 · Satélites** | Landing page estática (`test-simple-stock-flow-page`) y sembrador CLI vía API HTTP (`test-simple-stock-flow-tool`). | Contenedor tool ejecuta seed contra API real; landing carga sin dependencias de API. |
| **P7 · Validación Final y Auditoría** | READMEs en los 6 repositorios (las 5 preguntas obligatorias), barrido de idioma (Art. XI) y docker-compose up end-to-end. | Sistema 100% operativo y verificado. |

---

## 2. Mapa de Tareas (T-01 a T-23)

* **T-01:** Configuración inicial de contenedor Docker y herramientas de calidad (Deptrac, PHPStan, PHPUnit).
* **T-02:** Definición de Puertos Outbound (UnitOfWork, Repositories, Clock, Storage, Hasher, TokenGenerator).
* **T-03:** Dominio Puro — Modelos `Product`, `Category`, `User`, Value Objects `Money`, `Quantity`.
* **T-04:** Caso de uso `ProductCatalogService` y puerto `ManageProducts`.
* **T-05:** Guardas de moneda, regla RN-02 y RN-09.
* **T-06:** Autenticación `AuthenticationService` con JWT y Argon2 (RN-10).
* **T-07:** Consulta de ventas `GetSalesService` y total derivado (RN-12).
* **T-08:** Reporte de ventas `SalesReportService` agrupado en motor (DP-01, DP-02).
* **T-09:** Baja lógica de productos (RN-08, ADR-003).
* **T-10:** Registro de venta atómica `PlaceSaleService` con bloqueo optimista (RN-01, RN-04, RN-05, ADR-002).
* **T-11:** Congelación de precio y nombre al momento de la venta (RN-06).
* **T-12:** Auditoría y autoría en ventas y productos.
* **T-13:** Índices de base de datos para búsqueda sin distinción de tildes ni mayúsculas.
* **T-14:** Almacenamiento binario local para imágenes con clave opaca.
* **T-20:** Restricciones `CHECK` a nivel de base de datos para defensas en profundidad.
* **T-21:** Invariantes de valor en Value Objects.
* **T-22:** Inmutabilidad de ventas registradas (RN-07 verificada por ausencia).
* **T-23:** Deptrac y pruebas de arquitectura automáticas (R-01 a R-06).
