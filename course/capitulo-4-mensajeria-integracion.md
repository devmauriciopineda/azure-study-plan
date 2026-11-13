# Capítulo 4. Mensajería, integración y procesamiento asíncrono

## Propósito de lectura

Al finalizar esta lectura podrás distinguir colas, topics y eventos; elegir
entre Azure Service Bus, Azure Event Grid y Azure Queue Storage; integrar
Azure Functions con servicios de mensajería y diseñar procesamiento asíncrono
con reintentos, idempotencia y manejo de errores.

El capítulo parte de la API y la persistencia del capítulo anterior. Su
objetivo es desacoplar productores y consumidores cuando el trabajo no debe
completarse dentro de la solicitud HTTP. No busca todavía profundizar en
contenedores, Kubernetes o pipelines de entrega.

## 1. Mensajería y desacoplamiento

La **mensajería** permite que un componente envíe información a otro mediante
un servicio intermediario. El productor no necesita invocar directamente al
consumidor en el mismo momento. El **desacoplamiento** reduce la dependencia
temporal y estructural entre componentes: el productor puede continuar aunque
el consumidor esté ocupado o momentáneamente no disponible.

Una **cola** conserva mensajes para que uno o más consumidores los procesen.
Un mensaje suele contener un cuerpo, identificadores, propiedades y metadatos.
El consumidor recibe el mensaje, ejecuta el trabajo y lo confirma cuando
termina. Según el servicio, un mensaje no confirmado puede volver a estar
disponible.

Un **evento** es una notificación de que algo ocurrió, como la creación de un
blob o el cambio de estado de un recurso. Un evento normalmente comunica un
hecho y permite que varios suscriptores reaccionen. No necesariamente contiene
todo lo necesario para completar una transacción.

El procesamiento asíncrono separa la aceptación de una solicitud de la
finalización del trabajo. En lugar de mantener un HTTP request abierto durante
una operación larga, la API puede validar, guardar un mensaje y devolver una
respuesta como `202 Accepted` con un identificador de seguimiento. Un worker
procesa después y actualiza el estado. Esta estrategia mejora disponibilidad y
capacidad de absorción, pero exige estados, reintentos, seguimiento y manejo de
duplicados.

## 2. Azure Service Bus

**Azure Service Bus** es un servicio de mensajería empresarial para integrar
aplicaciones mediante colas y publicación-suscripción. Está orientado a
mensajes que requieren controles como orden relativo, sesiones, transacciones,
programación, duplicación detectada o dead-lettering, según las capacidades
usadas.

Una **queue** (cola) entrega cada mensaje a un consumidor, normalmente con un
modelo de competencia entre receptores. Una **topic** (topic) recibe un mensaje
una vez y lo distribuye a varias **subscriptions** (suscripciones). Cada
suscripción tiene su propio flujo de mensajes y puede aplicar filtros para
recibir solo eventos relevantes.

Una **dead-letter queue (DLQ)** es un subcanal donde se colocan mensajes que no
pueden procesarse, por ejemplo después de demasiados intentos, por expiración o
por una regla explícita. La DLQ no es un basurero: requiere monitoreo,
clasificación, corrección y un procedimiento de reintento o descarte.

Service Bus es apropiado cuando el mensaje representa trabajo de negocio y se
necesitan garantías y controles de entrega más ricos. Ejemplos:

- procesar un pedido una vez lógica aunque haya reentregas;
- distribuir un evento de negocio a facturación y notificaciones;
- retrasar una operación hasta una hora determinada;
- aislar mensajes inválidos en una DLQ.

El productor debe definir contrato, versión, identificador y expiración del
mensaje. El consumidor debe confirmar solo después de completar la operación
requerida. Confirmar antes puede perder trabajo; confirmar después de cada
efecto sin protección puede producir duplicados si el proceso falla entre el
efecto y la confirmación.

## 3. Azure Event Grid

**Azure Event Grid** es un servicio de distribución de eventos para reaccionar
ante cambios ocurridos en Azure o en aplicaciones. Un **publisher** (publicador)
emite un evento. Una **event subscription** (suscripción de eventos) define
qué eventos recibe un endpoint. Un **event handler** procesa la notificación.

Event Grid se adapta a arquitecturas orientadas a eventos donde un hecho debe
desencadenar reacciones, por ejemplo:

- un blob creado inicia validación de contenido;
- un recurso cambia de estado y actualiza un inventario;
- una aplicación publica un evento de dominio para varias reacciones.

