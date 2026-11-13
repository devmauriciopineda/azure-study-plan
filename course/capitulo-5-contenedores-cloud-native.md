# Capítulo 5. Contenedores y aplicaciones cloud-native

## Propósito de lectura

Al finalizar esta lectura podrás empaquetar una API o un worker Python en una
imagen Docker, publicarla en Azure Container Registry y desplegarla en Azure
Container Apps. También podrás razonar sobre configuración, escalamiento,
revisiones, versionado de imágenes y la elección entre App Service, Functions y
Container Apps.

El capítulo parte del backend y del procesamiento asíncrono de los capítulos
anteriores. Un contenedor cambia la forma de empaquetar y ejecutar la
aplicación, pero no reemplaza el diseño de identidad, persistencia, red,
mensajería ni observabilidad.

## 1. Docker e imágenes

**Docker** es una tecnología para construir y ejecutar aplicaciones dentro de
contenedores. Un **contenedor** es un proceso aislado que comparte el kernel
del sistema anfitrión, mientras una máquina virtual incluye un sistema
operativo invitado completo. El contenedor ofrece aislamiento y portabilidad,
pero no es una frontera de seguridad absoluta.

Una **imagen** es un paquete inmutable compuesto por capas, metadatos y un
comando de inicio. Una imagen sirve como plantilla; un contenedor es una
instancia ejecutándose a partir de esa imagen. La imagen debe incluir el código,
dependencias y archivos necesarios para arrancar, pero no secretos ni datos
persistentes.

El **Dockerfile** describe cómo construir la imagen mediante instrucciones
como:

- seleccionar una imagen base;
- establecer el directorio de trabajo;
- copiar archivos necesarios;
- instalar dependencias;
- declarar el puerto esperado;
- definir el comando de inicio.

Un Dockerfile bien diseñado usa una imagen base mantenida y específica, reduce
archivos innecesarios, fija versiones según la estrategia del proyecto y
ejecuta el proceso con un usuario sin privilegios cuando sea posible. Las
imágenes pequeñas reducen tiempo de transferencia y superficie de ataque, pero
no se debe eliminar una dependencia requerida solo para ahorrar espacio.

El **contexto de construcción** es el conjunto de archivos disponible para
Docker durante el build. Un archivo `.dockerignore` evita enviar secretos,
entornos virtuales, caches, repositorios y artefactos locales. La imagen
resultante debe ser reproducible a partir del código y de las dependencias
declaradas.

## 2. Contenedores, redes y procesos

Un contenedor debe ejecutar un proceso principal en primer plano. Si ese
proceso termina, el contenedor termina. La aplicación no debe asumir que el
sistema de archivos local conservará datos después de reinicio o reemplazo;
para persistencia se usan servicios como bases de datos y Blob Storage.

Una **red de contenedores** permite que procesos containerizados se comuniquen
mediante nombres, puertos y protocolos. El puerto declarado en la imagen no
publica automáticamente la aplicación en Internet; la plataforma de
hospedaje debe enrutar tráfico hacia el puerto que el proceso escucha.

La aplicación debe escuchar en la interfaz y puerto proporcionados por el
entorno. Un servidor ligado solo a `localhost` puede funcionar dentro del
contenedor, pero no recibir tráfico del componente de entrada. Para APIs
HTTP, el proceso debe tener un comando de inicio de producción, manejar señales
de terminación y producir logs hacia la salida estándar.

Una imagen es inmutable; la configuración debe inyectarse en ejecución
mediante variables de entorno, referencias a secretos o mecanismos de la
plataforma. Así, la misma imagen puede promoverse entre `dev`, `test` y
`prod`, cambiando recursos y permisos sin reconstruir el código.

## 3. Azure Container Registry

**Azure Container Registry (ACR)** es un registro administrado para almacenar,
proteger y distribuir imágenes de contenedor. Un **repository** agrupa
imágenes relacionadas y un **tag** es una etiqueta mutable que apunta a una
versión concreta. Un **digest** identifica de forma inmutable el contenido de
una imagen.

Un flujo básico es:

1. construir y probar una imagen localmente o en un proceso automatizado;
2. asignarle un nombre de registro, repositorio y versión;
3. autenticarse en ACR con una identidad o credencial adecuada;
4. publicar la imagen;
5. conceder al entorno de ejecución permiso para extraerla;
6. desplegar usando un tag controlado o, preferiblemente, un digest;
7. conservar una política de retención y limpiar versiones obsoletas.

No se deben usar tags ambiguos como `latest` como único mecanismo de
despliegue, porque pueden cambiar sin que el manifiesto revele qué código está
ejecutándose. Un tag semántico o de commit ayuda a localizar la versión; el
digest garantiza el contenido exacto.

