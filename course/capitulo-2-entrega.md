# Entrega del capítulo 2. Modelo de ejecución para la API de documentos

## Alcance y supuestos

Esta propuesta continúa el entorno `dev` del capítulo 1. Diseña el código y
su ejecución, pero no crea recursos reales, no configura una suscripción y no
incluye credenciales. Se asume que el código usa Python 3.11, FastAPI para la
API HTTP y Blob Storage para documentos. La disponibilidad exacta de Python
3.11, del plan elegido y de la integración de red debe confirmarse antes de
desplegar.

Se propone separar los hosts, pero compartir un paquete de dominio pequeño:
la API y la tarea programada reutilizan reglas para identificar documentos
temporales vencidos, mientras sus adaptadores de entrada permanecen
independientes. El mismo artefacto lógico y el mismo contrato de configuración
se promueven entre `dev` y `test`, con valores distintos por ambiente.

## 1. Decisión de arquitectura

| Carga | Servicio elegido | Razón |
|---|---|---|
| API REST para registrar, consultar y eliminar documentos | Azure App Service Linux | La API tiene varias rutas HTTP relacionadas, necesita un proceso web continuo y se beneficia de un punto de entrada, configuración y diagnóstico unificados. |
| Limpieza nocturna de temporales | Azure Functions con timer trigger | Es una ejecución periódica, acotada y sin necesidad de mantener un proceso HTTP activo durante todo el día. El costo y el escalamiento se vinculan mejor con invocaciones. |
| Reglas de negocio compartidas | Paquete Python común dentro del repositorio | Evita duplicar la regla de vencimiento, pero no hace que App Service y Functions compartan estado en memoria. El estado persistente queda en Blob Storage. |

### Comparación breve

| Criterio | App Service | Azure Functions | Decisión aplicada |
|---|---|---|---|
| Modelo de ejecución | Aplicación web con proceso HTTP persistente | Funciones activadas por eventos | API en App Service; limpieza en Functions. |
| Escalamiento | Capacidad y reglas del App Service plan | Plan de Functions y demanda de invocaciones | La API recibe tráfico continuo; el timer solo necesita capacidad durante su ejecución. |
| Operación | Un proceso, rutas y configuración centralizadas | Triggers y funciones independientes | Se evita cargar una tarea nocturna dentro del proceso web. |
| Configuración | Application settings por aplicación/slot | Application settings de la Function App | Ambos reciben el mismo esquema, con valores separados por ambiente. |
| Costo | Se paga capacidad del plan aunque no haya solicitudes | Adecuado para trabajo intermitente, sujeto al plan y límites | Se acepta el costo fijo de la API y se evita usarlo para la limpieza. |
| Límites | Menos acoplado a duración de una invocación | Duración, arranque en frío y límites dependen del plan | La limpieza se mantiene acotada; si deja de caber, se reevalúa el plan. |
| Despliegue | Comando de inicio explícito y proceso web | Host administra el trigger y el punto de entrada | Cada servicio tiene una prueba de arranque y ejecución propia. |

Compartir un App Service plan entre API y otros procesos podría reducir costo,
pero produciría competencia por CPU/memoria y acoplaría sus escalas. En esta
etapa se mantiene la limpieza en Functions. La API y la Function App sí pueden
compartir la cuenta de almacenamiento de `dev`, siempre que usen contenedores
y permisos definidos; no comparten variables en memoria ni archivos locales.

## 2. Estructura de ejecución

### API en App Service

- **Runtime:** Python 3.11 en App Service Linux; validar disponibilidad y
  soporte del runtime en la región antes del despliegue.
- **Punto de entrada:** `app.main:app`, donde `app` es la instancia ASGI de
  FastAPI.
- **Dependencias:** `requirements.txt` con FastAPI, servidor ASGI y
  bibliotecas Azure SDK necesarias. Las versiones deben fijarse o acotarse y
  validarse en pruebas.
