# Capítulo 1. Fundamentos de Azure y organización de recursos

## Propósito de lectura

Al finalizar esta lectura podrás explicar cómo se organiza Azure, elegir una
región y un grupo de recursos con criterios razonables, administrar recursos
mediante el portal, la CLI y Cloud Shell, y aplicar controles básicos de
identidad, acceso, red y almacenamiento. El capítulo prepara las decisiones de
arquitectura y operación que se necesitarán en los capítulos posteriores.

## 1. La arquitectura de organización de Azure

Azure es una plataforma de servicios en la nube. Un **recurso** es una
instancia administrable de un servicio, como una cuenta de almacenamiento, una
red virtual o una base de datos. Los recursos se organizan en varios niveles:

- Un **tenant** (inquilino) de Microsoft Entra ID es una instancia dedicada de
  identidad que representa a una organización. Contiene usuarios, grupos,
  aplicaciones y políticas de identidad. Un tenant puede asociarse con una o
  más suscripciones.
- Una **subscription** (suscripción) es un límite de facturación, cuotas y
  control de acceso para recursos de Azure. Una organización puede usar varias
  suscripciones para separar entornos, equipos, unidades de negocio o
  necesidades de gobierno.
- Un **resource group** (grupo de recursos) es un contenedor lógico para
  recursos que comparten un ciclo de vida. No es una red ni una frontera de
  seguridad por sí misma. Un recurso pertenece a un solo grupo de recursos,
  aunque servicios relacionados puedan residir en grupos distintos.
- Una **región** es una ubicación geográfica de Azure compuesta por uno o más
  centros de datos. La selección de región afecta latencia, disponibilidad de
  servicios, residencia de datos, cumplimiento y precio.

Estas relaciones no son intercambiables: el tenant administra identidades, la
suscripción administra principalmente facturación y alcance de gobierno, y el
grupo de recursos organiza recursos relacionados. Por ejemplo, una API y su
cuenta de almacenamiento pueden estar en el mismo grupo porque se despliegan y
eliminan juntas, mientras que una red compartida puede pertenecer a otro grupo
con un ciclo de vida más largo.

### Regiones y disponibilidad

Elegir la región más cercana a los usuarios normalmente reduce latencia, pero
no basta como criterio. Antes de crear recursos conviene validar:

1. que el servicio requerido esté disponible en la región;
2. que la región cumpla requisitos de residencia o soberanía de datos;
3. que existan opciones de redundancia adecuadas;
4. que el precio y las cuotas sean compatibles con el entorno.

Una **zona de disponibilidad** es una ubicación físicamente separada dentro de
una región, con energía, refrigeración y red independientes. Las zonas pueden
reducir el impacto de una falla localizada, pero requieren que el servicio y la
arquitectura soporten esa distribución. Región y zona no son sinónimos:
distribuir entre zonas protege frente a ciertos fallos regionales parciales;
distribuir entre regiones aborda un ámbito de fallo más amplio y suele añadir
complejidad, latencia y coste.

## 2. Herramientas de administración

**Azure Portal** es una interfaz web para explorar recursos, consultar estado,
configurar propiedades y revisar costos o diagnósticos. Es útil para aprender,
inspeccionar una configuración y ejecutar tareas puntuales. Los cambios
manuales en el portal pueden ser difíciles de repetir y auditar si no se
documentan.

**Azure CLI** es una herramienta de línea de comandos para automatizar tareas
con comandos reproducibles. Su contexto suele incluir la suscripción activa y
el grupo de recursos que se indique en cada operación. Antes de crear recursos,
es importante iniciar sesión y confirmar la suscripción, porque un comando
válido puede modificar el entorno equivocado.

**Azure Cloud Shell** es un entorno de shell hospedado por Azure, accesible
desde el portal. Ofrece una terminal Bash o PowerShell y herramientas de Azure
preinstaladas. Su almacenamiento persistente, cuando se configura, permite
conservar archivos de trabajo; la sesión de shell y los recursos de Azure son
conceptos diferentes.

En equipos y entornos reales, Portal, CLI y Cloud Shell pueden complementarse:
Portal facilita la observación, CLI facilita la repetibilidad y Cloud Shell
reduce la necesidad de preparar una máquina local. Ninguna herramienta reemplaza
la revisión del alcance, permisos y parámetros antes de ejecutar cambios.

## 3. Ciclo de vida y etiquetas

El **ciclo de vida de un recurso** comprende su planificación, creación,
configuración, operación, modificación y eliminación. Un grupo de recursos
puede servir como unidad de ciclo de vida: si una aplicación de prueba y sus
dependencias se eliminan juntas, agruparlas facilita la limpieza. En producción,
un grupo debe proteger recursos cuyo ciclo de vida sea más largo para evitar
borrados accidentales.

Las **etiquetas** son pares clave-valor asociados a recursos y grupos de
recursos. No cambian el comportamiento técnico del servicio, pero permiten
clasificar costos, responsables, entornos y aplicaciones. Un esquema inicial
podría incluir `environment`, `application`, `owner` y `costCenter`. Las
etiquetas no sustituyen el control de acceso: indicar `owner=equipo-api` no
otorga permisos al equipo.

Un esquema de nombres y etiquetas consistente debe ser:

- estable y legible para búsquedas;
- suficientemente específico para diferenciar entornos;
- compatible con restricciones del servicio;
- acordado antes de multiplicar recursos.

No todos los servicios admiten exactamente las mismas etiquetas o nombres. Por
eso, el estándar debe documentar excepciones y no asumir que una etiqueta
aplicada a un grupo se comportará como una herencia universal para cada
recurso.

## 4. Identidad y acceso

