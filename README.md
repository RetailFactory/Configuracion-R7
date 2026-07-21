# Configuracion-R7
Repositorio de **configuración general de conexiones** de Retail7, que centraliza los parámetros y definiciones necesarias para la comunicación entre servicios, aplicaciones y entornos del sistema POS.

## FEL en contingencia

El Deployment del backend obtiene `FEL_CONTINGENCY_ENABLED` del Secret
`backend-secrets`. La clave es opcional para permitir un despliegue seguro con
el valor por defecto `false`, pero debe existir explícitamente antes de habilitar
la funcionalidad en un ambiente.

La identidad `FEL_CONTINGENCY_INSTALLATION_ID` y `LOCAL_DEVICE_ID` pertenecen a
cada Host local. No deben agregarse al Secret compartido del backend Cloud ni
reutilizarse entre sucursales.

Para anexar la bandera sin reemplazar las demás claves del Secret:

```bash
kubectl -n r7-dev patch secret backend-secrets \
  --type merge \
  --patch '{"stringData":{"FEL_CONTINGENCY_ENABLED":"false"}}'
```

Use el namespace correspondiente en QA o producción. No registre el contenido
del Secret en Git. Para comprobar solamente que la clave existe, sin imprimir
su valor:

```bash
kubectl -n r7-dev get secret backend-secrets \
  -o go-template='{{if index .data "FEL_CONTINGENCY_ENABLED"}}present{{else}}missing{{end}}{{"\n"}}'
```

Mantenga la bandera en `false` hasta que la migración 139 exista en Cloud y en
el Host, el rango esté preparado/activado y las pruebas offline hayan pasado.
