# ADR-006 — Ubicación de los Puertos en la Capa Application

**Estado:** Aceptada · **Fecha:** 2026-10-03

## Contexto
En algunas interpretaciones clásicas de la arquitectura Onion (Jeffrey Palermo), las interfaces de repositorio residen dentro del Dominio. Sin embargo, los Artículos II y IV de la constitución del proyecto establecen el Principio de Inversión de Dependencias (DIP) aplicado a los casos de uso: los puertos salientes (`ports/outbound/`) representan las necesidades del caso de uso respecto al exterior.

## Decisión
Todos los puertos entrantes (`Inbound`) y salientes (`Outbound`), incluidos los repositorios y `UnitOfWork`, residirán en `app/Application/Ports/`.

## Consecuencias
* La capa `Domain` se mantiene completamente pura y agnóstica de mecanismos de persistencia o puertos.
* Un test de arquitectura verifica que `app/Domain/` no contenga interfaces de repositorio ni referencias a operaciones de E/S.