- **Comando de arranque propuesto:**

  ```text
  gunicorn --bind=0.0.0.0:$PORT --workers=2 --timeout=60 -k uvicorn.workers.UvicornWorker app.main:app
  ```

  `$PORT` lo proporciona el entorno. No se usa el servidor de desarrollo de
  FastAPI/Uvicorn como comando de producción. El número de workers es inicial y
  debe ajustarse con mediciones de memoria y concurrencia.
- **Artefacto:** código Python, `requirements.txt`, configuración no sensible
  y archivos necesarios para arrancar. Se excluyen `.env`, claves, tokens,
  archivos subidos, temporales, cachés y pruebas innecesarias.
- **Salud:** una ruta `GET /health` comprueba que el proceso inició. No debe
  fingir que Blob Storage está disponible si no se verificó; una comprobación
  de dependencia separada puede reportar su estado sin ocultar el error.

### Function App para limpieza

- **Runtime:** Python 3.11 en una Function App separada por ambiente.
- **Trigger:** timer trigger con expresión `0 0 2 * * *`, cada día a las
  02:00 UTC. La zona horaria y el horario de mantenimiento deben confirmarse
  con operaciones.
- **Punto de entrada conceptual:** `function_app.py`, función
  `cleanup_expired_files`.
- **Entrada:** contexto del timer, incluida la indicación de ejecución
  atrasada cuando el host la proporcione.
- **Salida:** no responde HTTP; produce métricas y logs estructurados con
  cantidad examinada, cantidad eliminada y duración. Los nombres de blobs y
  credenciales no se escriben en logs.
- **Estado:** la lista de blobs y su `last_modified`/metadatos viven en Blob
  Storage. La memoria de una instancia no se considera persistente.

La Function App y App Service reciben identidad administrada distinta por
ambiente o una identidad con permisos claramente auditados. En ambos casos,
el rol de datos se limita al contenedor necesario.

## 3. Contrato mínimo de la API

Los ejemplos usan JSON y una identificación lógica `document_id`. El esquema
de persistencia queda fuera de esta actividad; el contrato solo define lo que
observa un cliente.

| Método y ruta | Entrada | Respuesta esperada | Error principal |
|---|---|---|---|
| `GET /health` | Sin cuerpo | `200` con `{"status":"ok","version":"<build>"}` | `503` si el proceso no puede atender o una comprobación explícita de dependencia falla. |
| `POST /documents` | `multipart/form-data`: archivo requerido, `content_type` permitido y opcionalmente `temporary=true` | `201` con `{"document_id":"...","status":"stored"}` y encabezado `Location` | `400` por multipart inválido o tipo no permitido; `413` por tamaño mayor al límite; `409` si el identificador idempotente ya existe; `503` por fallo de Blob Storage. |
| `GET /documents/{document_id}` | Identificador de ruta; opcional `download=false` | `200` con metadatos y referencia de descarga; si se solicita descarga, contenido con `Content-Type` correcto | `400` por identificador inválido; `404` si no existe; `403` si el principal no está autorizado; `503` por dependencia no disponible. |
| `DELETE /documents/{document_id}` | Identificador de ruta | `204` sin cuerpo; la operación es idempotente si el documento ya fue eliminado | `400` por identificador inválido; `404` si la política exige reportar ausencia; `403` por autorización; `503` por fallo transitorio de almacenamiento. |

La validación ocurre antes de escribir en Blob Storage: tamaño máximo, nombre
normalizado, tipo permitido y ausencia de rutas controladas por el cliente.
Los errores devuelven un formato estable, por ejemplo
`{"error":{"code":"invalid_request","message":"..."}}`, sin stack traces,
tokens ni connection strings. La autenticación de usuarios finales no se
diseña todavía; `403` representa la autorización futura del consumidor.

## 4. Diseño de triggers

### HTTP trigger de la API

