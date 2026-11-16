# Capítulo 1. Implementación paso a paso del entorno base en Azure

## 0. Objetivo, punto de partida y límites

Esta guía implementa el entorno `dev` diseñado en
[capitulo-1-entrega.md](./capitulo-1-entrega.md), adaptándolo a las
condiciones reales de este ejercicio.

El punto de partida es completamente vacío:

- solo existe una cuenta personal de Azure;
- esa cuenta tiene acceso a una única suscripción;
- no existen otras suscripciones ni tenants;
- no existen grupos de recursos, usuarios adicionales, grupos de Microsoft
  Entra ID, identidades administradas ni recursos de Azure;
- la cuenta personal es la única persona que operará el laboratorio.

Por tanto, la cuenta personal se utiliza como identidad de arranque. No se
crean cuentas ficticias, otro tenant, otra suscripción ni el grupo
`grupo-backend-dev` solo para imitar el diseño organizativo. Ese grupo se
podrá crear en un capítulo posterior cuando haya más personas que administrar.
La identidad administrada `mi-api-docs-dev` tampoco se crea ahora: solo tiene
sentido cuando exista una API desplegada.

Al finalizar existirán únicamente:

1. el grupo de recursos `rg-docs-dev-eus2`;
2. una cuenta StorageV2 con Blob Storage y el contenedor privado `documents`;
3. una VNet con las subnets `snet-api` y `snet-private-endpoints`;
4. un NSG asociado a `snet-api`;
5. una asignación de datos al usuario actual sobre `documents`;
6. etiquetas, presupuesto y evidencias de validación.

No se crean API, App Service, Functions, máquina virtual, Private Endpoint,
Key Vault, base de datos, Service Bus, AKS ni servicios de IA. Tampoco se
configura una regla de entrada HTTPS para una API que todavía no existe.

> **Regla de seguridad:** no pegar en este documento ni en el repositorio
> contraseñas, tokens, claves de Storage, connection strings, SAS ni datos
> personales reales. Los nombres, identificadores y correos de los ejemplos
> son valores de configuración, no secretos.

## 1. Decisiones y costos antes de crear recursos

| Elemento | Decisión para este laboratorio |
|---|---|
| Región | `East US 2` (`eastus2`), solo después de comprobar disponibilidad y política de datos. |
| Storage | `StorageV2`, `Standard_LRS`, nivel `Hot`, TLS 1.2 y HTTPS obligatorio. |
| Red de Storage | Endpoint público habilitado, pero firewall con la IP pública actual; no se permite acceso anónimo. |
| VNet | `10.20.0.0/16`; API `10.20.1.0/24`; endpoints privados `10.20.2.0/24`. |
| Identidad | Usuario personal actual para operar el laboratorio y probar blobs. |
| Roles | El `Owner` existente se usa solo para bootstrap; no se otorga `Owner` a nadie más. El acceso de datos se limita al contenedor. |
| Datos | Solo archivos ficticios o de prueba. |
| Limpieza | El grupo completo se elimina al terminar cada sesión si no se necesita conservar datos. |

La capa gratuita o el crédito de la suscripción no significa que todo Azure
sea gratuito. Storage, operaciones, transferencia, reglas de ciclo de vida y
otros servicios pueden generar cargos. Antes de continuar, abrir
**Cost Management > Cost analysis** y revisar saldo, oferta, fecha de
vencimiento, límites y alertas de la suscripción. Si la suscripción no
permite crear recursos o la región no está autorizada, detenerse aquí.

`East US 2` es una hipótesis del diseño, no una garantía. Confirmar antes de
crear recursos que la región cumple residencia de datos, latencia, cuotas y
disponibilidad de Storage y Network. Si no cumple, elegir otra región y usarla
de forma consistente en todos los pasos.

## 2. Preparar las herramientas y comprobar el acceso

### 2.1 Instalar Azure CLI

Instalar Azure CLI desde la documentación oficial de Microsoft si aún no está
instalada. Abrir una nueva ventana de PowerShell y comprobarla:

```powershell
az version
```

Si el comando no existe, instalar Azure CLI antes de continuar. No intentar
solucionarlo habilitando accesos públicos ni usando claves de Storage.

### 2.2 Iniciar sesión y descubrir el único contexto disponible

Ejecutar:

```powershell
az login
az account list --output table
az account show `
  --query "{subscriptionName:name,subscriptionId:id,tenantId:tenantId,user:user.name}" `
  --output table
```

Debe aparecer exactamente la suscripción personal que se va a utilizar.
Copiar su `subscriptionId` de la salida, no escribirlo de memoria. El
`tenantId` mostrado es el tenant existente; no crear otro.

