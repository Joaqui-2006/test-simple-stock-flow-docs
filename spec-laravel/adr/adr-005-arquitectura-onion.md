# ADR-005 — Adopción de Arquitectura Onion de 4 Anillos + Bootstrap

**Estado:** Aceptada · **Fecha:** 2026-10-03

## Contexto
La especificación base del proyecto está documentada con arquitectura hexagonal (puertos y adaptadores). Para la implementación en Laravel (PHP) y React (TypeScript), se requiere una estructura concéntrica clara que impida que el framework o la base de datos contaminen la lógica del negocio.

## Decisión
Se adopta formalmente una **Arquitectura Onion** compuesta por:
1. **Domain (Anillo 1):** Entidades, Value Objects, Excepciones puras de negocio.
2. **Application (Anillo 2):** Casos de uso y definición de puertos Inbound/Outbound.
3. **Infrastructure (Anillo 3):** Implementaciones de persistencia Eloquent, mappers y seguridad.
4. **Presentation (Anillo 4):** Controladores, serializadores y entrega HTTP.
5. **Bootstrap (Punto de Ensamblaje):** `PortBindingsServiceProvider` para enlazar puertos e implementaciones en el contenedor.

## Consecuencias
* Se garantiza que el Dominio sea 100% testeable sin bootear Laravel ni requerir base de datos activa.
* Se establece una frontera estricta entre la presentación y la infraestructura.