Los filtros permiten seleccionar eventos por tipo, sujeto u otras propiedades.
La entrega es orientada a notificación, no una garantía de que el consumidor
pueda mantener una transacción larga dentro del evento. El consumidor debe
validar el esquema, verificar el identificador, responder correctamente al
endpoint y poder procesar reentregas.

Event Grid y Service Bus pueden trabajar juntos. Event Grid notifica que algo
ocurrió y Service Bus conserva trabajo que requiere procesamiento controlado.
No se debe usar Event Grid como sustituto automático de una cola duradera ni
Service Bus como simple canal de cada notificación sin evaluar necesidades.

## 4. Azure Queue Storage

**Azure Queue Storage** es un servicio sencillo de colas basado en Storage
Accounts. Permite almacenar mensajes para trabajo asíncrono, como tareas de
procesamiento de archivos, limpieza o comunicación entre componentes simples.

Su modelo tiene menos capacidades empresariales que Service Bus. Es útil cuando
se necesita una cola de trabajo económica y directa, sin topics, suscripciones,
sesiones o transacciones de mensajería avanzadas. La aplicación sigue teniendo
que controlar formato, visibilidad, reintentos, expiración e idempotencia.

La **visibilidad** es el periodo durante el cual un mensaje recibido no aparece
para otros consumidores mientras el consumidor actual lo procesa. Si el
consumidor no elimina el mensaje antes de que termine ese periodo, puede volver
a estar disponible. El periodo debe superar la duración esperada o renovarse
con cuidado; hacerlo demasiado corto puede provocar procesamiento concurrente.

Queue Storage es una buena opción para una tarea simple y desacoplada vinculada
a almacenamiento. Si se necesitan varias rutas de publicación, filtros,
ordenamiento, transacciones o administración avanzada de mensajes, Service Bus
puede representar mejor el requisito.

## 5. Integración con Azure Functions

Azure Functions puede actuar como consumidor o productor. Un **trigger de
mensajería** inicia una función cuando llega un mensaje o evento. Un **output
binding** puede publicar un mensaje sin que la función gestione manualmente
todo el cliente, aunque el contrato, permisos y errores deben seguir siendo
explícitos.

Un flujo típico es:

1. un productor guarda el estado inicial en la base de datos;
2. publica un mensaje con identificador y datos mínimos;
3. el trigger activa una función;
4. la función valida el esquema y verifica duplicados;
5. procesa el trabajo y actualiza el estado;
6. confirma o elimina el mensaje cuando el efecto requerido terminó;
7. envía a DLQ o registra un error permanente si no puede continuar.

El mensaje debe ser autocontenido en lo necesario para localizar el dato, pero
no debe copiar grandes archivos ni secretos. Un patrón común es enviar
`document_id`, `blob_name`, tipo de operación y versión del contrato; la función
lee el contenido desde Blob Storage usando su identidad.

La función debe ser corta y observable. El timeout del trigger, la concurrencia
del host, la capacidad del servicio y la duración de la operación afectan el
diseño. Si el trabajo no cabe en una ejecución, se debe dividir, extender la
visibilidad o usar una orquestación apropiada en vez de asumir que el runtime
esperará indefinidamente.

## 6. Reintentos y entrega de mensajes

La entrega de mensajes suele ser **al menos una vez**: el sistema intenta que
el mensaje llegue, pero puede entregarlo más de una vez. Un consumidor no debe
interpretar “recibí una vez” como garantía de ejecución única.

Un **reintento** vuelve a ejecutar una operación después de un error que podría
ser transitorio, como un timeout o una interrupción de red. Un error
permanente, como un esquema inválido, no debe reintentarse indefinidamente.

Una política de reintentos debe definir:

- máximo de intentos;
- retraso fijo o backoff exponencial;
- aleatoriedad (*jitter*) para evitar picos sincronizados;
- clases de error reintentables;
- timeout total;
- destino del mensaje cuando se agota el límite;
- métricas y alertas.

Reintentar una operación de escritura puede crear duplicados. Por ejemplo, si
el consumidor guarda un resultado y pierde la respuesta antes de confirmar el
mensaje, procesará de nuevo al siguiente intento. El reintento solo es seguro
si la operación es idempotente o cuenta con una protección equivalente.

## 7. Idempotencia

Una operación es **idempotente** cuando ejecutarla varias veces produce el
mismo estado observable que ejecutarla una vez. “Mismo estado” no significa que
no se realicen llamadas repetidas, sino que sus efectos de negocio no se
duplican.

