# Actividad del Capítulo 6. Desplegar una API básica en AKS

## Propósito

Diseñar el despliegue de la API de documentos en AKS a partir de la imagen
publicada en ACR. La actividad integra objetos fundamentales de Kubernetes,
configuración, exposición, salud, recursos, autoscaling y permisos sin exigir
crear un clúster real.

## Caso de aplicación

La solución tiene una API Python que consulta documentos y un worker que
consume mensajes de procesamiento. Ambos usan imágenes versionadas en ACR:

- la API debe recibir tráfico HTTPS externo;
- el worker debe permanecer interno y consumir mensajes;
- la API requiere configuración no sensible y acceso a un secreto;
- ambos componentes deben ejecutarse en el Namespace `dev`;
- la API debe tener al menos dos réplicas y el worker debe escalar según la
  carga de mensajes;
- un despliegue nuevo debe poder validarse y revertirse.

No se proporciona un clúster, rangos de red, métricas ni valores reales de
secretos. Declara supuestos y especifica qué debe verificarse.

## Entregable

Crea un documento Markdown en `course/` llamado
`capitulo-6-entrega.md`. Incluye:

1. **Arquitectura:** diagrama de control plane, nodes, Namespace, API, worker,
   Services, Ingress, ACR y dependencias.
2. **Manifiestos:** YAML o pseudomanifiestos para Namespace, ConfigMap, Secret,
   Deployments, Services, probes y HPA.
3. **API:** selector, réplicas, puerto, Service, Ingress, TLS y rutas expuestas.
4. **Worker:** trigger o mecanismo de consumo, réplicas, permisos, ausencia de
   Ingress y estrategia ante reentrega.
5. **Recursos:** requests y limits de CPU y memoria con justificación.
6. **Salud:** startup, readiness y liveness probes, endpoints, tiempos y
   consecuencias de cada fallo.
7. **Autoscaling:** métrica, mínimo, máximo, condiciones de estabilización y
   límites impuestos por las dependencias.
8. **AKS y ACR:** node pools, identidad, acceso de pull, versión de imagen,
   contexto y Namespace de despliegue.
9. **Operación:** validación, actualización rolling, rollback, errores
   esperados y supuestos.

## Pasos

1. Dibuja la ruta de tráfico externo hasta la API y separa el flujo interno del
   worker. Indica qué componentes no deben exponerse.
2. Define etiquetas y selectores únicos. Explica cómo un Service encontrará
   únicamente los Pods de la API.
3. Elige un Deployment para API y otro para worker. Define imagen por digest o
   tag inmutable, réplicas y estrategia de actualización.
4. Crea un ConfigMap con valores no sensibles y un Secret referenciado sin
   escribir su valor. Explica quién puede leer cada uno.
5. Define startup, readiness y liveness probes. Incluye qué dependencia debe
   comprobar cada endpoint y por qué no reiniciarías todo el Pod ante cualquier
   error externo.
6. Propón requests y limits de CPU/memoria usando supuestos explícitos.
   Relaciónalos con número de réplicas y capacidad de los nodos.
7. Diseña HPA para la API y el worker. Usa una métrica apropiada para cada
   carga y establece mínimos, máximos y protección de la base de datos o cola.
8. Describe cómo AKS extraerá imágenes de ACR y qué revisarías ante
   `ImagePullBackOff`.
9. Escribe una secuencia de despliegue segura: verificar contexto, aplicar
   Namespace y configuración, desplegar, esperar readiness, probar Ingress,
   revisar logs y ejecutar rollback si corresponde.

## Criterios de aceptación

La entrega es aceptable cuando:

- explica control plane, nodes, Pods y modelo declarativo;
- usa Pods mediante Deployments, no como unidades manuales permanentes;
- define Services con selectores y puertos coherentes;
- separa API y worker dentro de un Namespace explícito;
- distingue ConfigMap de Secret y evita valores sensibles en el documento;
- define Ingress solo para la API y describe TLS y rutas;
- configura startup, readiness y liveness con consecuencias distintas;
- incluye requests, limits y una justificación relacionada con carga;
- propone HPA con métricas, mínimos, máximos y límites de dependencias;
- describe AKS, node pools, contexto y permisos de extracción desde ACR;
- contempla actualización rolling, rollback, reentrega del worker y errores de
  imagen o configuración;
- incluye supuestos y verificaciones observables;
- no requiere ejecutar un clúster ni adelanta políticas completas de Key Vault,
  API Management o CI/CD.

## Resultado esperado

Obtendrás una especificación suficientemente precisa para convertirla en
manifiestos y desplegar una API y un worker en AKS. La propuesta debe
demostrar que exposición, salud, recursos, autoscaling, permisos e imágenes
forman un sistema coherente.

## Preguntas abiertas para la revisión

- ¿Qué ocurre si Ingress está disponible pero ningún Pod está ready?
- ¿La métrica de CPU representa el trabajo del worker o debe usarse backlog?
- ¿Los requests caben en los node pools elegidos durante una actualización?
- ¿Qué permisos separan lectura de ConfigMap, Secret y extracción de ACR?
- ¿El rollback de imagen es compatible con cambios de esquema o mensajes?
- ¿Qué regla evita exponer accidentalmente el Service del worker?

## Fuera de alcance

No crees ni administres un clúster real, no publiques imágenes, no incluyas
secretos y no implementes pipelines. No desarrolles todavía OAuth, Key Vault,
Private Endpoints, API Management, dashboards ni políticas avanzadas de
seguridad; se tratarán en capítulos posteriores.
