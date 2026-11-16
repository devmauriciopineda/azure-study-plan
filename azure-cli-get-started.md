# Azure CLI: primeros pasos

## 1. ¿Qué es Azure CLI?

Azure CLI es una herramienta de línea de comandos para administrar recursos de
Microsoft Azure. Permite iniciar sesión, consultar la configuración, crear y
modificar recursos, automatizar tareas y repetir operaciones de forma más
consistente que una secuencia manual de clics.

La CLI no es una suscripción ni un recurso de Azure. Es únicamente una
herramienta cliente que envía operaciones a Azure usando la identidad con la
que se inicia sesión.

Esta guía usa **PowerShell en Windows**, aunque la mayoría de los comandos
también funciona en Bash y Azure Cloud Shell. En PowerShell, el carácter de
continuación de línea es la comilla invertida: `` ` ``.

> **Seguridad:** nunca pegar contraseñas, tokens, claves de Storage,
> connection strings o SAS en comandos, scripts o repositorios. Usar
> Microsoft Entra ID y mecanismos de autenticación autorizados.

## 2. Instalar Azure CLI

Instalar Azure CLI desde la documentación oficial de Microsoft. Después de la
instalación, abrir una nueva ventana de PowerShell para que el sistema
reconozca el comando.

Comprobar que está instalada:

```powershell
az version
```

Si aparece información de versión, la CLI está disponible. Si PowerShell
indica que `az` no existe, cerrar y abrir la terminal; si el problema
continúa, revisar la instalación y la variable `PATH`.

## 3. Obtener ayuda

La ayuda integrada es el primer recurso para conocer la sintaxis:

```powershell
az --help
az account --help
az group --help
az storage --help
```

La estructura general es:

```text
az <grupo-de-comandos> <operación> [parámetros]
```

Ejemplos:

```text
az account show
az group list
az storage account show
```

Para consultar ejemplos de uso relacionados con una operación:

```powershell
az find "az storage account create"
az find "list resource groups"
```

`az find` puede consultar ejemplos publicados para la CLI. No ejecutar
automáticamente un comando encontrado sin revisar antes sus parámetros,
suscripción, región y posibles costos.

## 4. Iniciar sesión

El comando básico de autenticación es:

```powershell
az login
```

Normalmente se abre un navegador para completar el inicio de sesión. Si la
terminal no puede abrirlo, Azure CLI puede mostrar una URL y un código para
autenticarse desde otro dispositivo.

Después de iniciar sesión, listar los contextos disponibles:

```powershell
az account list --output table
```

El formato `table` facilita leer los resultados. Otros formatos útiles son:

```powershell
az account list --output json
az account list --output tsv
```

- `table`: salida legible para una persona.
- `json`: salida estructurada para inspección o automatización.
- `tsv`: valores separados por tabulaciones, útil para asignar variables.

## 5. Comprobar la cuenta y la suscripción activa

Mostrar el contexto seleccionado:

```powershell
az account show --output table
```

Consultar solo los datos más importantes:

```powershell
az account show `
  --query "{name:name,id:id,tenantId:tenantId,user:user.name}" `
  --output table
```

Antes de ejecutar cualquier comando que cree o modifique recursos, comprobar:

1. el nombre de la suscripción;
2. el ID de la suscripción;
3. el tenant;
4. la identidad conectada.

El nombre de una suscripción no siempre es único. Para operaciones
importantes es preferible confirmar el ID.

## 6. Seleccionar una suscripción

Cambiar la suscripción activa mediante su ID o nombre:

```powershell
az account set --subscription "<SUBSCRIPTION_ID_O_NOMBRE>"
```

Comprobar el resultado inmediatamente:

```powershell
az account show `
  --query "{name:name,id:id,tenantId:tenantId}" `
  --output table
