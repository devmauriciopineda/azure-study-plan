# Capítulo 7. Seguridad, identidad y administración de APIs

## Propósito de lectura

Al finalizar esta lectura podrás distinguir autenticación y autorización con
Microsoft Entra ID, explicar los roles de OAuth 2.0 y OpenID Connect, elegir
entre identidades administradas system-assigned y user-assigned, proteger
secretos, claves y certificados con Azure Key Vault, y organizar APIs mediante
Azure API Management. También podrás diseñar acceso privado con Virtual
Network y Private Endpoints aplicando mínimo privilegio.

El capítulo parte de las aplicaciones y workloads desplegados en Azure y AKS.
La seguridad no es un único producto: combina identidad, autorización, red,
secretos, certificados y controles del borde de la API. Una red privada no
corrige permisos excesivos, y un token válido no debe conceder más acceso del
necesario.

## 1. Microsoft Entra ID, autenticación y autorización

**Microsoft Entra ID** es el servicio de identidad y acceso de Azure. Administra
usuarios, grupos, aplicaciones, identidades administradas y políticas de acceso
para recursos y aplicaciones.

Una **identidad** es la representación de una persona, aplicación, servicio o
dispositivo que puede solicitar acceso. Un **principal de seguridad** es la
identidad a la que se asignan permisos. Una **aplicación registrada** describe
cómo una aplicación se integra con Entra ID, qué identificadores usa y qué
permisos puede solicitar.

**Autenticación** verifica la identidad: responde quién presenta una
credencial o token. **Autorización** decide qué puede hacer esa identidad sobre
un recurso: responde qué operación y alcance están permitidos. Confundirlas
produce errores frecuentes: validar que un usuario inició sesión no implica que
pueda leer cualquier documento.

Un **token de acceso** representa autorización para un recurso y audiencia
determinados durante un periodo limitado. La API debe validar firma,
emisor, audiencia, expiración, permisos y, cuando corresponda, tenant. No debe
aceptar cualquier JWT solo porque tenga formato correcto ni confiar en datos
del cliente sin verificar el token.

La autorización puede basarse en roles de aplicación, grupos, scopes u otros
atributos. Debe estar definida en la API y ser consistente con el recurso que
se protege. El backend debe devolver respuestas apropiadas, como `401` cuando
falta autenticación válida y `403` cuando la identidad autenticada no tiene
permiso suficiente, sin revelar información innecesaria.

## 2. OAuth 2.0

**OAuth 2.0** es un marco de autorización delegada. Permite que un cliente
obtenga un token para acceder a un recurso protegido sin recibir directamente
las credenciales del propietario del recurso. Sus roles principales son:

- **resource owner:** quien puede autorizar acceso;
- **client:** aplicación que solicita un token;
- **authorization server:** emite tokens después de autenticar y autorizar;
- **resource server:** API que valida el token y sirve el recurso.

Un **scope** representa un permiso delegado, por ejemplo `documents.read`.
Un **app role** representa un rol de aplicación que puede asignarse a una
identidad, como `DocumentProcessor`. El permiso debe ser específico al recurso
y no convertirse en una autorización global por conveniencia.

El flujo **authorization code** permite que una aplicación obtenga autorización
de un usuario y canjee un código por tokens. **PKCE** añade un verificador para
proteger el intercambio, especialmente en clientes públicos. El flujo
**client credentials** sirve para comunicación de aplicación a aplicación sin
usuario interactivo; la aplicación usa su identidad y permisos propios.

El **refresh token** puede obtener nuevos access tokens en escenarios
permitidos, pero debe protegerse como una credencial. La API normalmente valida
access tokens, no refresh tokens. Los tokens deben tener vida limitada y
audiencia correcta.

OAuth 2.0 por sí solo no define una identidad del usuario ni el formato de
claims. Por eso se complementa con OpenID Connect cuando una aplicación
necesita autenticación de usuario.

## 3. OpenID Connect

**OpenID Connect (OIDC)** es una capa de identidad sobre OAuth 2.0. Permite que
el cliente conozca quién se autenticó mediante un **ID token**, además de usar
access tokens para acceder a APIs.

El ID token está dirigido al cliente que inició sesión; no debe enviarse a una
API como si fuera un access token. El access token está dirigido al resource
server. La aplicación debe validar cada token según su tipo, emisor, audiencia,
firma, expiración y flujo.

Los **claims** son afirmaciones incluidas en un token, como identificador de
usuario, tenant, roles o scopes. Un claim no es automáticamente una verdad
autorizadora: el backend debe verificar que proviene de un emisor confiable,
que corresponde a la audiencia y que la política de acceso lo utiliza de forma
segura.