Si aparecen varias suscripciones, seleccionar la única autorizada para este
curso. Si no aparece ninguna, resolver el acceso antes de crear recursos:

```powershell
az account set --subscription "<SUBSCRIPTION_ID_CONFIRMADO>"
az account show --query "{name:name,id:id,tenantId:tenantId}" --output table
```

La cuenta personal puede ser `Owner` de su suscripción. Eso es una condición
de arranque, no una recomendación para producción. No crear otra identidad
para “reemplazar” al único operador ni asignar `Owner` a otra persona.

Comprobar el object ID del usuario actual:

```powershell
$developerObjectId = az ad signed-in-user show --query id --output tsv
$developerObjectId
```

Si este comando falla por una extensión o por permisos de consulta, continuar
solo con la creación del grupo y de los recursos. Antes de ejecutar la
asignación RBAC de datos de la sección 5.2, obtener el object ID desde
**Microsoft Entra ID > Users > tu usuario** en el portal o resolver el permiso
que impide la consulta. No ejecutar esa asignación con una variable vacía.

### 2.3 Comprobar disponibilidad de proveedores

Un proveedor de recursos es el componente de Azure que habilita un tipo de
servicio dentro de la suscripción. En este capítulo, `Microsoft.Storage`
proporciona Storage Account y Blob Storage, mientras que `Microsoft.Network`
proporciona VNet, subnets y NSG. Registrar un proveedor no crea recursos ni
genera el entorno: solo habilita la suscripción para poder crear después esos
tipos de recursos.

Consultar el estado:

```powershell
az provider show --namespace Microsoft.Storage `
  --query registrationState --output tsv
az provider show --namespace Microsoft.Network `
  --query registrationState --output tsv
```

Si alguno no está registrado, registrarlo y esperar a que finalice:

```powershell
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Network
az provider show --namespace Microsoft.Storage `
  --query registrationState --output tsv
az provider show --namespace Microsoft.Network `
  --query registrationState --output tsv
```

Continuar únicamente cuando ambos estados sean `Registered`.

## 3. Definir variables y comprobar la región

Ejecutar este bloque en la misma ventana de PowerShell. Sustituir solo los
valores entre `<...>`:

```powershell
$subscriptionId = "<SUBSCRIPTION_ID_CONFIRMADO>"
$location = "eastus2"
$resourceGroup = "rg-docs-dev-eus2"
$storageAccount = "<NOMBRE_STORAGE_UNICO>"
$vnetName = "vnet-docs-dev-eus2"
$apiSubnet = "snet-api"
$privateEndpointSubnet = "snet-private-endpoints"
$nsgName = "nsg-api"
$containerName = "documents"
$tags = "environment=dev application=document-api owner=equipo-backend costCenter=CC-0000"

az account set --subscription $subscriptionId
az account show --query "{name:name,id:id,tenantId:tenantId}" --output table
```

El nombre de Storage debe tener entre 3 y 24 caracteres, solo minúsculas y
números, y ser único en todo Azure. Por ejemplo:
`stdocsdeveus2mvp26`. No usar guiones ni el correo de la cuenta.

Comprobar que la región está disponible para la suscripción:

```powershell
az account list-locations `
  --query "[?name=='$location'].{name:name,displayName:displayName}" `
  --output table
```

Obtener la IP pública desde la que se accederá a Storage:

```powershell
$publicIp = (Invoke-RestMethod -Uri "https://api.ipify.org?format=json").ip
$publicIp
```

Si se usa VPN, proxy o CGNAT, comprobar que esa es la IP que Azure verá.
Nunca usar `0.0.0.0/0` como solución rápida.

## 4. Crear el grupo de recursos

Este es el primer recurso. Agrupa únicamente recursos propios y desechables
de `dev`; no agregar después recursos compartidos o productivos.

```powershell
az group create `
  --name $resourceGroup `
  --location $location `
  --tags $tags `
  --output table

az group show `
  --name $resourceGroup `
  --query "{name:name,location:location,tags:tags}" `
  --output json
```

Comprobar las cuatro etiquetas antes de continuar. La etiqueta `owner` ayuda
a identificar responsabilidad y costos, pero no concede permisos.

## 5. Crear Storage y el contenedor privado

### 5.1 Crear la cuenta con configuración de laboratorio

```powershell
az storage account create `
  --name $storageAccount `
  --resource-group $resourceGroup `
  --location $location `
  --sku Standard_LRS `
  --kind StorageV2 `
  --access-tier Hot `
  --allow-blob-public-access false `
  --min-tls-version TLS1_2 `
  --https-only true `
  --public-network-access Enabled `
  --default-action Deny `
  --bypass None `
  --tags $tags `
  --output table
