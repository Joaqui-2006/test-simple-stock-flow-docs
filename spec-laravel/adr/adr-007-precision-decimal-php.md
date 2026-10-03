# ADR-007 — Precisión Decimal en PHP con Brick\Math\BigDecimal

**Estado:** Aceptada · **Fecha:** 2026-10-03

## Contexto
A diferencia de Python (`decimal.Decimal`) o .NET (`System.Decimal`), el lenguaje PHP nativo no dispone de un tipo primitivo para decimales de precisión fija; el tipo `float` de PHP introduce errores de redondeo de punto flotante IEEE 754 inaceptables para cálculos monetarios y auditoría contable (RN-02, RN-09, RN-12).

## Decisión
Se utiliza la librería `brick/math` y su clase inmutable `Brick\Math\BigDecimal` encapsulada en el Value Object `Money`.
* Todas las operaciones monetarias se realizan con escala fija de 2 decimales y modo de redondeo `RoundingMode::HALF_UP`.
* El mapper de infraestructura rechaza entradas con más de dos cifras decimales en lugar de redondear silenciosamente.

## Consecuencias
* Se evita cualquier pérdida de centavos en la suma de líneas de venta y reportes.
* En base de datos se almacena en columnas `DECIMAL(12, 2)`.