```

El contexto activo se aplica a los comandos que no reciben explícitamente otra
suscripción. Un comando válido puede modificar la suscripción equivocada si
no se revisa el contexto.

Para evitar ambigüedad, también se puede indicar la suscripción en una
operación concreta cuando el comando lo admita:

```powershell
az group list `
  --subscription "<SUBSCRIPTION_ID>" `
  --output table
```

## 7. Consultar regiones y proveedores

### 7.1 Listar regiones disponibles

```powershell
az account list-locations `
  --query "[].{name:name,displayName:displayName}" `
  --output table
```

Filtrar una región concreta:

```powershell
az account list-locations `
  --query "[?name=='eastus2'].{name:name,displayName:displayName}" `
  --output table
```

Que una región aparezca en la lista no garantiza que todos los servicios,
cuotas, tamaños o características estén disponibles allí.

### 7.2 Comprobar proveedores de recursos

Un proveedor de recursos habilita un tipo de servicio en la suscripción. Por
ejemplo, `Microsoft.Storage` proporciona Storage Account y
`Microsoft.Network` proporciona VNet y NSG.

Consultar su estado:

```powershell
az provider show `
  --namespace Microsoft.Storage `
  --query registrationState `
  --output tsv

az provider show `
  --namespace Microsoft.Network `
  --query registrationState `
  --output tsv
```

El resultado esperado es `Registered`. Si aparece `NotRegistered`, registrar
el proveedor solo si la identidad tiene permiso para hacerlo:

```powershell
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Network
```

Registrar un proveedor no crea una Storage Account ni una VNet. Solo habilita
la suscripción para poder crear posteriormente recursos de ese tipo.

## 8. Consultar recursos sin crear nada

Listar grupos de recursos:

```powershell
az group list --output table
```

Listar todos los recursos de la suscripción:

```powershell
az resource list --output table
```

Filtrar por tipo:

```powershell
az resource list `
  --resource-type Microsoft.Storage/storageAccounts `
  --output table
```

Consultar un grupo concreto:

```powershell
az group show `
  --name "<RESOURCE_GROUP_NAME>" `
  --output json
```

Consultar solo algunas propiedades:

```powershell
az group show `
  --name "<RESOURCE_GROUP_NAME>" `
  --query "{name:name,location:location,tags:tags}" `
  --output table
```

`--query` usa JMESPath para seleccionar o transformar parte de la respuesta.
Es preferible consultar solo los datos necesarios cuando la salida es grande.

## 9. Configurar valores predeterminados opcionales

Para no repetir algunos parámetros, Azure CLI permite definir valores
predeterminados:

```powershell
az configure --defaults group="<RESOURCE_GROUP_NAME>"
```

Desde ese momento, algunos comandos pueden omitir `--resource-group`.

Consultar la configuración:

```powershell
az config get
```

Eliminar el valor predeterminado:

```powershell
az configure --defaults group=""
```

Los valores predeterminados son locales a la configuración de Azure CLI. Antes
de ejecutar un comando importante, comprobarlos o escribir explícitamente el
grupo y la suscripción para evitar operar sobre un contexto inesperado.

## 10. Crear un primer grupo de recursos

Solo después de comprobar la suscripción y elegir una región, crear un grupo
de recursos:

```powershell
az group create `
  --name "<RESOURCE_GROUP_NAME>" `
  --location "eastus2" `
  --tags environment=dev application=learning owner=student `
  --output table
```

Consultar el grupo recién creado:

```powershell
az group show `
  --name "<RESOURCE_GROUP_NAME>" `
  --query "{name:name,location:location,tags:tags}" `
  --output json
