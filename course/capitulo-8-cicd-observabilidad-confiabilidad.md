# Capítulo 8. CI/CD, observabilidad y confiabilidad

## Propósito de lectura

Al finalizar esta lectura podrás estructurar un workflow de GitHub Actions para
build, test y deployment, desplegar aplicaciones y contenedores en Azure,
recoger métricas, logs y trazas con Azure Monitor, Application Insights y Log
Analytics, y usar Kusto Query Language (KQL) para investigar. También podrás
aplicar timeouts, retries y circuit breaker, razonar sobre alta disponibilidad
y recuperación ante fallos, y revisar una solución con Azure Well-Architected
Framework.

El capítulo parte de la seguridad e identidad del capítulo anterior. Un
pipeline debe usar identidades y secrets protegidos; la observabilidad debe
proteger datos sensibles; y la confiabilidad debe considerar dependencias,
costos y recuperación, no solo que el proceso termine correctamente.

## 1. CI/CD y GitHub Actions

**CI/CD** combina integración continua y entrega o despliegue continuo. La
**integración continua (CI)** valida cambios frecuentemente mediante build,
tests, análisis y empaquetado. La **entrega continua** deja un artefacto listo
para desplegar; el **despliegue continuo** publica automáticamente cuando se
cumplen las condiciones definidas.

**GitHub Actions** es una plataforma de automatización basada en workflows.
Un **workflow** es un proceso descrito en YAML que se activa por eventos, como
un push, pull request, tag o ejecución manual. Un **job** es un conjunto de
pasos que se ejecuta en un runner. Un **step** es una acción individual, como
instalar dependencias, ejecutar pruebas o publicar una imagen.

Un workflow típico contiene:

1. checkout del código;
2. configuración de Python o Docker;
3. instalación reproducible de dependencias;
4. lint, type-check o validaciones;
5. ejecución de pruebas;
6. build del artefacto o imagen;
7. publicación en ACR u otro registro;
8. deployment a un ambiente;
9. smoke test y reporte del resultado.

Los jobs pueden depender unos de otros. Separar `test`, `build` y `deploy`
permite detener el flujo antes de publicar si una validación falla. Las
condiciones por rama o ambiente deben ser explícitas: un pull request no debe
desplegar producción por accidente.

## 2. Secrets y seguridad del pipeline

Un **secret** de GitHub Actions es un valor protegido que el workflow puede
usar sin escribirlo directamente en el archivo. No debe imprimirse, colocarse
en artefactos ni pasarse a procesos que lo registren. Los secrets de
repositorio, ambiente y organización tienen alcances diferentes; debe elegirse
el más pequeño.

Una alternativa preferible a una credencial persistente es la federación de
identidad mediante **OpenID Connect (OIDC)**. El workflow obtiene un token
temporal y Azure confía en un sujeto y condiciones concretas. Esto reduce la
necesidad de almacenar una contraseña o service principal de larga duración,
pero requiere configurar correctamente audiencia, repositorio, rama o
ambiente y permisos.

El pipeline debe usar una identidad con permisos separados:

- build y tests no necesitan administrar Azure;
- publicación necesita push al registro;
- deployment necesita modificar el servicio destino;
- rollback puede requerir permisos adicionales y aprobación.

Las credenciales de runtime no deben reutilizarse como credenciales del
pipeline. Una aplicación necesita acceso a datos; el pipeline necesita
publicar una versión. Separarlas limita el impacto de una exposición.

## 3. Pipelines de build, test y deployment

Un **build** transforma código y dependencias en un artefacto, wheel, paquete o
imagen. El build debe ser reproducible, registrar versión y excluir secretos.
En un contenedor, la etiqueta debe asociarse al commit y el deployment debe
referenciar una versión identificable.

Los **tests** pueden incluir:

- unitarios para lógica aislada;
- integración para servicios y contratos;
- pruebas de API;
- seguridad y dependencias;
- smoke tests después del despliegue.