App Service recibe solicitudes HTTPS en las rutas anteriores. El adaptador HTTP
valida método, ruta, encabezados, tamaño y cuerpo; la lógica de documentos
llama al cliente de Blob Storage con la identidad del proceso. Un timeout de
cliente no debe provocar reintentos ciegos de una subida: `POST` acepta un
identificador de idempotencia para que una repetición conocida devuelva el
resultado existente o un conflicto controlado.

El proceso no debe almacenar documentos en el sistema de archivos local de
App Service. El artefacto desplegado contiene código y recursos estáticos; los
archivos subidos y temporales van a Blob Storage. El disco local puede usarse
solo como espacio efímero durante una solicitud, con límites y limpieza.

### Timer trigger de limpieza

| Propiedad | Diseño |
|---|---|
| Programación | `0 0 2 * * *` (02:00 UTC cada día); el horario local se confirma antes de producción. |
| Entrada | Contexto del timer y configuración `TEMP_RETENTION_DAYS`; no recibe datos de usuario. |
| Selección | Buscar blobs del contenedor `temp` con marca temporal o metadato de expiración anterior al umbral. |
| Salida | Métricas/logs: inicio, fin, cantidad examinada, eliminada, omitida y error resumido. |
| Duración inicial | Menor que 10 minutos como supuesto; se debe medir con volumen real y revisar el límite del plan. |
| Repetición | La eliminación es idempotente: un blob ausente se considera ya limpio y no se vuelve a crear. |
| Solapamiento | Adquirir un lease corto sobre un blob de bloqueo antes de listar/eliminar; si no se obtiene, registrar `cleanup_skipped_overlap` y terminar sin borrar. Liberar el lease en `finally`. |
| Fallo parcial | Continuar con elementos independientes, registrar el conteo de errores y devolver estado fallido si quedaron elementos sin procesar; la siguiente ejecución reintenta por criterio de vencimiento. |

El lease evita dos ejecuciones del mismo ambiente modificando simultáneamente
el conjunto de temporales. La regla de selección por vencimiento y el borrado
repetible protegen frente a una ejecución atrasada o repetida. Si el volumen
supera la duración disponible, se debe particionar por prefijo o mover el
trabajo a una arquitectura asíncrona posterior, no aumentar reintentos sin
límite.

## 5. Configuración por ambiente

Los valores siguientes son ejemplos no secretos. App Service y Functions
inyectan estas variables desde su configuración del ambiente; no se versiona
un archivo con valores reales.

| Variable | `dev` | `test` | Validación |
|---|---|---|---|
| `APP_ENV` | `dev` | `test` | Obligatoria; solo admite ambientes conocidos. |
| `APP_VERSION` | `local-or-build-id` | `build-id` | Obligatoria para diagnóstico; no contiene secretos. |
| `AZURE_REGION` | `eastus2` | `eastus2` | Obligatoria; debe coincidir con la región del recurso. |
| `BLOB_ACCOUNT_URL` | `https://stdocsdeveus2.blob.core.windows.net` | `https://stdocstest...blob.core.windows.net` | Obligatoria; debe ser HTTPS y pertenecer al ambiente. El segundo valor es ilustrativo, no un recurso existente. |
| `BLOB_CONTAINER_DOCUMENTS` | `documents` | `documents` | Obligatoria; no vacía y coincide con el contenedor autorizado. |
| `BLOB_CONTAINER_TEMP` | `temp` | `temp` | Obligatoria para el timer; no vacía. |
| `MAX_DOCUMENT_BYTES` | `10485760` | `10485760` | Entero positivo dentro del límite aprobado. |
| `ALLOWED_CONTENT_TYPES` | `application/pdf,image/png,text/plain` | `application/pdf,image/png,text/plain` | Lista no vacía; rechaza tipos no declarados. |
| `TEMP_RETENTION_DAYS` | `7` | `14` | Entero positivo; no debe exceder la política aprobada. |
| `LOG_LEVEL` | `INFO` | `INFO` | Enumeración permitida (`DEBUG`, `INFO`, `WARNING`, `ERROR`); `DEBUG` no se habilita en producción por defecto. |
| `CORS_ALLOWED_ORIGINS` | `http://localhost:3000` | `https://test.example.invalid` | Lista explícita; no se acepta `*` para rutas con documentos. |
| `CLEANUP_SCHEDULE` | `0 0 2 * * *` | `0 0 2 * * *` | Expresión válida y zona horaria documentada. |

