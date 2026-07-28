# Operación de QA y estacionamiento de SAP-dev

## Contrato del ambiente QA

- Código: ramas largas `qa` de Backend-R7 y Frontend-R7.
- Imágenes: jobs `Backend-R7-CI-QA` y `Frontend-R7-CI-QA`.
- GitOps: rama `deploy/qa`.
- Argo: `backend-qa`, `frontend-qa` y `routing-qa`.
- URL: `http://qa.98.85.131.168.nip.io:32047`.
- PostgreSQL y credenciales externas: compartidos con DEV.
- Redis: mismo servidor y credencial que DEV, pero QA usa el DB lógico `1`.
- Worker CSV: una réplica permanente en QA.

Los jobs QA solo publican imágenes. El despliegue se realiza con un commit
GitOps que actualiza los tags correspondientes en `deploy/qa`; nunca deben
modificar el overlay de DEV.

QA no ofrece aislamiento de datos ni de migraciones. Toda migración futura debe
ser retrocompatible y coordinarse con DEV. Una operación QA también puede
certificar en FEL o producir eventos SAP sobre los datos compartidos.

## Propiedad del worker SAP

El estado normal obligatorio es:

| Ambiente | `SAP_WORKER_ENABLED` |
| --- | --- |
| DEV | `true` |
| QA | `false` |
| SAP-dev | `false` |

Para probar el worker de QA:

1. Verificar que no existan eventos `PROCESSING` y registrar el conteo de
   `PENDING`/`RETRY`.
2. Cambiar DEV a `false` por GitOps y esperar el log
   `integration.sap_sync.worker.disabled`.
3. Cambiar QA a `true` por GitOps y esperar
   `integration.sap_sync.worker.started` y la adquisición del lease.
4. Ejecutar únicamente el evento controlado autorizado y reconciliar estado,
   enlace SAP y auditoría.
5. Volver QA a `false`, confirmar `worker.disabled`, activar DEV y comprobar
   `worker.started`.
6. Verificar que no quede un backlog nuevo.

Nunca se deben activar DEV y QA simultáneamente.

## SAP-dev estacionado

Snapshot previo al estacionamiento del 2026-07-28:

- GitOps: `deploy/sap-integration-dev-20260622` en `e139083a`.
- Backend: `docker.io/retailfactory/backend-r7:2fa09b46`.
- Frontend: `docker.io/retailfactory/frontend-r7:f7f68f44`.
- URL previa: `http://sap-dev.98.85.131.168.nip.io:32047`.
- Jobs conservados: `Backend-R7-CI-Integracion-Sap-20260622` y
  `Frontend-R7-CI-Integracion-Sap-20260622`.

Estacionado significa réplicas de backend/frontend en cero, servicios
ClusterIP, Ingress retirado y jobs Jenkins deshabilitados. Se conservan las
aplicaciones Argo, namespace, secretos, certificado, historial y rama GitOps.

Para revivirlo, habilitar primero los jobs, revertir el commit de
estacionamiento y esperar que Argo restaure pods e Ingress. El worker SAP debe
permanecer en `false` salvo un relevo exclusivo aprobado.

## Verificación y rollback de QA

La prueba mínima de un despliegue QA es:

1. SHA de `qa` igual al checkout registrado por Jenkins.
2. Imagen existente en DockerHub con el tag de ocho caracteres.
3. Tag idéntico en `deploy/qa`.
4. Aplicación Argo `Synced` y `Healthy`.
5. Deployment y pod Ready con la imagen esperada.
6. HTTP 200 en `/` y `/api/v1/health`.
7. DEV mantiene su worker SAP activo y SAP-dev continúa sin pods.

Si QA causa presión de recursos, fallas de cola o regresiones, escalar los tres
deployments QA a cero o revertir el último commit de `deploy/qa`. No reducir
réplicas ni recursos de DEV para sostener QA.
