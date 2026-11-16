# Glosario del capítulo 1

Este glosario reúne los conceptos utilizados en la base teórica, la actividad,
el diseño y la implementación del entorno `dev` del capítulo 1. Incluye
únicamente organización de Azure, administración, identidad, red,
almacenamiento y control básico de costos. Los conceptos marcados como
**futuros** aparecen en el diseño, pero no se implementan en este capítulo.

## 1. Organización y administración de Azure

### Azure

Plataforma de servicios de computación en la nube de Microsoft. En este
capítulo proporciona los recursos de almacenamiento y red que forman el
entorno de desarrollo.

### Azure CLI

Herramienta de línea de comandos para iniciar sesión, consultar el contexto
activo, crear recursos, configurarlos y validarlos mediante instrucciones
reproducibles. Antes de ejecutar cambios se debe confirmar la suscripción
seleccionada.

### Azure Cloud Shell

Terminal hospedada por Azure, disponible en Bash o PowerShell desde el portal.
Incluye herramientas de administración preinstaladas. Su sesión y su
almacenamiento de trabajo son distintos de los recursos creados en Azure.

### Azure Portal

Interfaz web para consultar, crear y configurar recursos, revisar permisos y
analizar costos. Es útil para inspecciones y tareas puntuales; la CLI facilita
repetir y auditar los pasos.

### Recurso

Instancia administrable de un servicio de Azure, como una cuenta de
almacenamiento, una red virtual o un grupo de seguridad de red. Cada recurso
pertenece a un único grupo de recursos.

### Tenant (inquilino)

Instancia de Microsoft Entra ID que representa un directorio de identidad.
Contiene usuarios, grupos, aplicaciones y políticas. Un tenant puede estar
relacionado con una o más suscripciones. En este ejercicio se utiliza el
tenant existente; no se crea otro.

### Suscripción (subscription)

Límite de facturación, cuotas y gobierno para recursos de Azure. También es un
ámbito posible para las asignaciones RBAC. Este ejercicio usa una única
suscripción personal y no crea suscripciones adicionales.

### Grupo de recursos (resource group)

Contenedor lógico de recursos que comparten un ciclo de vida. En el capítulo
se utiliza `rg-docs-dev-eus2` para agrupar los recursos desechables del entorno
`dev`. No es una red ni una frontera de seguridad por sí mismo.

### Entorno `dev`

Entorno de desarrollo y pruebas. Sus recursos pueden configurarse, probarse y
eliminarse sin afectar un entorno productivo. El grupo de recursos y las
etiquetas identifican este propósito.

### Ciclo de vida del recurso

Conjunto de etapas de un recurso: planificación, creación, configuración,
operación, modificación y eliminación. Agrupar recursos por ciclo de vida
permite eliminarlos juntos y reduce el riesgo de afectar recursos compartidos.

### Región

Ubicación geográfica de Azure compuesta por uno o más centros de datos. Afecta
latencia, disponibilidad del servicio, residencia de datos, cuotas y precio.
La región propuesta en este capítulo es **East US 2** (`eastus2`), sujeta a
verificación antes de crear recursos.

### Zona de disponibilidad

Ubicación físicamente separada dentro de una región, con infraestructura
independiente de energía, refrigeración y red. Puede mejorar la resiliencia
ante ciertos fallos, pero requiere que el servicio lo soporte y puede aumentar
complejidad o costo. No es sinónimo de región.

### Cuota

Límite de capacidad o cantidad que Azure aplica a una suscripción, región o
servicio. Debe comprobarse antes de crear recursos; disponer de crédito no
garantiza que exista cuota suficiente.

### Latencia

Tiempo que tarda una solicitud en viajar entre un usuario, una aplicación y un
servicio. La distancia a la región suele influir en ella, pero debe medirse
desde los usuarios reales.

### Residencia o soberanía de datos

Requisito que determina en qué ubicación geográfica pueden almacenarse o
procesarse los datos. Elegir `East US 2` exige confirmar que los documentos de
prueba pueden almacenarse en Estados Unidos.

### Etiqueta (tag)

Par clave-valor asociado a un recurso o grupo de recursos. El capítulo usa
`environment`, `application`, `owner` y `costCenter` para identificar
entorno, aplicación, responsable y centro de costo. Una etiqueta no concede
permisos ni reemplaza RBAC.

### Estándar de nombres

Convención legible para nombrar recursos y distinguir aplicación, entorno y
región, por ejemplo `rg-docs-dev-eus2`. Cada servicio puede imponer
restricciones diferentes de longitud o caracteres; Storage, por ejemplo, solo
admite nombres globalmente únicos con minúsculas y números.

## 2. Identidad, autenticación y autorización

### Microsoft Entra ID

Servicio de identidad y acceso de Azure. Administra usuarios, grupos,
aplicaciones y otros principales de seguridad. En este laboratorio se usa la
cuenta personal existente como identidad de arranque.