`BLOB_ACCOUNT_URL` no es una credencial; aun así, se valida para evitar que
`dev` apunte por accidente a `test` o producción. La identidad administrada,
claves, SAS y otros secretos se configuran mediante el mecanismo seguro del
servicio y no aparecen en esta tabla ni en el repositorio.

Si falta una variable obligatoria, el proceso falla explícitamente durante el
arranque con un error de configuración identificable, no inicia tráfico
parcial y no usa un valor silencioso por defecto. La Function App debe marcar
la ejecución como configuración inválida y producir un log sin el valor
sensible. Las variables opcionales pueden tener un valor documentado, pero
nunca se aplican valores alternativos a una ruta de almacenamiento o permiso.

## 6. Acceso desde Python al Blob Storage

El SDK consume el servicio en tiempo de ejecución; no administra la
infraestructura como lo hace Azure CLI. El código de aplicación debe usar una
credencial distinta según el ambiente, sin cambiar la lógica del cliente:

```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient


def create_blob_service(account_url: str) -> BlobServiceClient:
    if not account_url.startswith("https://"):
        raise ValueError("BLOB_ACCOUNT_URL debe usar HTTPS")

    # Local: usa la identidad del desarrollador configurada en Azure CLI/IDE.
    # App Service/Functions: usa la identidad administrada del recurso.
    credential = DefaultAzureCredential()
    return BlobServiceClient(account_url=account_url, credential=credential)


def upload_document(service: BlobServiceClient, container: str,
                    blob_name: str, stream, content_type: str) -> None:
    blob = service.get_blob_client(container=container, blob=blob_name)
    blob.upload_blob(
        stream,
        overwrite=False,
        content_settings={"content_type": content_type},
    )
```

El fragmento es pseudocódigo orientativo: el tipo exacto de
`content_settings`, los timeouts, reintentos y cierre deben validarse contra la
versión fijada del SDK. `DefaultAzureCredential` no concede permisos: en local
la identidad del desarrollador necesita autorización sobre el contenedor de
`dev`; en App Service y Functions la identidad administrada necesita, como
mínimo, `Storage Blob Data Contributor` en el contenedor que escribe. El SDK
debe configurar timeouts y reintentos finitos, capturar errores específicos y
exponer un error controlado al contrato HTTP, sin un `except` general que
oculte la causa.

## 7. Flujo de despliegue y verificación

La secuencia es manual y reproducible para esta actividad; no construye un
pipeline CI/CD.

1. Preparar una rama o revisión identificable con `app/`, `function_app.py`,
   `requirements.txt`, pruebas y configuración de ejemplo sin valores
   sensibles.
2. Ejecutar localmente pruebas unitarias, validación de tipos/lint y pruebas
   de contrato. Confirmar que no se incluyen `.env`, claves, tokens,
   archivos subidos, cachés ni temporales.
3. Construir el artefacto desde una revisión fijada y registrar su
   identificador. No alterar manualmente el contenido después de las pruebas.
4. Confirmar la suscripción, el grupo `rg-docs-dev-eus2`, la región y el
   ambiente destino antes de publicar.
5. Configurar App Service con runtime Python, comando Gunicorn, variables de
   `dev` y la identidad administrada. Configurar la Function App con el timer,
   las mismas variables requeridas y su identidad.
6. Publicar el artefacto en App Service y el paquete de la Function App
   mediante el mecanismo aprobado (Portal o Azure CLI). La actividad no
   requiere ejecutar los comandos.
7. Revisar logs de arranque: versión, ambiente y nombres lógicos de recursos,
   pero nunca credenciales ni cuerpos de documentos.
