# ADR-010 — Entorno de Ejecución Docker y Gestión de Volúmenes

**Estado:** Aceptada · **Fecha:** 2026-10-03

## Contexto
El desarrollo multiplataforma (especialmente en entornos Windows como el de los aprendices y evaluadores) suele presentar problemas graves de rendimiento y permisos de lectura/escritura si los directorios `vendor/` de PHP o `node_modules/` de Node se montan directamente como bind-mounts hacia el sistema de archivos del host. Asimismo, el spec requiere levantar el ecosistema completo sin necesidad de instalar PHP, Composer ni MySQL en la máquina anfitriona.

## Decisión
1. **Volúmenes con nombre en Docker:** Los directorios `vendor/` y `node_modules/` se configuran como volúmenes persistentes con nombre administrados por el daemon de Docker (`api_vendor`, `app_node_modules`), garantizando alta velocidad de E/S.
2. **Docker Compose Unificado:** En `test-simple-stock-flow-infra` se orquesta la totalidad de los servicios (`db` MySQL 8.4, `api` Laravel, `app` React/Nginx) en una red interna `stockflow-network`.
3. **Migraciones al arranque:** El contenedor de la API aplica las migraciones y seeders de categorías al arrancar antes de exponer el puerto `8000`.

## Consecuencias
* Cualquier desarrollador o instructor puede clonar los repositorios e iniciar el entorno con un único comando `docker compose up --build`.
