# Integracion SAP Business One para SAP01

Estado actualizado: 22 de junio de 2026.

Esta guia aplica al backend R7 desplegado para `cod_emp=SAP01` en el ambiente
aislado `r7-sap-dev`. La URL de Service Layer y `CompanyDB` se almacenan en la
configuracion de base de datos:

- Service Layer: `https://200.12.40.202:50500/b1s/v1/`
- CompanyDB: `TEST_SBOTODOCALZA_`

Las credenciales SAP no deben guardarse en Git, Kustomize, seeds, imagenes,
logs ni manifiestos sin cifrar. El seed solo guarda referencias
`env:SAP01_SAP_USERNAME` y `env:SAP01_SAP_PASSWORD`.

## Variables de entorno

El deployment base lee todas estas variables de `backend-secrets` con
`optional: true`:

| Variable | Uso | Default del backend |
| --- | --- | --- |
| `SAP01_SAP_USERNAME` | Usuario de Service Layer para SAP01 | Sin default; requerida para conexion real |
| `SAP01_SAP_PASSWORD` | Password de Service Layer para SAP01 | Sin default; requerida para conexion real |
| `SAP_HTTP_ALLOW_SELF_SIGNED` | Permite certificado autofirmado en pruebas | `false` |
| `SAP_SESSION_CACHE_TTL_SECONDS` | TTL maximo de la sesion SAP en memoria | `1200` |
| `SAP_WORKER_ENABLED` | Activa el polling asincrono de `sap_evento_sincronizacion` | `false` |
| `SAP_WORKER_BATCH_SIZE` | Maximo de eventos reclamados por ciclo | `10` |
| `SAP_WORKER_POLL_INTERVAL_MS` | Intervalo entre ciclos del worker | `10000` |

El overlay aislado `sap-dev` establece `SAP_WORKER_ENABLED="false"` para evitar
que una build nueva escriba documentos reales en SAP sin una activacion
operativa explicita. El procesamiento solo ocurre si la variable de entorno y
`sap_cfg_empresa.config.worker.enabled` tambien estan activos para la empresa.
En QA y produccion la variable queda controlada por `backend-secrets`; si la
clave no existe, el backend conserva el default `false`.

Use `SAP_HTTP_ALLOW_SELF_SIGNED="true"` unicamente en dev/QA cuando el Service
Layer de pruebas use un certificado autofirmado. No lo habilite en produccion.

## Crear o actualizar backend-secrets

No escriba valores reales en este repositorio. Ejecute los comandos desde una
terminal autorizada y reemplace los placeholders al momento de ejecutarlos.

Ejemplo solicitado para crear/aplicar las credenciales:

```bash
kubectl -n r7-sap-dev create secret generic backend-secrets \
  --from-literal=SAP01_SAP_USERNAME='<usuario>' \
  --from-literal=SAP01_SAP_PASSWORD='<password>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

`backend-secrets` tambien contiene `DATABASE_URL`, `REDIS_URL`,
`RABBITMQ_URL`, claves JWT y otras variables. Si el Secret ya existe y no se
administra con un manifiesto que incluya todas sus claves, actualice solo las
claves SAP para no reconstruir ni perder las demas:

```bash
kubectl -n r7-sap-dev patch secret backend-secrets \
  --type merge \
  --patch '{"stringData":{"SAP01_SAP_USERNAME":"<usuario>","SAP01_SAP_PASSWORD":"<password>"}}'
```

Los controles opcionales tambien pueden administrarse en el Secret:

```bash
kubectl -n r7-sap-dev patch secret backend-secrets \
  --type merge \
  --patch '{"stringData":{"SAP_SESSION_CACHE_TTL_SECONDS":"1200","SAP_WORKER_BATCH_SIZE":"10","SAP_WORKER_POLL_INTERVAL_MS":"10000"}}'
```

Despues de actualizar el Secret, reinicie el deployment para que el proceso
reciba los nuevos valores:

```bash
kubectl -n r7-sap-dev rollout restart deployment/backend
kubectl -n r7-sap-dev rollout status deployment/backend
```

## Ejecutar el seed SAP01

Los exports actualizados el 22 de junio ya contienen el resultado estructural
de `72`, `75`, `77` y `89`. Despues de una restauracion completa se reejecuta
`89_reconcile_sap_spanish_schema_and_complete_operational_sync.sql` como
convergencia idempotente y despues
`90_realign_available_shared_catalog_sequences.sql`. En una base realmente
vacia el orden es `72 SAP -> 75 SAP -> 77 SAP -> 89 -> 90`.

`temp/tablasr7.sql` aun contiene regiones `-- missing source code`; no debe
tratarse como respaldo integral hasta regenerar/completar el schema dump.

Solo en dev/QA:

```bash
cd Backend-R7/backend-api
psql "$DATABASE_URL" \
  -v ON_ERROR_STOP=1 \
  -f db/seeds/erp_company_config_test_seed.sql
```

El seed es idempotente, configura SAP01 con las referencias a variables de
entorno y deja pendientes los valores SAP que deben obtenerse del ambiente
real. No inserta usuario ni password de Service Layer.

## Probar la conexion

El endpoint requiere JWT, permiso `ADMIN_COMPANY_VIEW` y alcance sobre SAP01:

```bash
curl -sS -X POST \
  "${BACKEND_URL}/api/v1/integrations/sap/SAP01/test-connection" \
  -H "Authorization: Bearer <jwt>"