8. Ejecutar pruebas de humo contra un ambiente sin datos reales:

   - `GET /health` devuelve `200` y la versión esperada.
   - `POST /documents` con un archivo válido devuelve `201` y el blob aparece
     en `documents`.
   - `POST /documents` con tipo o tamaño inválido devuelve `400` o `413` y no
     crea un blob.
   - `GET /documents/{document_id}` devuelve metadatos del documento creado y
     `404` para un identificador inexistente.
   - `DELETE /documents/{document_id}` devuelve `204` y repetirlo mantiene el
     comportamiento idempotente definido.
   - La aplicación puede acceder a Blob Storage con su identidad, pero no
     cambiar configuración de la cuenta ni leer otro contenedor no autorizado.
   - Ejecutar la limpieza en una ventana controlada o esperar la programación;
     verificar lease, cantidad examinada/eliminada y que una segunda ejecución
     concurrente se omita sin borrar datos.
   - Revisar logs de éxito, error de configuración, error de dependencia y
     duración; confirmar que no contienen secretos ni contenido sensible.

9. Comparar el identificador de versión observado con el artefacto publicado y
   registrar la evidencia. Si falla arranque, configuración, identidad o
   contrato, detener la promoción y corregir antes de reutilizar el ambiente.

## 8. Supuestos, riesgos y límites

### Supuestos

- Python 3.11, FastAPI y el modelo de despliegue ASGI son compatibles con el
  App Service y la región seleccionados.
- La limpieza nocturna procesa un volumen que termina en menos de 10 minutos.
- Los temporales tienen `last_modified` o metadatos suficientes para calcular
  vencimiento; la fuente definitiva no es el sistema de archivos local.
- Existe un contenedor `documents` y otro `temp` en el almacenamiento de
  `dev`/`test`, con RBAC asignado a las identidades correctas.
- El horario 02:00 UTC minimiza impacto; debe confirmarse con usuarios y
  operaciones.

### Riesgos y mitigaciones

- **Exceso de permisos:** asignar `Contributor` a la identidad de la API
  permitiría modificar infraestructura. Mitigación: identidad administrada y
  solo rol de datos sobre el contenedor requerido.
- **Secreto en el artefacto o logs:** connection strings, SAS o cuerpos de
  documentos podrían filtrarse. Mitigación: `DefaultAzureCredential`,
  configuración segura, exclusiones de empaquetado y logs estructurados sin
  datos sensibles.
- **Ejecución solapada o reintentos:** dos timers podrían borrar o contabilizar
  el mismo conjunto. Mitigación: lease de bloqueo, selección idempotente y
  reintentos finitos.
- **Límite de tiempo o costo:** el volumen de temporales puede superar el
  tiempo de Functions o generar muchas operaciones de listado/borrado.
  Mitigación: métricas de duración, lotes/prefijos y revisión del plan antes de
  producción.
- **Pérdida de archivos:** el almacenamiento local de la aplicación se pierde
  en reinicios o despliegues. Mitigación: Blob Storage como fuente de archivos,
  no disco local.
- **Configuración cruzada entre ambientes:** una URL equivocada puede hacer
  que `dev` lea o borre documentos de `test`. Mitigación: validar el ambiente,
  URL y contenedor al arranque y usar identidades separadas.

### Límites y decisiones pendientes

- No se define todavía autenticación de usuarios finales, base de datos,
  mensajería, contenedores, CI/CD ni observabilidad avanzada.
- Deben confirmarse duración máxima de limpieza, tamaño máximo de documentos,
  estrategia de escala de App Service, cantidad de workers, zona horaria,
  permisos exactos y presupuesto.
- Si la API requiere operaciones largas, streaming grande o alta
  disponibilidad, se debe revisar el plan y el modelo antes de aceptar el
  comando inicial.
- Antes de producción deben aprobarse residencia de datos, retención,
  redundancia, recuperación ante desastre y política de exposición pública.