### Identidad

Entidad que puede autenticarse en Azure o actuar sobre un recurso. Puede ser
una persona, un grupo, una aplicación o una identidad administrada.

### Principal de seguridad

Identidad a la que se pueden asignar permisos. El usuario personal es el
principal que opera este laboratorio; el grupo de backend y la identidad
administrada aparecen como opciones para escenarios futuros.

### Autenticación

Proceso de comprobar quién es una identidad. Responde a la pregunta
“¿quién eres?”. Iniciar sesión con `az login` autentica la cuenta usada por la
CLI.

### Autorización

Proceso de determinar qué operaciones puede realizar una identidad después de
autenticarse. Responde a la pregunta “¿qué puedes hacer?”.

### Azure RBAC

Control de acceso basado en roles para autorizar operaciones sobre recursos de
Azure. Una asignación RBAC combina un principal, un rol y un ámbito.

### Rol

Conjunto predefinido de permisos. `Contributor` permite administrar recursos
sin administrar el acceso; `Storage Blob Data Contributor` permite leer,
crear, actualizar y eliminar blobs dentro del alcance asignado.

### Asignación de rol

Vínculo entre un principal, un rol y un ámbito. En este capítulo, el usuario
actual recibe `Storage Blob Data Contributor` únicamente sobre el contenedor
`documents`.

### Ámbito (scope)

Nivel al que se aplica una asignación o permiso: grupo de administración,
suscripción, grupo de recursos, recurso o subrecurso. Un ámbito más pequeño
reduce el alcance de un permiso.

### Mínimo privilegio

Principio por el que cada identidad recibe solo los permisos necesarios, en el
alcance más pequeño y durante el tiempo necesario. Implica no conceder
`Owner` a nuevas identidades y no dar permisos de administración a una
aplicación que solo necesita leer blobs.

### Plano de administración (management plane)

Operaciones que modifican o consultan la configuración de recursos de Azure,
como crear una cuenta Storage, cambiar una regla de red o eliminar un grupo de
recursos.

### Plano de datos (data plane)

Operaciones sobre los datos contenidos en un servicio, como cargar, leer o
eliminar un blob. Tener permisos para administrar una Storage Account no
implica automáticamente que una aplicación deba acceder a todos sus datos.

### `Owner`

Rol administrativo amplio que permite administrar recursos y asignar acceso.
La cuenta personal puede tenerlo de forma preexistente en la única
suscripción. En este ejercicio no se concede a ninguna identidad nueva.

### `Contributor`

Rol que permite administrar recursos dentro de su ámbito, pero no asignar
roles de acceso. El diseño lo contempla limitado al grupo de recursos para un
equipo de desarrollo; la implementación con una sola cuenta no necesita crear
una asignación adicional redundante si la cuenta ya es `Owner`.

### Identidad administrada (managed identity) — futuro

Identidad de Microsoft Entra administrada por Azure para que un recurso, como
una API desplegada, acceda a otro servicio sin guardar claves en el código.
`mi-api-docs-dev` se reserva para un capítulo posterior y no se crea en este.

## 3. Redes

### Azure Virtual Network (VNet)

Red privada lógica de Azure que proporciona un espacio de direcciones IP y
permite organizar la conectividad entre recursos. El capítulo propone
`vnet-docs-dev-eus2` con el espacio `10.20.0.0/16`.

### Espacio de direcciones (address space)

Rango de direcciones IP que se reserva para una VNet. En este capítulo es
`10.20.0.0/16` y se divide en subnets con funciones distintas.

### CIDR

Notación para expresar un bloque de direcciones IP y su tamaño, como
`10.20.1.0/24`. El sufijo indica cuántos bits identifican la red.

### Subnet

Subdivisión del espacio de direcciones de una VNet. El diseño usa
`snet-api` (`10.20.1.0/24`) para la API futura y
`snet-private-endpoints` (`10.20.2.0/24`) para conectividad privada futura.
Una subnet organiza direccionamiento y reglas, pero no garantiza por sí sola
que un recurso sea seguro.

### Network Security Group (NSG)

Conjunto de reglas que permite o deniega tráfico según origen, destino,
protocolo y puerto. En este capítulo `nsg-api` se asocia a `snet-api` y
deniega explícitamente TCP 22 (SSH) y TCP 3389 (RDP).

### Regla NSG

Regla de entrada o salida de un NSG. Tiene prioridad, origen, destino,
protocolo, puerto y acción (`Allow` o `Deny`). La regla coincidente con mayor
prioridad determina el resultado.

### Tráfico de entrada (inbound)

Tráfico que llega a un recurso o subnet. La guía no permite SSH ni RDP desde
Internet y no abre todavía HTTPS porque no existe una API desplegada.

### Tráfico de salida (outbound)

