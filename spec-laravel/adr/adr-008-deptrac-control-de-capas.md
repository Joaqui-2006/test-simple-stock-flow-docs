# ADR-008 — Control Estricto de Capas con Deptrac

**Estado:** Aceptada · **Fecha:** 2026-10-03

## Contexto
El spec en Python garantiza la integridad arquitectónica con `import-linter` y tipado estricto `mypy`. En el ecosistema PHP es necesario un mecanismo automatizado equivalente que impida acoplamientos prohibidos (por ejemplo, que un controlador invoque repositorios Eloquent saltándose los casos de uso, o que el dominio importe clases de Laravel).

## Decisión
Se incorpora la herramienta **Deptrac** (`qossmic/deptrac-shim`) configurada en `depfile.yaml` para validar en CI/CD y en `verify.sh` las 6 reglas de dependencia (R-01 a R-06):
1. `Domain` no depende de nadie.
2. `Application` solo depende de `Domain`.
3. `Presentation` no depende de `Infrastructure`.
4. Los constructores de `Application` solo reciben tipos de `Domain` o puertos `Outbound`.
5. Los constructores de `Presentation` solo reciben puertos `Inbound`.
6. Solo `Bootstrap` instancia clases de `Infrastructure`.

## Consecuencias
* Cualquier import o inyección indebida romperá el análisis estático de inmediato antes de que el código llegue a producción.
