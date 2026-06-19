# 2026-06-17 · Deploy del ambiente sandbox

**Severidad:** resolved
**Comienzo:** 2026-06-17 17:00 UTC-3
**Cierre:** 2026-06-17 17:30 UTC-3

## Resumen

Se desplegó la versión 1.1 del BaaS que introduce el ambiente sandbox
(`sk_test_`) sin afectar producción. Hubo una ventana de ~30 segundos
durante la cual los containers `api` y `worker` se reiniciaron para tomar
la migración alembic 0002 que agrega la columna `environment` a
`emission_jobs`.

## Impacto

- Jobs en vuelo: ninguno perdido (la migración es idempotente con
  `IF NOT EXISTS`).
- Latencia: el primer request post-restart tuvo cold start de ~3 s.
- Folios reales del SII: sin impacto.

## Resolución

Deploy completado vía rsync al droplet + `docker compose up -d --build
api worker migrate` + `alembic upgrade head`. Verificación con curl al
`/health` y emisión de prueba en producción (folio real OK).

## Lecciones aprendidas

- La migración alembic con `ADD COLUMN IF NOT EXISTS` permitió hacer el
  deploy sin downtime real (los jobs existentes siguen funcionando como
  `environment='live'` por default).
- El smoke test post-deploy debería incluir ambos modos (sandbox + live)
  para detectar regresión rápido.