Una prueba verde no demuestra que producción esté disponible: solo demuestra
lo que ese test observó. Las pruebas deben tener datos controlados, timeouts y
limpieza. Las integraciones con servicios Azure pueden ejecutarse contra
emuladores, recursos efímeros o ambientes dedicados según el nivel de
confianza requerido.

El **deployment** publica el artefacto en App Service, Functions, Container
Apps o AKS. Debe definir ambiente, configuración, identidad, estrategia de
promoción, migraciones y rollback. Un despliegue seguro valida primero una
revisión o porcentaje pequeño, no cambia producción sin evidencia.

Un pipeline puede separar:

1. validación de pull request;
2. build y publicación al fusionar;
3. despliegue automático a `dev`;
4. pruebas y aprobación para `test`;
5. aprobación explícita para `prod`;
6. verificación y rollback.

## 4. Despliegue de aplicaciones y contenedores en Azure

Para una aplicación en App Service o Functions, el pipeline entrega código o
paquete, runtime, variables y configuración de inicio. Para una aplicación
containerizada, construye una imagen, la publica en ACR y actualiza la
referencia del servicio. Para AKS, actualiza manifiestos o valores del
Deployment y espera que los Pods estén listos.

El artefacto debe promoverse entre ambientes sin reconstruirlo cuando sea
posible. Los valores que cambian son configuración, identidad, secretos,
ingreso, escala y recursos; el contenido de código debe permanecer
identificable. Esto reduce el riesgo de que `test` y `prod` ejecuten imágenes
distintas bajo la misma versión.

El pipeline debe registrar:

- commit y versión desplegada;
- ambiente y actor;
- imagen, tag o digest;
- resultado de validaciones;
- hora de inicio y finalización;
- revisión activa;
- evidencia del smoke test;
- rollback si ocurrió.

Un rollback de aplicación no revierte automáticamente cambios de base de
datos, mensajes ya procesados o efectos externos. Las migraciones deben ser
compatibles hacia atrás o tener un plan independiente.

## 5. Azure Monitor y Application Insights

**Azure Monitor** es el servicio de observabilidad de Azure para recopilar y
analizar métricas, logs y señales de recursos y aplicaciones. **Application
Insights** es una capacidad de monitoreo de aplicaciones que registra
solicitudes, dependencias, excepciones, rendimiento y disponibilidad, según la
instrumentación.

Una **métrica** es una serie temporal numérica, como tasa de solicitudes,
latencia, errores, CPU o mensajes pendientes. Es adecuada para alertas y
dashboards porque se agrega y consulta rápidamente. Un **log** es un registro
de eventos o mensajes con contexto; sirve para investigar detalles, pero puede
ser costoso y contener información sensible.

Application Insights puede correlacionar una solicitud con llamadas a
dependencias mediante un identificador de operación. La correlación permite
seguir una petición desde APIM o la API hasta una base de datos, Storage o
Service Bus. La instrumentación debe evitar registrar tokens, contraseñas,
contenido de documentos y datos personales innecesarios.

Una estrategia mínima define:

- señales de negocio, como documentos procesados;
- señales técnicas, como latencia, tasa de errores y saturación;
- retención y costo;
- dimensiones útiles sin alta cardinalidad excesiva;
- responsables y acciones ante alertas.

## 6. Log Analytics Workspace y KQL

Un **Log Analytics Workspace** es un espacio central para almacenar y consultar
logs de Azure Monitor. Puede recibir diagnósticos de recursos, logs de
aplicaciones, eventos y métricas convertidas según la configuración.

**Kusto Query Language (KQL)** es un lenguaje de consulta para explorar datos
de logs y construir análisis. Una consulta suele seleccionar una tabla, filtrar
por tiempo, proyectar columnas, resumir y ordenar. Un ejemplo conceptual:

