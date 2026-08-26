# Valores requeridos en el Secret QA

Estos valores deben cargarse en el Secret Kubernetes `backend-secrets` del
namespace `r7-qa`. Este archivo es solamente una plantilla y no debe contener
la contraseña real.

```text
POWERBI_USERNAME=powerbi_r7
POWERBI_API_KEY=PON_TU_CONTRASENA
POWERBI_COD_EMP=PON_TU_EMPRESA
```

La contraseña debe tener como mínimo 32 caracteres. Después de actualizar el
Secret, reiniciar el Deployment `backend` para que sus pods reciban los nuevos
valores.