Para lograrla, el mensaje puede incluir un **idempotency key** o identificador
único de operación. El consumidor registra ese identificador junto con el
resultado dentro de una operación atómica, o verifica un estado existente
antes de producir un efecto. La estrategia depende del almacén: una
restricción única en una base relacional, una condición de creación en un
documento o una tabla de operaciones procesadas son ejemplos posibles.

La idempotencia debe cubrir todos los efectos relevantes. Registrar una fila
una sola vez no basta si también se envía un correo, se carga un blob o se
actualiza otro servicio. Cuando no puede garantizarse una operación
transaccional entre sistemas, se debe documentar compensación, reconciliación o
un estado intermedio.

## 8. Manejo de errores y DLQ

Los errores de mensajería se pueden clasificar en:

- **transitorios:** red, límite temporal o dependencia momentáneamente
  indisponible;
- **permanentes del mensaje:** esquema inválido, campo obligatorio ausente o
  versión no soportada;
- **permanentes del entorno:** permisos, configuración o recurso inexistente;
- **errores de negocio:** regla que rechaza la operación aunque el mensaje sea
  válido.

Los transitorios pueden reintentarse con límites. Los permanentes del mensaje
deben enviarse a DLQ o a un flujo de rechazo con suficiente información para
corregirlos. Los errores de configuración o permisos requieren alerta y
corrección del entorno, no una tormenta de reintentos.

Un mensaje en DLQ debe conservar su identificador, motivo, número de intentos,
fecha y referencia al error, evitando incluir secretos o datos innecesarios.
El procedimiento de recuperación debe responder:

1. ¿quién revisa la DLQ?
2. ¿cómo se distingue un error corregible de uno que debe descartarse?
3. ¿cómo se reinyecta el mensaje sin duplicar efectos?
4. ¿cómo se evita que el mensaje fallido bloquee a los demás?

## 9. Elegir entre Service Bus, Event Grid y Queue Storage

| Criterio | Service Bus | Event Grid | Queue Storage |
|---|---|---|---|
| Intención | Trabajo y mensajería empresarial | Notificación de eventos | Cola simple de trabajo |
| Distribución | Queues o topics con subscriptions | Publicador y suscripciones | Consumidores de una cola |
| Capacidades | Filtros, sesiones, DLQ y controles avanzados | Filtros y entrega de eventos | Visibilidad y operaciones básicas |
| Uso típico | Procesamiento fiable y controlado | Reacción a hechos ocurridos | Tareas sencillas y económicas |
| Elección clave | Garantías y contrato de trabajo | Evento y múltiples reacciones | Simplicidad y costo |

Usa **Service Bus** cuando el mensaje sea una unidad de trabajo de negocio,
requiera topics, suscripciones, DLQ o controles avanzados. Usa **Event Grid**
cuando el hecho de que algo ocurrió deba notificar a varios consumidores. Usa
**Queue Storage** cuando una cola sencilla resuelva el problema y no se
necesiten capacidades empresariales adicionales.

La elección también debe evaluar volumen, latencia, retención, orden,
duplicados, integración con Functions, permisos, costo y facilidad de
operación. Un diseño puede publicar un evento en Event Grid, filtrar
consumidores y poner trabajo durable en Service Bus; esa composición debe
aportar valor suficiente para justificar su complejidad.

## 10. Método de diseño

Para convertir una operación síncrona en un flujo asíncrono, usa esta
secuencia:

1. **Define el hecho o trabajo:** distingue evento de unidad de trabajo.
2. **Identifica productores y consumidores:** documenta contrato, versión y
   responsabilidades.
3. **Elige el servicio:** compara Service Bus, Event Grid y Queue Storage con
   garantías necesarias.
4. **Define estado:** registra recibido, procesando, completado, rechazado o
   pendiente de recuperación.
5. **Diseña duplicados:** asigna un identificador de operación y protección
   idempotente.
6. **Clasifica errores:** separa transitorios, permanentes y de configuración.
7. **Configura reintentos y DLQ:** establece límites, backoff, alertas y
   procedimiento de recuperación.
8. **Prueba fallos:** simula timeout, reentrega, caída del consumidor, mensaje
   inválido y efecto parcial.

## Cierre

Service Bus, Event Grid y Queue Storage resuelven problemas relacionados, pero
no equivalentes: uno controla mensajería empresarial, otro distribuye eventos
y el tercero ofrece una cola sencilla. El procesamiento asíncrono exige
contratos, estados, reintentos, idempotencia y recuperación de errores. En el
siguiente capítulo estos componentes podrán ejecutarse junto con APIs y
workers containerizados en Azure Container Apps.
