# Actividad del Capítulo 8. Pipeline observable y confiable

## Propósito

Diseñar un flujo de CI/CD y un esquema de observabilidad y resiliencia para la
API y el worker de documentos. La actividad integra GitHub Actions, despliegue
en Azure, métricas, logs, trazas, alertas, recuperación y Well-Architected sin
requerir ejecutar recursos reales.

## Caso de aplicación

La solución tiene una API desplegada en Azure y un worker que procesa mensajes.
Ambos se publican desde imágenes en ACR. La API llama a una base de datos y a
Blob Storage; el worker consume una cola y actualiza el estado del documento.

El equipo necesita:

- validar cada pull request;
- desplegar automáticamente a `dev`;
- promover a `test` y `prod` con controles apropiados;
- detectar latencia alta, errores, backlog y fallos de dependencias;
- evitar que una dependencia lenta agote la API;
- recuperar el servicio tras una versión defectuosa o una caída parcial.

No se proporcionan secretos, métricas base, objetivos de disponibilidad ni
recursos reales. Declara supuestos y marca qué debe medirse.

## Entregable

Crea un documento Markdown en `course/` llamado
`capitulo-8-entrega.md`. Incluye:

1. **Workflow de GitHub Actions:** eventos, jobs, steps, dependencias, runners,
   permisos y controles por ambiente.
2. **Pipeline:** build, test, análisis, publicación de imagen, deployment,
   smoke test, aprobación y rollback.
3. **Identidad y secrets:** OIDC o alternativa, alcance de permisos y valores
   que nunca deben registrarse.
4. **Observabilidad:** métricas, logs, traces, correlación y destino en Azure
   Monitor, Application Insights o Log Analytics.
5. **Consultas KQL:** al menos dos consultas conceptuales para errores,
   latencia, backlog o dependencias.
6. **Alertas y dashboard:** señales, umbrales, ventanas, severidad,
   responsables y acción.
7. **Resiliencia:** timeout, retry, backoff, idempotencia y circuit breaker para
   base de datos, Blob Storage y cola.
8. **Alta disponibilidad y recuperación:** réplicas, backups, RTO, RPO,
   rollback, failover y pruebas de restauración.
9. **Revisión Well-Architected:** una decisión, riesgo, métrica y mejora para
   cada pilar.

## Pasos

1. Define qué ocurre en pull request, merge a la rama principal, publicación
   de tag y promoción a cada ambiente.
2. Separa jobs de validación, build, publicación y deployment. Asigna a cada
   uno la mínima identidad necesaria y decide qué requiere aprobación.
3. Diseña el uso de OIDC o secrets protegidos. Comprueba que tokens,
   credenciales y contenido de documentos no aparezcan en logs o artefactos.
4. Define una imagen versionada y un procedimiento para verificar que la
   revisión desplegada corresponde al commit.
5. Elige métricas técnicas y de negocio: tasa de solicitudes, p95, errores,
   backlog, duración de procesamiento, consumo y disponibilidad.
6. Define logs estructurados y contexto de correlación. Indica qué campos se
   excluyen por seguridad y qué retención usarías.
7. Escribe dos consultas KQL: una para investigar errores o latencia y otra
   para revisar backlog, dependencia o tasa de éxito.
8. Configura alertas accionables y un dashboard que permita decidir si el
   incidente está en API, worker, cola, base de datos o almacenamiento.
9. Diseña timeouts y retries para fallos transitorios, circuit breaker para una
   dependencia degradada y protección idempotente para escrituras.
10. Define un escenario de rollback y otro de recuperación ante caída de una
    dependencia. Incluye RTO, RPO, datos en vuelo y prueba de restauración.
11. Revisa la propuesta con los cinco pilares de Well-Architected y registra
    al menos una tensión entre costo, rendimiento, seguridad y disponibilidad.

## Criterios de aceptación

La entrega es aceptable cuando:

- define workflow, jobs y steps de GitHub Actions con dependencias claras;
- separa build, test, publicación y deployment por ambientes;
- usa permisos mínimos y explica OIDC o protección de secrets;
- describe despliegue de aplicaciones o contenedores en Azure con smoke test;
- incluye métricas, logs, traces, correlación y destino de consulta;
- aporta dos consultas KQL conceptuales, acotadas por tiempo y propósito;
- define dashboard y alertas con umbrales, ventanas, severidad y acción;
- configura timeout, retry limitado, backoff, idempotencia y circuit breaker;
- contempla alta disponibilidad, RTO, RPO, backup, rollback y restauración;
- evalúa Reliability, Security, Cost Optimization, Operational Excellence y
  Performance Efficiency;
- identifica supuestos, límites y datos sensibles que deben excluirse;
- no ejecuta pipelines, despliegues ni operaciones destructivas.

## Resultado esperado

Obtendrás un diseño que conecta cambio de código, despliegue, señales de
operación y recuperación. Debe ser posible identificar quién puede publicar,
qué evidencia confirma un despliegue, cómo se detecta un problema y qué pasos
reducen su impacto.

## Preguntas abiertas para la revisión

- ¿Qué jobs podrían ejecutarse con permisos de solo lectura?
- ¿Qué señal diferencia una API lenta de una base de datos lenta?
- ¿Qué ocurre si los retries aumentan la carga de una dependencia ya
  degradada?
- ¿Qué datos de logs deben enmascararse o excluirse?
- ¿El RTO definido es compatible con restaurar datos y renovar identidades?
- ¿Qué pilar de Well-Architected recibe menos cobertura y por qué?

## Fuera de alcance

No conectes repositorios reales, no ejecutes workflows, no agregues secretos,
no despliegues recursos ni generes dashboards reales. No desarrolles todavía
Azure OpenAI, AI Search, RAG o evaluación de IA; esos temas pertenecen al
Capítulo 9.