```kusto
AppRequests
| where TimeGenerated > ago(1h)
| summarize requests=count(), failures=countif(Success == false)
  by bin(TimeGenerated, 5m), OperationName
| order by TimeGenerated asc
```

La consulta debe limitar tiempo y columnas para reducir costo y ruido. Las
dimensiones de alta cardinalidad, como un identificador único por solicitud,
pueden dificultar agregaciones. Para investigar un incidente, se empieza con
un intervalo y servicio concretos, se identifica una anomalía y luego se
profundiza en una operación o correlación.

Los logs de diagnóstico no aparecen automáticamente en todas las tablas ni con
la misma estructura. Hay que documentar qué recurso envía datos, qué
transformaciones se aplican y cuánto tiempo se retienen. Un workspace debe
protegerse con RBAC y controles de datos, porque los logs pueden revelar
arquitectura o información de usuarios.

## 7. OpenTelemetry y trazabilidad distribuida

**OpenTelemetry** es un estándar y conjunto de herramientas para generar,
recopilar y exportar telemetría. Sus señales principales son **traces**,
**metrics** y **logs**.

Una **traza distribuida** representa el recorrido de una operación a través de
servicios. Un **span** es una unidad de trabajo dentro de esa traza, como una
solicitud HTTP, consulta SQL o publicación de mensaje. El contexto de
propagación permite que el identificador viaje entre API, worker y
dependencias.

La instrumentación automática puede cubrir frameworks y clientes comunes; la
instrumentación manual añade spans de negocio cuando ayudan a comprender
duración o resultado. No se debe crear un span por cada detalle trivial ni
incluir secretos o contenido sensible en atributos.

La trazabilidad no prueba por sí sola causalidad. Un span ausente puede deberse
a una biblioteca no instrumentada, propagación rota o muestreo. El **sampling**
reduce costo seleccionando qué trazas conservar, pero debe mantener suficientes
ejemplos de errores y rutas críticas.

## 8. Métricas, logs, alertas y dashboards

Un **dashboard** reúne visualizaciones para comprender estado y tendencia. Una
**alerta** compara una señal con una condición y notifica o ejecuta una acción.
Una alerta útil define severidad, ventana, umbral, destinatario, enlace a
consulta y procedimiento de respuesta.

Ejemplos:

- métrica: latencia p95 de la API;
- log: excepción con `operation_id`;
- trace: llamada lenta a PostgreSQL;
- alerta: tasa de errores por encima del objetivo durante diez minutos;
- dashboard: solicitudes, errores, saturación, backlog y disponibilidad.

Evita alertar cada error aislado sin contexto. Las alertas deben representar
impacto o riesgo accionable y evitar tormentas durante una caída. El dashboard
no es una colección de gráficos: debe ayudar a responder si el sistema está
disponible, si el usuario se ve afectado, qué dependencia falla y qué acción
corresponde.

Los **SLO** (objetivos de nivel de servicio) expresan metas medibles, como
disponibilidad o latencia. Un **error budget** es la cantidad de incumplimiento
aceptable durante un periodo. Estos conceptos ayudan a decidir cuándo
priorizar confiabilidad sobre velocidad de cambio.

## 9. Resiliencia: timeouts, retries y circuit breaker

Un **timeout** limita cuánto espera una operación. Sin timeout, una conexión
colgada puede consumir hilos, workers o sockets hasta agotar capacidad. Cada
llamada externa debe tener un timeout coherente con el presupuesto total de la
solicitud.

Un **retry** vuelve a intentar una operación ante un fallo transitorio. Debe
definir máximo, backoff, jitter, clases de error y límite total. No se deben
reintentar errores de validación, autorización o configuración como si fueran
transitorios. Las escrituras requieren idempotencia para evitar duplicados.

Un **circuit breaker** protege una aplicación que llama a una dependencia
fallida. En estado **closed** deja pasar llamadas y observa errores. En
**open** rechaza o degrada rápidamente durante una ventana. En **half-open**
permite algunas pruebas para comprobar recuperación. El circuit breaker no
soluciona la dependencia: evita que una falla se propague y permite recuperar
capacidad.