Tráfico que sale de un recurso o subnet hacia otro destino. No se bloquea por
completo en este capítulo porque todavía no se conocen las dependencias de la
API futura.

### Firewall de Storage

Reglas de red de una Storage Account que restringen desde qué IPs o redes se
puede alcanzar su endpoint. La implementación permite la IP pública actual
del laboratorio y no abre el acceso a todo Internet.

### Acceso público de blobs

Posibilidad de leer blobs o contenedores sin autenticación. Se deshabilita en
la cuenta y el contenedor `documents`; que el endpoint sea técnicamente
accesible no significa que los blobs sean anónimos.

### Private Endpoint — futuro

Interfaz de red privada que permite acceder a un servicio de Azure mediante
una dirección privada dentro de una VNet. En este capítulo solo se reserva
`snet-private-endpoints`; no se crea ningún Private Endpoint.

## 4. Almacenamiento

### Storage Account

Recurso que proporciona servicios de almacenamiento bajo una configuración
común de rendimiento, redundancia, seguridad y acceso. En el laboratorio se
crea una cuenta con `StorageV2`, nivel `Hot` y redundancia `Standard_LRS`.

### StorageV2

Tipo de cuenta de almacenamiento de propósito general de Azure. Permite usar
Blob Storage y configurar opciones comunes de acceso, rendimiento y
redundancia.

### Blob Storage

Servicio de almacenamiento de objetos para datos no estructurados, como
documentos, imágenes y archivos. No sustituye una base de datos para consultas
relacionales, relaciones o transacciones de aplicación.

### Blob

Objeto que contiene datos dentro de un contenedor. Su nombre puede usar `/`
para representar una ruta lógica, pero esto no implica necesariamente carpetas
físicas tradicionales.

### Contenedor (container)

Agrupación lógica de blobs dentro de una Storage Account. El contenedor
`documents` del capítulo es privado y contiene documentos de prueba de la API
futura.

### `Standard_LRS`

Opción de redundancia de almacenamiento estándar con copias locales dentro de
un único centro de datos o ubicación primaria. Se usa aquí por su bajo costo y
porque los datos son reproducibles; no es una decisión automática para
producción.

### LRS (locally redundant storage)

Almacenamiento con redundancia local. Protege frente a ciertos fallos locales,
pero no ofrece la misma resiliencia que opciones zonales o georredundantes.

### Nivel de acceso `Hot`

Nivel optimizado para datos que se leen o escriben con frecuencia. Se elige
para los documentos de prueba del entorno `dev`; debe revisarse con el patrón
real de uso.

### TLS 1.2

Versión mínima del protocolo criptográfico configurada para proteger las
comunicaciones con Storage. La cuenta también exige HTTPS y rechaza tráfico
HTTP sin cifrar.

### HTTPS

Protocolo HTTP protegido mediante TLS. Es el transporte requerido para acceder
a Storage y para los flujos seguros previstos de la API futura.

### Retención

Periodo durante el cual se conservan los datos. El diseño propone eliminar
documentos de prueba tras 30 días y temporales tras 7 días, pero una política
automática solo debe aplicarse a datos ficticios y no sujetos a conservación
legal.

### Política de ciclo de vida de Storage

Regla automática que puede mover o eliminar blobs según su antigüedad,
prefijo u otras condiciones. Puede reducir costos, pero una regla de
eliminación mal definida causa pérdida de datos.

### Clave de cuenta, connection string y SAS

Credenciales o tokens alternativos para acceder a Storage. Son secretos: no
deben guardarse en el repositorio ni pegarse en código. La guía prioriza
Microsoft Entra ID y Azure RBAC para el acceso del laboratorio.

## 5. Costos, gobierno y validación

### Crédito o capa gratuita

Beneficio económico asociado a la oferta de Azure. No convierte
automáticamente todos los servicios, operaciones o transferencias en
gratuitos; se deben revisar saldo, vencimiento, precios y límites.

### Presupuesto (budget)

Límite de referencia en Cost Management que genera alertas cuando el gasto
real o pronosticado alcanza determinados porcentajes. No detiene recursos
automáticamente ni sustituye la eliminación del grupo de laboratorio.

### Cost Management

Conjunto de herramientas de Azure para consultar costos, analizar consumo y
crear presupuestos. En este capítulo se utiliza para vigilar el grupo
`rg-docs-dev-eus2`.

### Validación

Comprobación de que la configuración creada coincide con el diseño: recursos,
ubicación, etiquetas, Storage, firewall, contenedor, red, NSG, RBAC y
presupuesto. La validación debe incluir una carga y eliminación de un blob
ficticio.

### Eliminación del grupo de recursos

Operación que elimina el grupo y los recursos que contiene. Es la forma
recomendada de detener el laboratorio cuando no se necesitan sus datos, pero
requiere revisar primero el inventario y conservar cualquier documento
necesario.