ACR también participa en la seguridad de la cadena de suministro. Se deben
limitar permisos de push y pull, escanear imágenes según las capacidades
disponibles, actualizar imágenes base, eliminar dependencias vulnerables y no
guardar secretos en capas. Borrar un tag no garantiza eliminar todas las
referencias a una imagen, por lo que la limpieza debe entender el
almacenamiento por manifiestos y digests.

## 4. Azure Container Apps

**Azure Container Apps** es un servicio administrado para ejecutar
aplicaciones containerizadas sin gestionar directamente un clúster de
Kubernetes. Permite desplegar APIs, workers y otros procesos, con
configuración de ingreso, revisiones, escalamiento y secretos.

Una **Container App** define una aplicación y sus contenedores, configuración,
variables, secretos, ingreso y reglas de escala. Un **environment** agrupa
aplicaciones relacionadas y proporciona un límite común para red, logs y
configuración de plataforma. El entorno no debe confundirse con `dev` o
`prod` como etiqueta funcional; puede representar el límite de infraestructura
que el equipo decide compartir.

Una **revisión** es una versión desplegable e inmutable de la configuración de
la aplicación. Al cambiar la imagen o propiedades relevantes se crea una nueva
revisión. Las revisiones pueden recibir todo el tráfico, una fracción o ningún
tráfico. Esta capacidad permite validar una versión, hacer pruebas controladas
y volver a una revisión anterior si la nueva falla.

El **ingreso** define cómo una aplicación recibe tráfico externo o interno.
Una API pública requiere reglas de exposición y autenticación apropiadas; un
worker normalmente no necesita ingreso HTTP. Los contenedores del mismo
contexto pueden comunicarse según la configuración de la aplicación y del
entorno, pero cada flujo debe justificarse.

Container Apps simplifica la operación del runtime, pero el equipo sigue
siendo responsable de imagen, configuración, permisos, contratos HTTP,
consumo, salud y comportamiento ante reinicios.

## 5. Despliegue de APIs y workers

Una **API containerizada** expone endpoints HTTP y debe definir puerto,
comando de inicio, configuración, salud y permisos para sus dependencias. Un
**worker containerizado** consume mensajes o ejecuta trabajo en segundo plano;
normalmente no necesita un endpoint público y debe manejar señales,
reentregas, duplicados y finalización ordenada.

Un diseño puede separar API y worker en Container Apps distintas aunque usen la
misma imagen base. La separación permite escalar cada carga de manera
independiente y asignar permisos diferentes. Compartir exactamente el mismo
contenedor no significa compartir identidad, configuración o ciclo de vida.

El flujo de despliegue debe incluir:

1. construir la imagen desde un contexto controlado;
2. ejecutar pruebas de unidad y una prueba de arranque;
3. revisar dependencias y contenido de la imagen;
4. publicar una versión identificable en ACR;
5. configurar ingreso, variables, secretos, identidad y reglas de escala;
6. crear una revisión;
7. probar tráfico y procesamiento de mensajes;
8. promover, dividir tráfico o revertir según evidencia.

La aplicación debe ser **stateless** respecto al disco local. El estado
persistente se guarda en bases de datos, colas y almacenamiento de objetos.
Las variables de entorno no deben contener secretos sin protección; se deben
usar secretos de la plataforma o identidad administrada para acceder a ellos.

## 6. Configuración y escalamiento

La configuración de una Container App incluye imagen, CPU, memoria, réplicas,
ingreso, variables, secretos, identidad y reglas de escala. Un **replica** es
una instancia de la aplicación que procesa carga. El escalamiento horizontal
agrega o elimina réplicas; el vertical cambia recursos asignados a cada
instancia.

La escala debe responder a una señal observable:

- solicitudes HTTP concurrentes o tasa de solicitudes;
- mensajes pendientes en una cola;
- eventos o métricas de consumo;
- un mínimo y máximo de réplicas;
- una programación conocida.

Un mínimo de cero puede reducir costo para cargas intermitentes, pero puede
introducir latencia de arranque. Un mínimo mayor mejora disponibilidad
inmediata, pero consume recursos aunque no haya tráfico. Escalar sin límite
puede aumentar costos o saturar dependencias; los límites deben considerar
base de datos, Storage, Service Bus y cuotas.

Un worker debe coordinar concurrencia con la visibilidad y el tiempo de
procesamiento del mensaje. Aumentar réplicas sin idempotencia puede multiplicar
efectos duplicados. Una API debe probar que más réplicas no dependan de memoria
local, archivos locales o estado global.

