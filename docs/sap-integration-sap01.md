# Integracion SAP Business One para SAP01

Esta guia aplica al backend R7 desplegado para `cod_emp=SAP01`. La URL de
Service Layer y `CompanyDB` se almacenan en la configuracion de base de datos:

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
| `SAP_WORKER_ENABLED` | Activa el polling asincrono de `sap_sync_event` | `false` |
| `SAP_WORKER_BATCH_SIZE` | Maximo de eventos reclamados por ciclo | `10` |
| `SAP_WORKER_POLL_INTERVAL_MS` | Intervalo entre ciclos del worker | `10000` |

El overlay `dev` establece `SAP_WORKER_ENABLED="true"`. En QA y produccion la
variable queda controlada por `backend-secrets`; si la clave no existe, el
backend conserva el default `false`.

Use `SAP_HTTP_ALLOW_SELF_SIGNED="true"` unicamente en dev/QA cuando el Service
Layer de pruebas use un certificado autofirmado. No lo habilite en produccion.

## Crear o actualizar backend-secrets

No escriba valores reales en este repositorio. Ejecute los comandos desde una
terminal autorizada y reemplace los placeholders al momento de ejecutarlos.

Ejemplo solicitado para crear/aplicar las credenciales:

```bash
kubectl -n r7-dev create secret generic backend-secrets \
  --from-literal=SAP01_SAP_USERNAME='<usuario>' \
  --from-literal=SAP01_SAP_PASSWORD='<password>' \
  --dry-run=client -o yaml | kubectl apply -f -
```

`backend-secrets` tambien contiene `DATABASE_URL`, `REDIS_URL`,
`RABBITMQ_URL`, claves JWT y otras variables. Si el Secret ya existe y no se
administra con un manifiesto que incluya todas sus claves, actualice solo las
claves SAP para no reconstruir ni perder las demas:

```bash
kubectl -n r7-dev patch secret backend-secrets \
  --type merge \
  --patch '{"stringData":{"SAP01_SAP_USERNAME":"<usuario>","SAP01_SAP_PASSWORD":"<password>"}}'
```

Los controles opcionales tambien pueden administrarse en el Secret:

```bash
kubectl -n r7-dev patch secret backend-secrets \
  --type merge \
  --patch '{"stringData":{"SAP_SESSION_CACHE_TTL_SECONDS":"1200","SAP_WORKER_BATCH_SIZE":"10","SAP_WORKER_POLL_INTERVAL_MS":"10000"}}'
```

Despues de actualizar el Secret, reinicie el deployment para que el proceso
reciba los nuevos valores:

```bash
kubectl -n r7-dev rollout restart deployment/backend
kubectl -n r7-dev rollout status deployment/backend
```

## Ejecutar el seed SAP01

Ejecute primero el flujo normal de migraciones de la version desplegada. El
seed requiere las tablas de configuracion ERP y sincronizacion SAP. Si el
ambiente puede contener nombres historicos en ingles, aplique tambien
`89_reconcile_sap_spanish_schema_and_complete_operational_sync.sql`.

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

Dev ya activa el worker en
`apps/backend/overlays/dev/kustomization.yaml`. Para QA o produccion, agregue
`SAP_WORKER_ENABLED="true"` a `backend-secrets` y reinicie el deployment. El
worker procesa la cola de forma asincrona; checkout solo encola y SAP no
bloquea la venta.

Verifique los logs sin buscar ni imprimir credenciales:

```bash
kubectl -n r7-dev logs deployment/backend --since=15m \
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

## Mapeos pendientes

Antes de sincronizar documentos reales, complete con valores confirmados en
SAP Business One:

| Mapeo | Tabla/campo |
| --- | --- |
| TaxCode para `IVA12` | `sap_mapeo_impuesto.sap_tax_code` |
| Series para `FACTURA` / `Invoices` | `sap_mapeo_serie.sap_series` |
| WarehouseCode para `SAP01-DISP` | `sap_mapeo_bodega.sap_warehouse_code` |
| CardCode del cliente por defecto o directo | `sap_mapeo_cliente.sap_card_code` |
| PaymentMethod, cuenta, tarjeta o banco | `sap_mapeo_metodo_pago` |

Si SAP exige UDFs, configure `sap_mapeo_udf`; no escriba nombres o valores UDF
directamente en handlers. Los productos se relacionan con SAP mediante
`inv_art_var.cod_sap`.

Una factura SAP de venta ya afecta inventario. No genere adicionalmente un
`InventoryGenExits` por la misma venta POS.

## Panel operativo

El frontend integrado expone `/administration/sap` para usuarios con
`ADMIN_COMPANY_VIEW`. Discovery, aprobacion de bodegas, creacion/vinculacion y
reintentos requieren `ADMIN_SYSTEM_CONFIG`.

El navegador usa el BFF de mismo origen:

- `/bff/integrations/sap/*`
- `/bff/integrations/sap-sync/*`

No se requieren variables frontend nuevas ni se exponen credenciales SAP.