```

Esta combinación no hace pública la información. El endpoint existe, pero el
firewall deniega el tráfico salvo a las redes o IP permitidas; el contenedor
también bloqueará acceso anónimo. `LRS` y `Hot` son decisiones de bajo costo
para datos reproducibles de laboratorio, no una configuración de producción.

Añadir la IP actual al firewall:

```powershell
az storage account network-rule add `
  --resource-group $resourceGroup `
  --account-name $storageAccount `
  --ip-address $publicIp `
  --output table
```

No activar el bypass `AzureServices` salvo que una operación concreta y
documentada lo requiera. Si se cambia, registrar por qué y volver a evaluar el
alcance del acceso.

### 5.2 Crear el contenedor privado usando Microsoft Entra ID

```powershell
az storage container create `
  --account-name $storageAccount `
  --name $containerName `
  --auth-mode login `
  --public-access off `
  --fail-on-exist false `
  --output table
```

Obtener el resource ID de la cuenta y asignar al usuario actual el rol de
datos únicamente sobre `documents`:

```powershell
$storageId = az storage account show `
  --resource-group $resourceGroup `
  --name $storageAccount `
  --query id --output tsv

$containerScope = "$storageId/blobServices/default/containers/$containerName"

az role assignment create `
  --assignee-object-id $developerObjectId `
  --assignee-principal-type User `
  --role "Storage Blob Data Contributor" `
  --scope $containerScope `
  --output table
```

Este rol permite leer, crear, actualizar y eliminar blobs del contenedor para
probar el laboratorio. No asignarlo a toda la suscripción. `Contributor` sobre
recursos no concede automáticamente permisos de datos de Blob.

La cuenta personal puede seguir mostrando `Owner` en la suscripción porque es
la cuenta de arranque. No se puede “quitar” ese acceso y seguir operando con
la única cuenta sin riesgo de bloqueo. Lo importante aquí es no conceder ese
rol a nuevas identidades y mantener el acceso de datos con el alcance mínimo.

### 5.3 Aplicar retención solo a datos de prueba

La eliminación automática es opcional. Aplicarla únicamente si todos los
blobs son ficticios y se acepta perderlos:

```powershell
$policyPath = Join-Path $env:TEMP "azure-chapter1-storage-policy.json"
@'
{
  "rules": [
    {
      "enabled": true,
      "name": "delete-dev-documents",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "delete": {
              "daysAfterModificationGreaterThan": 30
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["documents/"]
        }
      }
    }
  ]
}
'@ | Set-Content -Path $policyPath -Encoding utf8

az storage account management-policy create `
  --account-name $storageAccount `
  --resource-group $resourceGroup `
  --policy "@$policyPath" `
  --output table
```

Revisar el archivo antes de ejecutar el comando. El archivo temporal contiene
solo configuración y puede eliminarse después de comprobar la política:

```powershell
Remove-Item -Path $policyPath
```

No aplicar esta política a documentos reales, personales o sujetos a
retención legal.

## 6. Crear la red y el NSG

### 6.1 Crear la VNet y las subnets

```powershell
az network vnet create `
  --resource-group $resourceGroup `
  --name $vnetName `
  --location $location `
  --address-prefixes 10.20.0.0/16 `
  --subnet-name $apiSubnet `
  --subnet-prefixes 10.20.1.0/24 `
  --tags $tags `
  --output table

az network vnet subnet create `
  --resource-group $resourceGroup `
  --vnet-name $vnetName `
  --name $privateEndpointSubnet `
  --address-prefixes 10.20.2.0/24 `
  --output table
```

La segunda subnet reserva espacio para un Private Endpoint futuro. Reservar
espacio no crea el endpoint ni su costo. No asociar todavía Storage a una
VNet ni crear gateway, NAT Gateway o firewall.

### 6.2 Crear y asociar el NSG

```powershell
az network nsg create `
  --resource-group $resourceGroup `
  --name $nsgName `
  --location $location `
  --tags $tags `
  --output table

az network nsg rule create `
  --resource-group $resourceGroup `
  --nsg-name $nsgName `
  --name Deny-SSH `
  --priority 200 `
  --direction Inbound `
  --access Deny `
  --protocol Tcp `
  --source-address-prefixes Internet `
  --source-port-ranges '*' `
  --destination-address-prefixes '*' `
  --destination-port-ranges 22 `
  --output table

