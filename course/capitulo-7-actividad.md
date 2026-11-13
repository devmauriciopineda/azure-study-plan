# Actividad del Capítulo 7. Protección de una API y sus dependencias

## Propósito

Diseñar la seguridad de la API de documentos desplegada en AKS o Container
Apps, incluyendo autenticación, autorización, secretos, acceso privado y
políticas de API. La actividad no requiere crear recursos reales ni usar
credenciales.

## Caso de aplicación

La solución tiene tres actores:

- una aplicación cliente que solicita acceso a la API;
- una API que consulta metadatos y Blob Storage;
- un worker que procesa documentos y actualiza la persistencia.

La API debe publicarse mediante Azure API Management. El backend y sus
dependencias deben evitar exposición pública directa cuando sea posible. El
worker necesita leer y escribir datos, pero no administrar recursos. El equipo
necesita rotar un certificado TLS y un secreto de integración sin reconstruir
las imágenes.

No se proporciona tenant, dominio, red, roles concretos ni valores secretos.
Declara supuestos y marca las decisiones pendientes.

## Entregable

Crea un documento Markdown en `course/` llamado
`capitulo-7-entrega.md`. Incluye:

1. **Actores y flujos:** diagrama de cliente, Entra ID, APIM, API, worker,
   Key Vault, Storage y base de datos.
2. **Autenticación:** elección entre OAuth 2.0 y OIDC para el cliente y
   descripción del flujo, tokens, audiencia y validaciones.
3. **Autorización:** scopes o app roles para API y worker, roles RBAC, alcance
   y respuestas esperadas ante `401` y `403`.
4. **Managed Identities:** tipo system-assigned o user-assigned para API,
   worker y APIM, con justificación de ciclo de vida y compartición.
5. **Key Vault:** inventario de secretos, claves y certificados, permisos de
   lectura o uso, rotación y comportamiento ante expiración.
6. **API Management:** APIs, productos, subscriptions, policies, validación
   JWT, límites y conexión segura al backend.
7. **Red privada:** VNet, subnets, Private Endpoints, DNS privado, rutas y
   tráfico permitido o bloqueado.
8. **Mínimo privilegio:** tabla de identidades, acciones, recurso, alcance y
   razón de cada permiso.
9. **Pruebas de seguridad:** escenarios permitidos, denegados, rotación,
   secreto ausente, certificado expirado y bypass del backend.

## Pasos

1. Distingue autenticación de autorización para cliente, API y worker.
   Identifica quién emite, quién presenta y quién valida cada token.
2. Elige OIDC para identidad de usuario, OAuth para acceso a API o ambos.
   Especifica scopes, roles, audiencia, emisor, expiración y flujo de tokens.
3. Asigna permisos separados para API y worker. Evita que el worker administre
   la suscripción o que el cliente acceda directamente a Storage.
4. Elige identidades system-assigned o user-assigned. Explica qué ocurre al
   reemplazar el workload y si la identidad debe sobrevivir.
5. Diseña Key Vault con nombres lógicos de secretos, claves y certificados.
   Define quién puede leer, usar, rotar y administrar cada tipo.
6. Propón policies de APIM para validar JWT, limitar tráfico, enrutar al
   backend y ocultar detalles internos. Explica qué autorización conserva el
   backend.
7. Dibuja la VNet y los Private Endpoints. Incluye DNS privado y explica cómo
   evitarías que el backend siga accesible por un endpoint público.
8. Especifica rotación sin reconstruir imágenes y comportamiento durante
   transición, error de lectura o certificado expirado.
9. Redacta pruebas negativas y positivas que verifiquen permisos, tokens,
   red, secretos y políticas, sin usar datos reales.

## Criterios de aceptación

La entrega es aceptable cuando:

- distingue autenticación, autorización, OAuth 2.0 y OpenID Connect;
- define tokens con audiencia, emisor, permisos y validaciones;
- separa permisos de cliente, API, worker, APIM y operadores;
- justifica identidades system-assigned y user-assigned por ciclo de vida;
- usa Key Vault para secretos, claves y certificados sin incluir valores reales;
- describe rotación, expiración y revocación;
- define APIM con APIs, productos, subscriptions y policies coherentes;
- especifica una conexión segura entre APIM y backend;
- diseña VNet, subnets, Private Endpoints y DNS privado;
- aplica mínimo privilegio al plano de administración y al acceso a datos;
- incluye pruebas de `401`, `403`, token inválido, bypass, secreto ausente y
  certificado expirado;
- identifica supuestos, límites y dependencias pendientes;
- no implementa recursos, credenciales ni código de producción.

## Resultado esperado

Obtendrás una arquitectura de seguridad revisable para una API y un worker,
con responsabilidades y permisos explícitos. El resultado debe demostrar que
identidad, autorización, secretos y red se refuerzan entre sí sin depender de
una única barrera.

## Preguntas abiertas para la revisión

- ¿El cliente representa un usuario, una aplicación o ambos?
- ¿Qué permisos necesita el worker para procesar documentos sin administrar
  recursos?
- ¿La identidad user-assigned se comparte realmente o conviene separar
  identidades?
- ¿Qué ocurre si Key Vault no responde durante el arranque?
- ¿Cómo se evita que una API pública omita APIM?
- ¿Qué registros y alertas demostrarían que una policy o Private Endpoint están
  funcionando?

## Fuera de alcance

No registres aplicaciones reales, no crees secretos, certificados o tenants y
no configures APIM, VNets o Private Endpoints. No diseñes todavía pipelines,
dashboards, trazas distribuidas ni resiliencia avanzada; se tratarán en el
Capítulo 8.
