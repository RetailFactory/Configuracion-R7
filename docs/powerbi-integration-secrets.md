# Secret de integración Power BI

Antes de desplegar Backend-R7 en QA, agregar al Secret `backend-secrets` del
namespace `r7-qa`:

- `POWERBI_API_KEY`: clave aleatoria de al menos 32 caracteres.
- `POWERBI_COD_EMP`: empresa autorizada para Power BI.

Para autorizar varias empresas se puede usar `POWERBI_ALLOWED_COMPANIES`
(separadas por comas). Ambas variables de alcance ya están referenciadas de
forma opcional en el Deployment; debe configurarse al menos una. Los valores
reales no se versionan.
