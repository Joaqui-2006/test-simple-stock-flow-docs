# ADR-009 — Mitigación de Riesgos Propios del Ecosistema Laravel

**Estado:** Aceptada · **Fecha:** 2026-10-03

## Contexto
Laravel fomenta prácticas de desarrollo rápido (como Facades globales `DB::`, Active Record donde los modelos manejan persistencia y negocio, y auto-wiring indiscriminado) que entran en conflicto directo con los principios de Domain-Driven Design y la Arquitectura Onion.

## Decisión
Se establecen barreras explícitas de diseño:
1. **Modelos Eloquent confinados:** Los modelos de Eloquent residen exclusivamente en `app/Infrastructure/Persistence/Model/` y sus propiedades nunca se exponen al dominio. Se utilizan Mappers bidireccionales (`ProductMapper`, `SaleMapper`).
2. **Sin Facades en Dominio ni Aplicación:** Ni `DB::`, ni `Auth::`, ni `Log::` fuera de la capa de Infraestructura o Presentation.
3. **Manejo de Transacciones:** `DB::transaction()` solo existe dentro de `LaravelUnitOfWork` que implementa la interfaz `UnitOfWork` con `callable`.
4. **Reemplazo del 422 por defecto:** El gestor de excepciones de Laravel se reconfigura para que las violaciones de invariantes de negocio devuelvan `application/problem+json` con el campo `detail` en español y sin filtraciones de traza.

## Consecuencias
* Se conserva la potencia de Laravel para enrutamiento, migraciones y dependencias sin comprometer la pureza del modelo de dominio.