az network nsg rule create `
  --resource-group $resourceGroup `
  --nsg-name $nsgName `
  --name Deny-RDP `
  --priority 210 `
  --direction Inbound `
  --access Deny `
  --protocol Tcp `
  --source-address-prefixes Internet `
  --source-port-ranges '*' `
  --destination-address-prefixes '*' `
  --destination-port-ranges 3389 `
  --output table

az network vnet subnet update `
  --resource-group $resourceGroup `
  --vnet-name $vnetName `
  --name $apiSubnet `
  --network-security-group $nsgName `
  --output table
```

No crear una regla de entrada `443`: aún no hay API ni punto de entrada que
proteger. Azure mantiene sus reglas predeterminadas, incluida la denegación
de entrada no permitida. No añadir una denegación total de salida sin conocer
las dependencias de la API futura.

## 7. Configurar presupuesto y alertas

El presupuesto es una alarma, no un apagado automático. Configurarlo después
de crear el grupo, porque el alcance será ese grupo.

En el portal:

1. Abrir **Cost Management > Budgets > Add**.
2. Seleccionar como alcance `rg-docs-dev-eus2`.
3. Elegir periodicidad mensual y un importe que quede por debajo del crédito
   disponible. El valor `20` es solo un ejemplo; confirmar la moneda y el
   límite de la oferta.
4. Usar como fecha inicial el primer día del mes actual y como fecha final la
   fecha real de vencimiento de la oferta, si el portal permite ese límite.
5. Crear alertas reales en 50 % y 80 %, y pronosticada en 100 %.
6. Introducir un correo que la cuenta pueda revisar y guardar.

Después abrir **Cost Management > Cost analysis**, filtrar por el grupo y
confirmar que el presupuesto aparece. No asumir que una alerta impide
consumo posterior.

## 8. Validar en el mismo orden del diseño

### 8.1 Inventario, ubicación y etiquetas

```powershell
az resource list `
  --resource-group $resourceGroup `
  --query "[].{name:name,type:type,location:location}" `
  --output table

az resource list `
  --resource-group $resourceGroup `
  --query "[].{name:name,tags:tags}" `
  --output json
```

Debe existir el grupo y solo los recursos de este capítulo: Storage, VNet,
dos subnets, NSG y sus configuraciones. No deben aparecer API, VM, base de
datos, Private Endpoint, gateway, NAT Gateway o servicios no solicitados.

### 8.2 Storage, firewall y acceso de datos

```powershell
az storage account show `
  --resource-group $resourceGroup `
  --name $storageAccount `
  --query "{sku:sku.name,kind:kind,tier:accessTier,httpsOnly:enableHttpsTrafficOnly,publicBlobAccess:allowBlobPublicAccess,defaultAction:networkRuleSet.defaultAction,bypass:networkRuleSet.bypass}" `
  --output table

az storage account network-rule list `
  --resource-group $resourceGroup `
  --account-name $storageAccount `
  --output json

az storage container list `
  --account-name $storageAccount `
  --auth-mode login `
  --output table

az role assignment list `
  --scope $containerScope `
  --assignee $developerObjectId `
  --all `
  --query "[].{role:roleDefinitionName,scope:scope,principal:principalName}" `
  --output table
```

Confirmar `Standard_LRS`, `StorageV2`, `Hot`, HTTPS obligatorio, acceso
anónimo deshabilitado, firewall con la IP esperada, un único contenedor
`documents` y `Storage Blob Data Contributor` en el alcance del contenedor.

Probar una carga y borrado con un archivo ficticio:

```powershell
$testFile = Join-Path $env:TEMP "azure-chapter1-test.txt"
Set-Content -Path $testFile -Value "chapter-1-test"

az storage blob upload `
  --account-name $storageAccount `
  --container-name $containerName `
  --name chapter-1-test.txt `
  --file $testFile `
  --auth-mode login `
  --overwrite false `
  --output table

az storage blob delete `
  --account-name $storageAccount `
  --container-name $containerName `
  --name chapter-1-test.txt `
  --auth-mode login `
  --delete-snapshots include `
  --output table

Remove-Item -Path $testFile
```

Si falla por red, revisar IP pública, VPN y firewall. Si falla por
`AuthorizationPermissionMismatch`, revisar el rol y esperar la propagación
de RBAC. No habilitar acceso anónimo ni usar claves como solución.

### 8.3 VNet y NSG

```powershell
az network vnet show `
  --resource-group $resourceGroup `
  --name $vnetName `
  --query "{addressSpace:addressSpace.addressPrefixes,subnets:subnets[].{name:name,prefix:addressPrefix,nsg:networkSecurityGroup.id}}" `
  --output json

az network nsg rule list `
  --resource-group $resourceGroup `
  --nsg-name $nsgName `
  --query "[].{name:name,priority:priority,direction:direction,access:access,protocol:protocol,ports:destinationPortRange}" `
  --output table
```