**Microsoft Entra ID** es el servicio de identidad y acceso de Azure. Gestiona
identidades de usuarios, grupos, aplicaciones y otros principales de seguridad,
así como autenticación y políticas relacionadas. **Autenticación** responde
“¿quién es la identidad?”; **autorización** responde “¿qué puede hacer?”.

**Azure RBAC** (control de acceso basado en roles) autoriza operaciones sobre
recursos de Azure. Un rol reúne permisos, como leer, crear o administrar
recursos. Una **asignación de rol** vincula un principal de seguridad con un
rol y un ámbito. Los ámbitos pueden ser grupo de administración, suscripción,
grupo de recursos o recurso individual.

El **principio de mínimo privilegio** indica que cada identidad debe recibir
solo los permisos necesarios, en el alcance más pequeño razonable y durante el
tiempo necesario. En la práctica:

1. separar roles de lectura, operación y administración;
2. preferir un grupo o identidad de aplicación en vez de permisos individuales
   difíciles de mantener;
3. asignar permisos al grupo de recursos o recurso cuando no se necesita
   administrar toda la suscripción;
4. revisar y retirar asignaciones obsoletas;
5. evitar usar cuentas personales o credenciales compartidas para automatizar.

RBAC controla acciones sobre el plano de administración de Azure, como cambiar
la configuración de una cuenta. El acceso a los datos dentro de un servicio
puede tener controles adicionales. Por ejemplo, poder administrar una cuenta de
almacenamiento no implica automáticamente que una aplicación deba leer todos
sus blobs. Esta distinción evita confundir “administrar el recurso” con
“consumir sus datos”.

## 5. Redes: VNet, subnets y NSG

Una **Azure Virtual Network (VNet)** es una red privada lógica en Azure.
Proporciona un espacio de direcciones IP y permite conectar recursos entre sí,
con redes locales o con otras redes mediante mecanismos específicos.

Una **subnet** es una subdivisión del espacio de direcciones de una VNet. Puede
usarse para separar componentes por función, como aplicaciones, datos o
entrada. La subnet facilita aplicar reglas y organizar el direccionamiento,
pero por sí sola no convierte un recurso en seguro ni necesariamente privado.

Un **Network Security Group (NSG)** es un conjunto de reglas que permite o
deniega tráfico de red según propiedades como origen, destino, puerto y
protocolo. Sus reglas tienen prioridades: una coincidencia de mayor prioridad
determina el resultado. Un NSG puede asociarse a una subnet o a una interfaz de
red, según el servicio y el diseño.

Un diseño básico debe declarar qué tráfico necesita cada componente y bloquear
el resto. Abrir un puerto “para probar” y dejarlo abierto es una configuración
insegura. También es importante distinguir conectividad de autorización:
permitir tráfico de red no concede permisos de identidad, y un permiso RBAC no
crea una ruta de red.

## 6. Storage Accounts y Blob Storage

Una **Storage Account** es el recurso que proporciona servicios de
almacenamiento de Azure bajo una configuración común de rendimiento,
redundancia, seguridad y acceso. **Blob Storage** es el servicio de
almacenamiento de objetos para datos no estructurados, como documentos,
imágenes, archivos de exportación o artefactos.

En Blob Storage, un **contenedor** agrupa blobs y un **blob** es el objeto que
contiene los datos. El nombre del blob puede representar una ruta lógica, pero
esa ruta no implica necesariamente carpetas físicas tradicionales. Los datos se
pueden proteger con autorización de identidad, claves o tokens temporales,
según el escenario. Las credenciales de una cuenta deben tratarse como secretos
y no colocarse en código fuente.

Al diseñar una cuenta de almacenamiento se consideran:

- redundancia y durabilidad requeridas;
- acceso público o privado;
- rendimiento y patrón de lectura/escritura;
- límites de tamaño y operación del servicio;
- cifrado y control de claves;
- ciclo de vida de los datos y costo.

El almacenamiento de objetos no sustituye automáticamente a una base de datos.
Es adecuado para contenido y archivos; una base de datos aporta consultas,
relaciones, transacciones u otras garantías distintas. La elección depende del
tipo de dato y de cómo la aplicación necesita recuperarlo.

## 7. Método de aplicación

Para organizar un entorno nuevo, usa esta secuencia:

1. **Delimita el entorno:** identifica aplicación, equipo, etapa (`dev`,
   `test` o `prod`) y responsables.
2. **Elige suscripción y región:** verifica cuotas, disponibilidad, latencia,
   residencia de datos y redundancia.
3. **Define ciclo de vida:** decide qué recursos se crean, actualizan y
   eliminan juntos; crea grupos de recursos con ese criterio.
4. **Nombra y etiqueta:** aplica el estándar acordado y registra excepciones.
5. **Diseña acceso:** enumera identidades, acciones y ámbitos; asigna el rol
   mínimo necesario.
6. **Diseña red:** define subnets, flujos requeridos y reglas NSG; deniega
   tráfico innecesario.
7. **Configura almacenamiento:** determina tipo de datos, acceso, redundancia,
   retención y protección de secretos.
8. **Valida y documenta:** confirma suscripción, región, permisos, etiquetas y
   reglas antes de considerar terminado el cambio.

Este procedimiento no reemplaza una revisión de seguridad o de costos, pero
evita que la creación de recursos sea una secuencia aislada de clics.

## Cierre

Azure organiza recursos mediante tenants, suscripciones, grupos y regiones;
RBAC y Microsoft Entra ID controlan quién puede actuar; las VNets, subnets y
NSG delimitan conectividad; y Storage Accounts con Blob Storage resuelven una
parte del almacenamiento de aplicaciones. En el siguiente capítulo estas
decisiones servirán como base para desplegar y configurar aplicaciones Python y
APIs REST con servicios administrados.