```

Revise `data.connected`. La prueba inicia sesion y consulta una bodega en
Service Layer. Las llamadas se registran de forma sanitizada en `adm_int_log`;
no deben aparecer password, cookies `B1SESSION`/`ROUTEID` ni otros secretos.

## Activar y revisar el worker

`sap-dev` deja el worker apagado por defecto en
`apps/backend/overlays/sap-dev/kustomization.yaml`. Para una prueba comercial
controlada, cambie temporalmente el valor a `true` o agregue
`SAP_WORKER_ENABLED="true"` a `backend-secrets` si el overlay no lo sobreescribe,
active tambien `sap_cfg_empresa.config.worker.enabled` y reinicie el deployment.
El worker procesa la cola de forma asincrona; checkout solo encola y SAP no
bloquea la venta.

Verifique los logs sin buscar ni imprimir credenciales:

```bash
kubectl -n r7-sap-dev logs deployment/backend --since=15m \
  | grep 'integration.sap_sync.worker'
```

Liste eventos por API:

```bash
curl -sS \
  "${BACKEND_URL}/api/v1/integrations/sap-sync/SAP01/events?pgn=1&lmt=50" \
  -H "Authorization: Bearer <jwt>"
```

O revise la cola directamente:

```sql
SELECT
  id_evento,
  tipo_evento,
  tipo_ref_r7,
  id_ref_r7,
  estado,
  numero_intentos,
  max_reintentos,
  proximo_reintento_en,
  mensaje_error,
  fyh_nrd,
  fyh_urd
FROM sap_evento_sincronizacion
WHERE cod_emp = 'SAP01'
ORDER BY fyh_nrd DESC
LIMIT 100;
```

La trazabilidad e idempotencia del documento se revisan en
`sap_enlace_documento`:

```sql
SELECT
  id_enlace,
  id_evento,
  tipo_ref_r7,
  id_ref_r7,
  sap_object,
  sap_doc_entry,
  sap_doc_num,
  sap_series,
  estado_sincronizacion,
  numero_intentos,
  mensaje_error,
  fyh_nrd,
  fyh_urd
FROM sap_enlace_documento
WHERE cod_emp = 'SAP01'
ORDER BY fyh_nrd DESC
LIMIT 100;
```

## Discovery, provisioning y mapeos pendientes

Los impuestos se leen de `SalesTaxCodes`. Las series se leen mediante
`SeriesService_GetDocumentSeries`; el recurso `Series` no existe en este
Service Layer.

El provisioning de UDF y articulos es aditivo e idempotente:

```bash
SAP_PROVISION_ENABLED=true node dist/scripts/sap-provision.js \
  --cod-emp SAP01 --item-group-code 100 --warehouse 01

SAP_PROVISION_ENABLED=true node dist/scripts/sap-provision.js \
  --cod-emp SAP01 --item-group-code 100 --warehouse 01 --apply
```

Antes de sincronizar documentos reales, complete con valores confirmados en
SAP Business One:

| Mapeo | Tabla/campo |
| --- | --- |
| TaxCode para `IVA` | `sap_mapeo_impuesto.sap_tax_code` |
| Series para factura/notas | `sap_mapeo_serie.sap_series` |
| WarehouseCode para `SAP01-DISP` | `sap_mapeo_bodega.sap_warehouse_code` |
| CardCode del cliente por defecto o directo | `sap_mapeo_cliente.sap_card_code` |
| PaymentMethod, cuenta, tarjeta o banco | `sap_mapeo_metodo_pago` |

Tarjeta ya tiene un mapping descubierto. Efectivo sigue inactivo hasta que
finanzas SAP apruebe una cuenta contable activa; no se debe inventar ese codigo.

Si SAP exige UDFs, configure `sap_mapeo_udf`; no escriba nombres o valores UDF
directamente en handlers. Los productos se relacionan con SAP mediante
`inv_art_var.cod_sap`.

Una factura SAP de venta ya afecta inventario. No genere adicionalmente un
`InventoryGenExits` por la misma venta POS.

## Jenkins, Argo CD, exposicion y frontend

SAP se despliega separado de las apps DEV existentes. No use `backend-dev`,
`frontend-dev` ni el Ingress `/` actual para validar SAP. Jenkins debe construir
imagenes desde pipelines nuevos de la rama `Integracion-Sap` y actualizar solo
los overlays `apps/*/overlays/sap-dev`.

Argo CD usa apps nuevas:

- `backend-sap-dev` -> `apps/backend/overlays/sap-dev` -> namespace `r7-sap-dev`
- `frontend-sap-dev` -> `apps/frontend/overlays/sap-dev` -> namespace `r7-sap-dev`
- `routing-sap-dev` -> `apps/routing/overlays/sap-dev` -> Kong host dedicado

Kong actual expone `/api` y `/` para el ambiente DEV normal en `32047`. Para no
competir con ese Ingress, `sap-dev` usa un host dedicado en el mismo Kong:

```text
Frontend DEV actual: http://98.85.131.168:32047/
Frontend SAP DEV:    http://sap-dev.98.85.131.168.nip.io:32047/
Panel SAP aislado:   http://sap-dev.98.85.131.168.nip.io:32047/administration/sap
API SAP DEV:         http://sap-dev.98.85.131.168.nip.io:32047/api/v1/health
```

```bash
kubectl -n argocd get applications backend-sap-dev frontend-sap-dev routing-sap-dev
kubectl -n r7-sap-dev rollout status deployment/backend
kubectl -n r7-sap-dev rollout status deployment/frontend
kubectl -n r7-sap-dev get ingress r7-sap
kubectl -n r7-sap-dev get svc backend frontend
kubectl -n r7-sap-dev get deploy backend frontend \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[0].image}{"\n"}{end}'
```