Confirmar VNet `10.20.0.0/16`, `snet-api` `10.20.1.0/24`,
`snet-private-endpoints` `10.20.2.0/24`, NSG asociado a `snet-api`, y reglas
explícitas de denegación para TCP 22 y 3389.

### 8.4 Límites de identidad y RBAC

```powershell
az role assignment list `
  --scope "/subscriptions/$subscriptionId/resourceGroups/$resourceGroup" `
  --include-inherited `
  --all `
  --query "[].{principal:principalName,role:roleDefinitionName,scope:scope}" `
  --output table
```

Es normal que la cuenta personal aparezca como `Owner` heredado en la
suscripción: es la única identidad de arranque y no se ha creado una
identidad alternativa. La comprobación importante es que:

- no se creó ningún usuario, grupo, tenant o suscripción adicional;
- no se asignó `Owner` a una identidad nueva;
- el único acceso de datos creado por esta guía apunta a `documents`;
- no existe una identidad administrada de API antes de que exista la API.

Cuando el proyecto incorpore más personas, crear un grupo de Entra y asignar
roles al grupo, no a cuentas individuales. Cuando exista una API, crear su
identidad administrada y asignarle solo el rol de datos que necesite; nunca
darle `Contributor`, `Owner` ni permisos de administración.

## 9. Actualizar una IP o etiquetas

Si cambia la IP pública, añadir primero la nueva, probar el acceso y retirar
después la antigua desde **Storage Account > Networking > Firewalls and
virtual networks**:

```powershell
$newPublicIp = (Invoke-RestMethod -Uri "https://api.ipify.org?format=json").ip

az storage account network-rule add `
  --resource-group $resourceGroup `
  --account-name $storageAccount `
  --ip-address $newPublicIp `
  --output table
```

Mantener IP antiguas amplía innecesariamente el acceso.

Para corregir etiquetas del grupo:

```powershell
az group update `
  --name $resourceGroup `
  --set tags.environment=dev `
        tags.application=document-api `
        tags.owner=equipo-backend `
        tags.costCenter=CC-0000 `
  --output table
```

Actualizar también las etiquetas de cada recurso si se modificaron
individualmente. Las etiquetas imputan costos, pero no sustituyen RBAC.

## 10. Finalizar, conservar o eliminar el laboratorio

### 10.1 Eliminación recomendada

Antes de eliminar, comprobar que el grupo contiene solo recursos de este
capítulo y que no hay documentos que conservar. Si se deben conservar,
descargarlos a un destino aprobado antes de continuar.

```powershell
az resource list --resource-group $resourceGroup --output table
az group delete `
  --name $resourceGroup `
  --yes `
  --no-wait
```

Comprobar hasta que termine:

```powershell
az group show --name $resourceGroup --output table
```

Un resultado de grupo inexistente confirma la eliminación. `--no-wait` solo
inicia la operación; no significa que los recursos ya hayan desaparecido.

### 10.2 Conservar Storage

Si hay un motivo válido para conservar los blobs, copiar primero los datos
necesarios y eliminar solo VNet y NSG:

```powershell
az network vnet delete --resource-group $resourceGroup --name $vnetName
az network nsg delete --resource-group $resourceGroup --name $nsgName
```

Storage seguirá generando posibles cargos por almacenamiento, operaciones y
transferencia. Cuando ya no se necesite:

```powershell
az storage account delete `
  --resource-group $resourceGroup `
  --name $storageAccount `
  --yes
```

No eliminar un grupo sin revisar antes que no contenga recursos compartidos.

## 11. Criterio de terminado

El capítulo está implementado cuando:

- se verificaron la única suscripción, el tenant existente y la región;
- el grupo y todos sus recursos tienen ubicación y etiquetas correctas;
- Storage usa `StorageV2`, `Standard_LRS`, `Hot`, TLS 1.2 y HTTPS;
- el acceso anónimo está deshabilitado y el firewall no está abierto a todo
  Internet;
- `documents` es privado y la carga de prueba funcionó con Microsoft Entra ID;
- la VNet y sus dos subnets tienen los prefijos aprobados;
- `snet-api` tiene asociado `nsg-api`;
- TCP 22 y 3389 están denegados explícitamente;
- existe presupuesto y alerta en el alcance del grupo;
- no se crearon usuarios, grupos, tenants, suscripciones, API, identidad
  administrada ni servicios fuera del alcance;
- existe una evidencia de inventario y un procedimiento probado de eliminación.

