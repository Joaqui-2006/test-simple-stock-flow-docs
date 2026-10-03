# Guía de Arquitectura Onion — Simple Stock Flow

> **Aclaración sobre la especificación original:**  
> Las carpetas de referencia `spec-python/` y `spec-.net/` describen originalmente una arquitectura **hexagonal** (puertos y adaptadores con inbound/outbound).  
> Para la entrega de la prueba técnica en Laravel y React, **esta edición traduce formalmente esas decisiones a una Arquitectura Onion** de 4 anillos concéntricos más 1 punto de ensamblaje (`Bootstrap`), cumpliendo con rigor los principios de inversión de dependencias y aislamiento del dominio.

---

## 1. Los 4 Anillos + Bootstrap

1. **Anillo 1 · Domain (Dominio Puro):**
   * Modelos de agregado (`Product`, `Sale`, `User`), entidades (`SaleItem`, `Category`), Value Objects (`Money`, `Quantity`, etc.) y excepciones de negocio en español (`BusinessRuleViolation`).
   * **Cero dependencias:** No conoce Laravel, ni Eloquent, ni bases de datos, ni librerías de terceros (salvo librerías matemáticas puras para precisión decimal como `brick/math`).

2. **Anillo 2 · Application (Casos de Uso y Orquestación):**
   * Contiene los casos de uso (`PlaceSaleService`, `ProductCatalogService`, etc.).
   * Define los **Puertos Inbound** (interfaces hacia la presentación) y los **Puertos Outbound** (interfaces hacia la infraestructura, como `UnitOfWork` y repositorios).
   * No contiene lógica de negocio intrínseca ni llamadas directas al framework.

3. **Anillo 3 · Infrastructure (Persistencia y Adaptadores Externos):**
   * Implementaciones concretas de los puertos Outbound: Eloquent ORM, mappers que aíslan las entidades del dominio de las tablas, almacenamiento de archivos, generador de JWT y hasher de contraseñas.
   * Manejo de la concurrencia optimista (`version` en base de datos traducido a `ConcurrencyConflict`).

4. **Anillo 4 · Presentation (Presentación y Entrega HTTP):**
   * Controladores HTTP, FormRequests exclusivamente para validación de forma, Resources para serialización camelCase, y Renderers de Problem Details según contrato.
   * Prohibido importar o conocer la capa de Infraestructura directamente.

5. **Punto de Ensamblaje · Bootstrap:**
   * Ubicado en `app/Bootstrap/PortBindingsServiceProvider.php`.
   * Es el único lugar donde se asocian las interfaces de los puertos con sus implementaciones de infraestructura en el contenedor de servicios de Laravel.

---

## 2. Reglas de Validación de Dependencias

Se hace cumplir de forma automatizada mediante **Deptrac**:
* `Domain` no importa nada del resto de capas ni de `Illuminate\*`.
* `Application` solo importa `Domain`.
* `Presentation` solo importa `Application` (específicamente puertos Inbound). Nunca `Infrastructure`.
* `Infrastructure` solo implementa puertos de `Application` y traduce hacia `Domain`.