OIDC resulta apropiado para iniciar sesión y establecer una sesión de
identidad. OAuth 2.0 resulta apropiado para autorizar llamadas a APIs. Una
solución puede usar ambos: el usuario se autentica con OIDC y el cliente
solicita un access token OAuth para la API.

## 4. Managed Identities

Una **Managed Identity** es una identidad administrada por Azure para que un
recurso o workload solicite tokens sin almacenar credenciales en el código.
Después se asignan permisos a esa identidad sobre Key Vault, Storage, bases de
datos u otros recursos.

Una identidad **system-assigned** se crea junto con un recurso y se elimina
cuando ese recurso se elimina. Su ciclo de vida está ligado a una única
aplicación o servicio; es simple cuando la identidad no debe sobrevivir al
recurso.

Una identidad **user-assigned** es un recurso independiente que puede asociarse
con uno o varios recursos. Su ciclo de vida se administra por separado y puede
mantenerse al reemplazar una aplicación. Es útil cuando varios workloads deben
compartir una identidad controlada, aunque aumenta la necesidad de gobernar
qué recursos pueden usarla.

La identidad administrada no concede permisos por defecto. El recurso debe
obtener un token y el principal correspondiente debe tener un rol o permiso
sobre el destino. Debe elegirse el alcance más pequeño: una identidad que solo
lee secretos de un vault no necesita administrar el vault ni la suscripción.

Para escoger tipo:

| Criterio | System-assigned | User-assigned |
|---|---|---|
| Ciclo de vida | Unido al recurso | Independiente |
| Compartición | Normalmente una aplicación | Puede asociarse a varios recursos |
| Reemplazo | Requiere reasignar permisos al recrear | Puede sobrevivir al reemplazo |
| Riesgo principal | Pérdida al eliminar recurso | Compartición excesiva de privilegios |

## 5. Azure Key Vault

**Azure Key Vault** es un servicio para almacenar y controlar **secrets**,
**keys** y **certificates**. Un **secret** es un valor confidencial, como una
contraseña o token. Una **key** es una clave criptográfica que puede usarse
para operaciones de cifrado o firma. Un **certificate** combina información de
identidad y material criptográfico para establecer confianza o TLS.

Key Vault separa el almacenamiento y control del secreto de la aplicación. El
workload solicita el valor con una identidad autorizada, lo utiliza y evita
imprimirlo o persistirlo en disco. La aplicación debe recuperar solo los
secretos necesarios y manejar correctamente ausencia, expiración y rotación.

El acceso debe seguir un modelo de permisos mínimo. Dependiendo de la
configuración, se usan roles de Azure RBAC o políticas de acceso. En ambos
casos deben distinguirse operaciones como leer un secreto, listar secretos,
administrar claves o emitir certificados. Poder administrar el recurso no debe
implicar poder leer todos sus valores.

La **rotación** reemplaza un secreto, clave o certificado antes de que expire o
se comprometa. Una aplicación robusta puede usar versiones, renovación
coordinada y recarga controlada. Rotar un secreto no corrige una copia ya
expuesta; los valores comprometidos deben revocarse y reemplazarse.

Key Vault no elimina la necesidad de proteger logs, variables de entorno,
volcados, pipelines y estaciones de desarrollo. Una referencia a un secreto
también puede revelar nombres sensibles o topología, por lo que debe
administrarse con cuidado.

## 6. Azure API Management

**Azure API Management (APIM)** es una plataforma administrada para publicar,
proteger, transformar, observar y gobernar APIs. Separa el punto de entrada
público o de consumidores del backend que implementa la lógica.

Una **API** en APIM representa operaciones, rutas, métodos y contrato. Un
**producto** agrupa una o más APIs y define cómo se ofrecen a consumidores,
incluidas suscripciones, acceso y visibilidad. Una **subscription** de APIM
identifica el consumo y puede exigir una clave, pero no sustituye OAuth,
autorización de negocio ni controles de red.

Una **policy** es una regla declarativa ejecutada en entrada, salida o
backend. Puede validar JWT, limitar tasa, transformar encabezados, reescribir
URLs, controlar caché o enrutar solicitudes. Las policies deben ser pequeñas,
revisables y coherentes con el contrato; ocultar errores o copiar secretos en
encabezados no es una política de seguridad aceptable.

APIM puede validar tokens de Entra ID antes de enviar tráfico al backend,
limitar consumidores y centralizar documentación. El backend debe seguir
validando autorización relevante, porque una solicitud que pasó por APIM puede
llegar por otra ruta interna o contener permisos insuficientes para la
operación.

El backend debe estar protegido para no permitir que un consumidor omita APIM
si la arquitectura requiere políticas centrales. Esto puede implicar red
privada, autenticación entre APIM y backend y reglas de ingreso. APIM no
convierte una API insegura en segura por el solo hecho de publicarla.