Timeouts, retries y circuit breaker deben coordinarse. Reintentar una operación
después de que el circuito abrió contradice el propósito del circuito. Un
timeout de cada dependencia debe caber dentro del timeout de la solicitud
superior, dejando tiempo para respuesta y registro.

## 10. Alta disponibilidad y recuperación ante fallos

La **alta disponibilidad** busca que un sistema continúe prestando servicio
cuando fallan componentes dentro del ámbito previsto. Se logra con redundancia,
distribución, health checks, capacidad suficiente y procedimientos probados.
Más réplicas no protegen contra una dependencia única, una región caída o un
despliegue defectuoso.

La **recuperación ante fallos** incluye detectar, contener, restaurar y aprender.
Un plan debe definir:

- dependencias críticas y ámbitos de fallo;
- RTO (tiempo objetivo para recuperar);
- RPO (cantidad de datos que se acepta perder);
- copias, replicación y retención;
- pasos de failover y failback;
- responsables y comunicación;
- pruebas de restauración.

Los backups no son recuperación hasta que se demuestra que pueden restaurarse.
Un diseño debe saber qué pasa con mensajes en vuelo, operaciones parciales,
identidades, DNS, certificados, configuración y datos después de una caída.

## 11. Azure Well-Architected Framework

**Azure Well-Architected Framework** es un conjunto de principios para revisar
decisiones de arquitectura. Sus pilares son:

- **Reliability:** capacidad de resistir y recuperarse de fallos.
- **Security:** proteger identidades, datos, red y operaciones.
- **Cost Optimization:** entregar valor evitando gasto innecesario.
- **Operational Excellence:** operar, desplegar y mejorar con procesos
  repetibles.
- **Performance Efficiency:** usar recursos de forma eficiente y escalar
  según demanda.

Los pilares pueden entrar en tensión. Replicar en más regiones puede aumentar
disponibilidad, pero también costo y complejidad operativa. Cifrar o registrar
más señales puede mejorar seguridad y diagnóstico, pero exige administrar
claves, retención y presupuesto.

Una revisión útil formula:

1. ¿Qué requisito y riesgo aborda la decisión?
2. ¿Qué métrica demuestra que funciona?
3. ¿Qué dependencia o límite introduce?
4. ¿Qué ocurre si falla?
5. ¿Cuánto cuesta operar y recuperar?
6. ¿Quién puede cambiarla y cómo se audita?

## 12. Método de aplicación

Para entregar y operar un servicio, usa esta secuencia:

1. **Valida el cambio:** lint, tipos, tests y seguridad de dependencias.
2. **Construye artefacto:** versión reproducible, sin secretos, con evidencia.
3. **Publica y promueve:** identidad mínima, ambiente y aprobación apropiados.
4. **Instrumenta:** métricas, logs, traces y correlación sin datos sensibles.
5. **Define objetivos:** disponibilidad, latencia, errores, backlog y costo.
6. **Protege dependencias:** timeout, retry, circuit breaker e idempotencia.
7. **Prepara recuperación:** backup, RTO, RPO, rollback y failover.
8. **Revisa pilares:** fiabilidad, seguridad, costo, operación y rendimiento.
9. **Aprende del resultado:** usa alertas, incidentes y métricas para ajustar.

## Cierre

CI/CD convierte cambios en artefactos y despliegues repetibles; Azure Monitor,
Application Insights, Log Analytics, KQL y OpenTelemetry hacen observable el
comportamiento; timeouts, retries, circuit breaker y recuperación limitan el
impacto de fallos. Well-Architected conecta estas decisiones con fiabilidad,
seguridad, costo, operación y rendimiento. El siguiente capítulo aplicará este
marco a soluciones de inteligencia artificial en Azure.
