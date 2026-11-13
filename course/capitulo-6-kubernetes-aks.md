# Capítulo 6. Kubernetes y Azure Kubernetes Service

## Propósito de lectura

Al finalizar esta lectura podrás explicar la arquitectura básica de Kubernetes,
describir la función de Pods, Deployments, Services y Namespaces, separar
configuración de secretos, exponer una aplicación mediante Ingress, configurar
health probes, requests, limits y Horizontal Pod Autoscaler, y relacionar esos
conceptos con la creación y operación básica de un clúster AKS.

El capítulo parte de las imágenes publicadas en Azure Container Registry. La
orquestación añade un modelo declarativo para ejecutar y mantener contenedores;
no convierte automáticamente una aplicación en escalable o segura. Las
decisiones sobre identidad, red, persistencia, mensajes y observabilidad siguen
siendo necesarias.

## 1. Arquitectura y modelo declarativo

**Kubernetes** es una plataforma de orquestación que mantiene aplicaciones
containerizadas conforme a un estado deseado. En vez de describir solo “ejecuta
este proceso”, el operador declara réplicas, imágenes, puertos, configuración,
recursos y reglas. Kubernetes compara ese estado con el estado observado y
coordina acciones para acercarlos.

Un **clúster** es el conjunto de componentes de control y nodos que ejecutan
cargas. El **control plane** recibe cambios, almacena el estado del clúster,
programa Pods y coordina controladores. Un **node** es una máquina de trabajo
que ejecuta Pods. Un **kubelet** en cada nodo comunica el estado de los Pods
con el control plane y ayuda a mantenerlos ejecutándose.

El **scheduler** selecciona un nodo para un Pod según recursos disponibles,
restricciones y afinidades. Un **controller** observa recursos y crea, elimina
o actualiza objetos para alcanzar el estado deseado. El control loop explica
por qué un Pod eliminado puede volver a aparecer: el Deployment que lo
administra sigue declarando que debe existir.

El estado declarativo suele escribirse en manifiestos YAML o generarse mediante
herramientas de Kubernetes. Un manifiesto debe ser versionable, revisable y
separar valores específicos del ambiente. Aplicar un manifiesto no elimina la
responsabilidad de verificar qué permisos, imágenes y conexiones está
introduciendo.

## 2. Pods

Un **Pod** es la unidad mínima que Kubernetes programa. Contiene uno o más
contenedores que comparten red y almacenamiento definido para el Pod. Los
contenedores dentro del mismo Pod pueden comunicarse por `localhost`; no debe
usarse un Pod como sinónimo de una máquina virtual.

Un Pod está pensado para ser reemplazable. Si un nodo falla, el Pod puede
desaparecer y ser recreado en otro nodo por su controlador. El sistema de
archivos local del contenedor o del Pod no es almacenamiento persistente salvo
que se conecte explícitamente un volumen adecuado. Para datos de la aplicación
se prefieren bases de datos, Blob Storage u otros servicios durables.

Aunque un Pod puede contener un contenedor auxiliar, cada contenedor debe tener
una responsabilidad clara. Colocar API y worker sin relación en el mismo Pod
impide escalarlos y actualizarlos de forma independiente. La red del Pod tiene
una dirección efímera; los clientes no deben guardar esa dirección como
endpoint estable.

## 3. Deployments

Un **Deployment** administra un conjunto de Pods para una aplicación sin
estado. Declara la imagen, cantidad deseada de réplicas, selector, plantilla
del Pod y estrategia de actualización. El Deployment crea un **ReplicaSet**,
que mantiene el número de Pods requerido y permite gestionar revisiones.

Una **réplica** es una instancia del Pod. Si se declaran tres réplicas, el
controlador intenta mantener tres Pods listos, sujeto a capacidad y
restricciones. Más réplicas no corrigen un cuello de botella en la base de
datos ni garantizan que el código sea seguro ante concurrencia.

Una actualización **rolling** reemplaza gradualmente Pods antiguos por nuevos.
La disponibilidad durante el cambio depende de readiness probes, número de
réplicas y parámetros de actualización. Un rollback devuelve la plantilla a
una revisión anterior, pero no revierte efectos externos ni migraciones de
datos. Las imágenes deben identificarse por tag controlado o digest para saber
qué versión se ejecuta.

Los Deployments son adecuados para API y workers sin estado. Un worker que
consume mensajes debe tolerar reentregas e idempotencia; aumentar réplicas
puede aumentar concurrencia y duplicados si el diseño no lo contempla.

## 4. Services

Un **Service** ofrece un endpoint estable para un conjunto de Pods que
coinciden con sus selectores. Kubernetes actualiza los endpoints cuando Pods
entran o salen, de modo que el cliente no necesita conocer direcciones
efímeras.

Tipos comunes:

- **ClusterIP:** acceso interno dentro del clúster; es el comportamiento
  predeterminado.