## 7. Revisiones y publicación de imágenes

La **publicación** es el proceso de hacer que una revisión reciba tráfico o
empiece a consumir trabajo. Un despliegue **blue-green** mantiene dos versiones
y cambia el tráfico entre ellas. Un despliegue **canary** dirige una fracción
pequeña a la nueva revisión y aumenta gradualmente si las métricas son
aceptables.

Las revisiones ayudan a separar “la imagen está publicada” de “la aplicación
está lista”. Antes de promover, se debe validar:

- inicio correcto y disponibilidad del puerto;
- endpoint de salud o prueba funcional;
- lectura de configuración del ambiente;
- acceso autorizado a dependencias;
- consumo de mensajes sin duplicados;
- errores y latencia dentro de límites.

Si una revisión falla, se puede detener su tráfico y restaurar la anterior,
siempre que el cambio de esquema de datos sea compatible. Un rollback de imagen
no revierte automáticamente migraciones de base de datos ni efectos externos;
la compatibilidad de datos debe formar parte del plan.

## 8. Versionado de imágenes

El **versionado de imágenes** permite identificar qué código y dependencias
ejecuta cada revisión. Un tag basado en el commit, una versión de aplicación o
ambos ayudan a rastrear cambios. El digest es la referencia más precisa para
reproducibilidad.

Una política mínima debe:

1. no reutilizar tags de producción;
2. asociar imagen con commit y fuente;
3. conservar imágenes necesarias para rollback;
4. retirar imágenes no usadas según retención;
5. registrar qué revisión recibió tráfico;
6. actualizar base y dependencias con control.

Construir la misma imagen para todos los ambientes reduce diferencias
accidentales. Lo que cambia entre ambientes debe ser configuración, secreto,
identidad, red y escala, no un código recompilado manualmente.

## 9. Comparar App Service, Functions y Container Apps

| Criterio | App Service | Azure Functions | Container Apps |
|---|---|---|---|
| Unidad principal | Aplicación web o API | Función activada por eventos | Aplicación containerizada |
| Empaquetado | Código o contenedor según configuración | Código y runtime administrado | Imagen Docker |
| Modelo | Proceso HTTP continuo | Invocaciones y triggers | API, worker o proceso containerizado |
| Escalamiento | Plan y reglas de aplicación | Plan y demanda de triggers | Réplicas y reglas de escala |
| Control | Runtime y hospedaje administrados | Mayor abstracción por función | Más control sobre imagen y proceso |
| Adecuado para | APIs web convencionales | Tareas y eventos acotados | APIs y workers con dependencias propias |

Elige **App Service** cuando una aplicación web o API continua encaja con el
runtime administrado y no necesitas empaquetar el entorno completo. Elige
**Functions** cuando el trabajo se expresa naturalmente como invocaciones
activadas por eventos, HTTP o temporizadores. Elige **Container Apps** cuando
necesitas controlar la imagen, ejecutar API y workers, usar dependencias del
contenedor o escalar procesos sin administrar Kubernetes directamente.

No hay una jerarquía universal. Container Apps no reemplaza automáticamente a
App Service o Functions: añade flexibilidad de empaquetado y operación, junto
con responsabilidad por imágenes, puertos, salud y seguridad de la cadena de
suministro.

## 10. Método de aplicación

Para containerizar un componente, usa esta secuencia:

1. **Clasifica el proceso:** API HTTP, worker de mensajes o tarea puntual.
2. **Define el contrato:** puerto, comando, variables, secretos, señales y
   persistencia externa.
3. **Construye la imagen:** Dockerfile mínimo, contexto limpio y usuario sin
   privilegios.
4. **Prueba localmente:** arranque, endpoint, configuración inválida y parada.
5. **Publica en ACR:** tag inmutable, permisos de push/pull y revisión de
   vulnerabilidades.
6. **Despliega una revisión:** ingreso, identidad, secretos, recursos y
   escalamiento.
7. **Valida y promueve:** pruebas funcionales, tráfico o mensajes, logs y
   métricas.
8. **Opera el ciclo:** conservar rollback, retirar imágenes obsoletas y
   revisar costos y límites.

## Cierre

Docker empaqueta una aplicación; ACR conserva y distribuye sus imágenes; Azure
Container Apps ejecuta APIs y workers mediante revisiones y reglas de escala.
La imagen no debe contener estado ni secretos, y una publicación no equivale a
un despliegue saludable. En el siguiente capítulo se estudiará cuándo hace
falta un orquestador más explícito mediante Kubernetes y AKS.
