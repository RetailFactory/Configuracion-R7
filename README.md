# Configuracion-R7
Repositorio de **configuración general de conexiones** de Retail7, que centraliza los parámetros y definiciones necesarias para la comunicación entre servicios, aplicaciones y entornos del sistema POS.

## FEL en contingencia

El Deployment del backend obtiene `FEL_CONTINGENCY_ENABLED`,
`FEL_EXTERNAL_ENABLED` y `FEL_EXTERNAL_AUTO_CERTIFY` del Secret
`backend-secrets`. En `r7-dev` las tres deben permanecer en `true` para probar
la emisión offline y la certificación automática al recuperar conectividad.

Los rangos FEL pertenecen únicamente a empresa y sucursal. No agregue
`FEL_CONTINGENCY_INSTALLATION_ID`: esa variable fue eliminada. El
`LOCAL_DEVICE_ID` sigue siendo parte del protocolo general de sincronización
del Host, pero no se agrega al Secret Cloud ni se asocia al rango.

Para anexar la bandera sin reemplazar las demás claves del Secret:

```bash
kubectl -n r7-dev patch secret backend-secrets \
  --type merge \
  --patch '{"stringData":{"FEL_CONTINGENCY_ENABLED":"true","FEL_EXTERNAL_ENABLED":"true","FEL_EXTERNAL_AUTO_CERTIFY":"true"}}'
```

Use el namespace correspondiente en QA o producción. No registre el contenido
del Secret en Git. Para comprobar solamente que la clave existe, sin imprimir
su valor:

```bash
kubectl -n r7-dev get secret backend-secrets \
  -o go-template='{{if index .data "FEL_CONTINGENCY_ENABLED"}}present{{else}}missing{{end}}{{"\n"}}'
```

Antes de generar el EXE, confirme que las migraciones 139 y 140 existan en
Cloud y en el Host, y que el rango de la sucursal esté `ACTIVE`.