- **NodePort:** expone un puerto en los nodos; requiere revisar seguridad y
  normalmente se usa como parte de otra solución.
- **LoadBalancer:** solicita un balanceador externo según el entorno.

El Service no despliega Pods ni verifica por sí solo que una aplicación sea
funcional. Selecciona Pods mediante etiquetas y puede enviar tráfico a un Pod
que existe pero todavía no está listo si no se configuran correctamente las
probes. Puerto del contenedor, `targetPort` y puerto del Service deben ser
coherentes.

## 5. Namespaces

Un **Namespace** es un espacio lógico para agrupar y aislar objetos dentro de
un clúster. Puede separar `dev`, `test`, `prod` o equipos, según el modelo de
gobierno. No es una frontera de seguridad completa ni reemplaza redes,
identidad, políticas o clústeres separados.

Los nombres de muchos objetos son únicos dentro de un Namespace, no
necesariamente en todo el clúster. Un contexto de `kubectl` puede indicar
Namespace y clúster activos; operar en el contexto equivocado puede cambiar
otro ambiente. Antes de aplicar o eliminar, conviene verificar contexto,
Namespace y manifiesto.

Los **ResourceQuotas** y **LimitRanges** pueden limitar consumo y establecer
valores predeterminados en un Namespace. Ayudan a evitar que una carga agote
recursos compartidos, pero deben acompañarse de requests y limits coherentes.

## 6. ConfigMaps y Secrets

Un **ConfigMap** contiene configuración no sensible, como nombre de un
ambiente, un nivel de logging o un endpoint no secreto. Un **Secret** contiene
valores sensibles, como tokens, contraseñas o certificados. Ambos pueden
inyectarse como variables de entorno o montarse como archivos.

Un Secret de Kubernetes no debe considerarse automáticamente un almacén seguro:
su protección depende de configuración del clúster, cifrado, RBAC, acceso al
control plane y política de rotación. Deben limitarse los permisos para leerlo
y preferirse integraciones con un gestor de secretos cuando el requisito lo
justifique. Nunca se deben incluir valores reales en manifiestos versionados.

Separar ConfigMap y Secret facilita cambiar configuración sin reconstruir la
imagen. La aplicación debe validar campos obligatorios y no imprimir secretos.
Una actualización de configuración puede requerir reiniciar Pods para que el
proceso lea el nuevo valor; el comportamiento debe conocerse antes de
promoverla.

## 7. Ingress y exposición de aplicaciones

Un **Ingress** describe reglas HTTP o HTTPS para enrutar tráfico externo hacia
Services. Puede asociar dominios, rutas, TLS y diferentes servicios. Un
**Ingress controller** implementa esas reglas mediante un componente que
recibe tráfico y lo dirige.

Ingress no es un firewall ni una política completa de autenticación. La
exposición debe definir:

- qué host y rutas son públicas;
- qué servicio interno recibe cada ruta;
- cómo se termina TLS y dónde se protegen certificados;
- qué red y controles de acceso aplican;
- qué respuestas y límites se necesitan.

Un worker normalmente no necesita Ingress. Exponerlo aumenta superficie de
ataque y no resuelve su consumo de mensajes. Una API puede exponerse mediante
Ingress mientras sus bases de datos y servicios internos permanecen accesibles
solo por red interna y permisos de identidad.

## 8. Health probes

Las **health probes** son comprobaciones que Kubernetes usa para decidir el
estado de un contenedor:

- **startup probe:** da tiempo a una aplicación lenta para arrancar y evita
  que liveness o readiness la marquen prematuramente;
- **liveness probe:** indica si el proceso está vivo; si falla repetidamente,
  Kubernetes puede reiniciar el contenedor;
- **readiness probe:** indica si el Pod puede recibir tráfico; si falla, el
  Pod se retira temporalmente de los endpoints del Service sin reiniciarlo.

Una probe debe ser barata, determinista y representar la condición correcta.
Un endpoint de liveness no debería fallar solo porque una base de datos externa
está temporalmente lenta, o Kubernetes podría reiniciar todas las réplicas y
empeorar el incidente. La readiness puede comprobar dependencias necesarias
para atender tráfico, pero debe evitar consultas costosas.

Configurar tiempos, periodos y umbrales demasiado agresivos causa reinicios o
retirada de tráfico innecesarios. Configurarlos demasiado permisivos retrasa
la detección. Las probes deben probarse con arranque lento, fallo de
dependencia, saturación y recuperación.

## 9. Requests y limits

Un **resource request** es la cantidad de CPU y memoria que el scheduler usa
para colocar un Pod. Un **resource limit** es el máximo permitido para el
contenedor. Sin requests, Kubernetes puede ubicar cargas de forma
impredecible; sin limits, una aplicación puede competir excesivamente por
recursos.