```

Un grupo de recursos puede organizar recursos que compartan ciclo de vida,
pero no es por sí mismo una frontera de seguridad. Las etiquetas ayudan a
clasificar recursos y costos, pero no conceden permisos.

## 11. Crear y revisar un recurso de prueba

La creación de recursos puede generar costos. Antes de ejecutar cualquier
comando, comprobar precio, cuota, región y oferta de la suscripción.

Ejemplo: crear una cuenta StorageV2 con configuración básica:

```powershell
az storage account create `
  --name "<NOMBRE_STORAGE_UNICO>" `
  --resource-group "<RESOURCE_GROUP_NAME>" `
  --location "eastus2" `
  --sku Standard_LRS `
  --kind StorageV2 `
  --access-tier Hot `
  --allow-blob-public-access false `
  --https-only true `
  --min-tls-version TLS1_2 `
  --output table
```

El nombre de una Storage Account debe ser globalmente único, tener entre 3 y
24 caracteres y usar solo minúsculas y números.

Consultar sus propiedades:

```powershell
az storage account show `
  --name "<NOMBRE_STORAGE>" `
  --resource-group "<RESOURCE_GROUP_NAME>" `
  --query "{name:name,kind:kind,sku:sku.name,tier:accessTier,httpsOnly:enableHttpsTrafficOnly}" `
  --output table
```

Este ejemplo usa Blob Storage, pero no crea todavía un contenedor ni carga
archivos.

## 12. Variables de PowerShell

Las variables evitan repetir valores y facilitan reutilizar una secuencia:

```powershell
$resourceGroup = "rg-cli-learning"
$location = "eastus2"
$storageAccount = "<NOMBRE_STORAGE_UNICO>"

az group show `
  --name $resourceGroup `
  --query "{name:name,location:location}" `
  --output table
```

Obtener un valor de Azure para usarlo en otro comando:

```powershell
$subscriptionId = az account show --query id --output tsv
$tenantId = az account show --query tenantId --output tsv
```

Antes de usar una variable crítica, comprobar que no está vacía:

```powershell
$subscriptionId
$tenantId
```

No almacenar secretos en variables persistentes, archivos de perfil ni
historial de comandos.

## 13. Cerrar sesión

Cerrar la sesión de la cuenta actual:

```powershell
az logout
```

Cerrar una sesión concreta:

```powershell
az logout --username "<USUARIO_O_CORREO>"
```

Cerrar sesión no elimina recursos ni revierte cambios realizados. Solo
termina la autenticación local de Azure CLI.

## 14. Secuencia mínima recomendada

Para una sesión inicial, usar esta secuencia y revisar cada salida:

```powershell
az version
az login
az account list --output table
az account show --output table
az account set --subscription "<SUBSCRIPTION_ID>"
az account show --query "{name:name,id:id,tenantId:tenantId}" --output table
az group list --output table
az resource list --output table
```

No continuar con comandos de creación hasta confirmar que la suscripción y el
tenant son los correctos. Al terminar el trabajo, eliminar los recursos de
prueba que ya no se necesiten y cerrar sesión si el equipo es compartido.

## 15. Errores iniciales frecuentes

| Situación | Causa probable | Acción |
|---|---|---|
| `az` no se reconoce | CLI no instalada o `PATH` sin actualizar | Reabrir PowerShell y revisar la instalación. |
| No aparece ninguna suscripción | Cuenta equivocada, tenant incorrecto o acceso inexistente | Ejecutar `az login` y revisar el directorio seleccionado. |
| `AuthorizationFailed` | La identidad no tiene el permiso requerido | Revisar IAM y solicitar el permiso mínimo necesario. |
| El recurso aparece en otra suscripción | Contexto activo incorrecto | Ejecutar `az account show` y `az account set`. |
| Nombre de Storage no disponible | Los nombres de Storage son globales | Elegir otro nombre que cumpla las restricciones. |
| `InvalidResourceName` | Nombre con mayúsculas, guiones o longitud no válida | Revisar las reglas del servicio. |
| `NotRegistered` | Proveedor de recursos no habilitado | Registrar el namespace con permisos adecuados. |
| El comando devuelve demasiada información | Salida sin filtro | Usar `--query` y elegir `--output table`, `json` o `tsv`. |