## 7. Azure Virtual Network y Private Endpoints

Una **Azure Virtual Network (VNet)** proporciona direccionamiento y
conectividad privada entre recursos y redes conectadas. Las **subnets**
organizan segmentos de la VNet y pueden asociarse a controles de red.

Un **Private Endpoint** es una interfaz de red privada que asigna una dirección
IP de una subnet para acceder a un servicio de Azure mediante Private Link. El
tráfico hacia ese servicio puede permanecer en una red privada en lugar de
usar un endpoint público. La resolución DNS privada debe dirigir el nombre del
servicio a la dirección correcta; tener una IP privada sin DNS coherente causa
fallos de conexión.

Una arquitectura de acceso privado debe definir:

- qué consumidores pueden llegar a la subnet;
- qué servicio se publica mediante Private Endpoint;
- quién administra aprobación y ciclo de vida de la conexión;
- qué zonas DNS privadas se necesitan;
- qué tráfico se permite y qué tráfico se bloquea;
- cómo se mantiene acceso administrativo controlado.

Private Endpoint limita el camino de red, pero no reemplaza autenticación ni
autorización. La aplicación aún necesita una identidad con permisos sobre el
servicio. Del mismo modo, una VNet no impide que una identidad autorizada
realice una operación destructiva.

## 8. Mínimo privilegio para aplicaciones y workloads

El **principio de mínimo privilegio** exige conceder solo los permisos
necesarios, al principal correcto, en el alcance más pequeño y durante el
tiempo requerido. Se aplica a usuarios, aplicaciones, identidades
administradas, pods, pipelines y operadores.

Para aplicarlo:

1. inventaria acciones, recursos y responsables;
2. separa lectura, escritura, administración y uso de datos;
3. asigna permisos al recurso o grupo más pequeño razonable;
4. usa identidades distintas para API, worker y despliegue cuando sus
   responsabilidades difieren;
5. evita credenciales compartidas y roles amplios por comodidad;
6. revisa asignaciones, expiraciones y accesos no utilizados;
7. prueba tanto acceso permitido como acceso denegado.

El acceso debe ser seguro por defecto. Si falta un token, permiso, secreto o
ruta de red, el sistema debe rechazar explícitamente la operación y registrar
un diagnóstico que no revele material sensible.

## 9. Protección de credenciales, certificados y configuración

Una **credencial** prueba identidad o permite obtener un token. Un
**certificado** puede autenticar un servicio o proteger TLS. La configuración
puede incluir endpoints, nombres de recursos y flags que, aunque no sean
secretos, revelan arquitectura o cambian controles.

Reglas prácticas:

- no guardar secretos en Git, Dockerfiles, imágenes, logs, tickets ni ejemplos;
- usar identidades administradas y Key Vault cuando sea posible;
- limitar lectura, escritura, listado y administración por rol;
- rotar credenciales y certificados antes de su expiración;
- validar certificados y hostname, sin desactivar TLS para “resolver” errores;
- separar valores por ambiente;
- auditar accesos y alertar sobre fallos repetidos;
- eliminar o revocar credenciales expuestas, no solo borrar el archivo.

La seguridad debe contemplar todo el trayecto: cliente, APIM, backend,
dependencias, red, almacenamiento y registros. Un token en tránsito, un
secreto en una variable o un certificado en un volumen requieren controles
distintos, pero forman parte de la misma superficie de riesgo.

## 10. Método de diseño

Para proteger una API y sus dependencias, usa esta secuencia:

1. **Identifica actores:** usuario, cliente, API, worker, operador y pipeline.
2. **Define autenticación:** Entra ID, OAuth/OIDC, identidad administrada o
   certificado según el flujo.
3. **Define autorización:** scopes, roles, RBAC y permisos de datos mínimos.
4. **Protege secretos:** Key Vault, rotación, versiones y acceso auditado.
5. **Diseña el borde:** APIM, validación de tokens, productos, policies y
   límites.
6. **Diseña red:** VNet, subnets, Private Endpoints, DNS y rutas permitidas.
7. **Valida fallos:** token inválido, rol insuficiente, secreto ausente,
   certificado expirado y endpoint público inesperado.
8. **Opera el acceso:** revisar permisos, rotar valores, auditar uso y retirar
   identidades o rutas obsoletas.

## Cierre

Entra ID proporciona identidad; OAuth 2.0 y OIDC estructuran autorización y
autenticación; Managed Identities evitan secretos embebidos; Key Vault protege
valores criptográficos; APIM gobierna el acceso a APIs; y VNet con Private
Endpoints limita los caminos de red. La protección real aparece cuando estas
capas se coordinan bajo mínimo privilegio. El siguiente capítulo automatizará
entrega, observabilidad y confiabilidad con GitHub Actions y Azure Monitor.