Un contenedor que supera el límite de memoria puede ser terminado por el
runtime. El CPU puede ser limitado, produciendo latencia. Requests demasiado
altos impiden programar réplicas aunque exista capacidad física; límites
demasiado bajos generan reinicios o degradación.

Los valores deben basarse en mediciones de una carga representativa. Para una
API, se relacionan con concurrencia y latencia; para un worker, con tamaño de
mensaje y duración. Requests y limits no sustituyen el escalamiento ni la
capacidad de las dependencias.

## 10. Horizontal Pod Autoscaler

**Horizontal Pod Autoscaler (HPA)** ajusta el número de réplicas de un
Deployment u otro objetivo según métricas. Puede usar CPU, memoria o métricas
personalizadas, como mensajes pendientes, si el entorno las expone.

El HPA necesita:

- requests bien definidos para métricas relativas;
- límites mínimo y máximo de réplicas;
- una señal que represente la carga;
- tiempo suficiente para que una réplica arranque;
- límites compatibles con base de datos, colas y servicios externos.

Escalar por CPU no siempre representa una cola creciente: un worker puede estar
esperando mensajes con poco CPU mientras el backlog aumenta. Escalar por
longitud o antigüedad de cola puede ser más apropiado, pero requiere una
integración de métricas. El HPA también puede reaccionar con retraso y producir
oscilaciones; deben definirse ventanas de estabilización y observarse el
resultado.

## 11. AKS: arquitectura y creación

**Azure Kubernetes Service (AKS)** es un servicio administrado de Azure para
crear y operar clústeres Kubernetes. Azure gestiona componentes del control
plane según el modelo del servicio, mientras el equipo configura nodos,
workloads, red, identidad, almacenamiento, actualizaciones y políticas.

Un **node pool** agrupa nodos con una configuración común. Puede haber un pool
de sistema para componentes esenciales y pools de usuario para cargas. El
tamaño, cantidad, región, red y versión de Kubernetes afectan disponibilidad,
costo y capacidad.

Crear un clúster exige decidir:

- versión y ciclo de actualización;
- identidad del clúster;
- red y rangos de direcciones;
- pools, tamaños y autoscaling;
- integración con ACR;
- acceso administrativo y RBAC;
- almacenamiento y exposición;
- límites de costo y operación.

La creación no termina cuando el clúster aparece disponible. Hay que validar
contexto, credenciales, nodos listos, namespaces, permisos, conectividad,
ingreso y capacidad de extraer imágenes. El acceso de desarrollo debe ser
acotado y auditable; no se debe compartir un kubeconfig administrativo sin
necesidad.

## 12. Publicar imágenes de ACR hacia AKS

Para que AKS ejecute una imagen, los nodos o la identidad configurada deben
tener permiso de extracción (`pull`) sobre ACR. El flujo conceptual es:

1. construir y probar la imagen;
2. publicarla en un repositorio de ACR con tag y digest identificables;
3. habilitar la relación de confianza o credencial de extracción necesaria;
4. referenciar el registro y la imagen en el Deployment;
5. aplicar el manifiesto en el Namespace correcto;
6. comprobar que los Pods descargan la imagen y quedan listos;
7. exponer mediante Service o Ingress según corresponda.

Un error `ImagePullBackOff` indica que el nodo no pudo obtener la imagen; puede
deberse a nombre, tag, digest, red o permisos. Reintentar el despliegue no
corrige una autorización faltante. Las imágenes de producción deben fijarse a
una versión revisable y no depender de `latest`.

## 13. Método de aplicación

Para desplegar una API básica en AKS, usa esta secuencia:

1. **Define el workload:** imagen, puerto, réplicas, estado y dependencias.
2. **Selecciona Namespace:** verifica contexto, cuotas y permisos.
3. **Escribe configuración:** ConfigMap, Secret o integración externa sin
   incluir valores sensibles en Git.
4. **Crea Deployment:** selector, imagen, estrategia, requests, limits y
   probes.
5. **Crea Service:** etiquetas, puerto interno y destino correcto.
6. **Expón solo lo necesario:** Ingress, TLS y rutas para la API; ningún
   ingreso público para el worker.
7. **Integra ACR:** permiso de pull, tag o digest y prueba de extracción.
8. **Ajusta HPA:** métrica, mínimos, máximos y límites de dependencias.
9. **Valida y opera:** Pods, endpoints, logs, salud, actualizaciones,
   rollback, costo y capacidad.

## Cierre

Kubernetes mantiene un estado deseado mediante Pods, Deployments, Services y
controladores. Namespaces organizan objetos; ConfigMaps y Secrets separan
configuración; Ingress expone rutas; probes, requests, limits y HPA controlan
salud y capacidad. AKS administra el control plane de Azure, pero el equipo
continúa diseñando workloads, nodos, permisos, red e imágenes. El siguiente
capítulo profundizará en identidad, secretos, redes privadas y administración
de APIs.
