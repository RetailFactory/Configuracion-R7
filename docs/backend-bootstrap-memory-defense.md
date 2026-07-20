# Defensa operacional para bootstrap offline v2

El deployment GitOps `apps/backend/base/deployment.yaml` controla el contenedor
`backend` en `r7-dev` (y overlays QA/producción). El diseño v2 pagina y materializa
en PostgreSQL; estos valores son una defensa adicional, no el fix primario.

- `NODE_OPTIONS=--max-old-space-size=768`
- request `384Mi`, limit `1Gi` (margen para heap, RSS, buffers y runtime)
- páginas: 250 filas / objetivo normal de 65,536 bytes
- fila individual: máximo duro de 1,048,576 bytes; una fila que supera el
  objetivo de página se entrega sola sin elevar el tamaño de todas las páginas
- 12 sesiones globales, 1 por empresa/dispositivo
- TTL 7,200 segundos, máximo temporal 1 GiB por snapshot
- timeout de página 30 segundos
- startup HTTP hasta 120 segundos
- readiness/liveness HTTP sobre `/api/v1/sync/connectivity`
- `terminationGracePeriodSeconds: 60`

Despliegue: aplicar primero la migración Backend 138, publicar la imagen Backend,
publicar Desktop/Frontend v2 y sincronizar las aplicaciones ArgoCD. Verificar
`sync.bootstrap.*`, reinicios del pod, heap/RSS y filas expiradas de staging.

Rollback: volver los tres artefactos a versiones compatibles y sólo entonces
ejecutar el rollback 138. No reactivar el endpoint monolítico en producción.
