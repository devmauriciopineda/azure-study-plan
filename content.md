# Plan de estudio: desarrollo de soluciones en Microsoft Azure

Este plan de estudios abarca, en nueve capítulos semanales, desde los fundamentos de Microsoft Azure hasta el desarrollo, despliegue, seguridad, operación e integración de inteligencia artificial en aplicaciones cloud-native. El recorrido utiliza como hilo conductor una aplicación backend desarrollada con Python, expuesta mediante una API REST, conectada a bases de datos relacionales y NoSQL, almacenamiento de objetos, servicios de mensajería y componentes de IA generativa. El enfoque combina conceptos, laboratorios y decisiones de arquitectura para que, al finalizar, sea posible diseñar, implementar, asegurar, observar y operar soluciones completas en Azure.

**Duración:** 9 semanas  
**Dedicación:** 15 horas semanales  
**Total:** 135 horas  
**Enfoque:** desarrollo backend, aplicaciones cloud-native, integración, contenedores, seguridad, operación e inteligencia artificial.

---

## Capítulo 1: Fundamentos de Azure y organización de recursos

**Objetivo:** Comprender la arquitectura fundamental de Azure, organizar recursos correctamente y aplicar los principios básicos de identidad, acceso, redes y almacenamiento.

**Temario:**
- Arquitectura y organización de Azure: tenants, subscriptions, resource groups y regiones.
- Azure Portal, Azure CLI y Azure Cloud Shell.
- Administración del ciclo de vida de los recursos y etiquetas.
- Microsoft Entra ID y Azure RBAC.
- Principio de mínimo privilegio y control de acceso.
- Azure Virtual Network, subnets y Network Security Groups.
- Storage Accounts y Blob Storage.

---

## Capítulo 2: Desarrollo de aplicaciones backend en Azure

**Objetivo:** Desplegar y configurar aplicaciones Python y APIs REST utilizando servicios administrados de Azure, seleccionando entre App Service y Azure Functions según el tipo de carga.

**Temario:**
- Azure App Service: estructura, configuración y despliegue.
- Despliegue de aplicaciones Python y APIs REST.
- Azure Functions: modelo de ejecución, triggers y bindings.
- Triggers HTTP y temporizadores.
- Azure SDK para Python.
- Configuración de aplicaciones y variables de entorno.
- Gestión de archivos y configuración específica por entorno.
- Criterios de decisión entre App Service y Functions.

---

## Capítulo 3: Bases de datos y almacenamiento de aplicaciones

**Objetivo:** Seleccionar e integrar servicios de persistencia relacional, NoSQL y almacenamiento de objetos para una aplicación Python.

**Temario:**
- Azure SQL Database: conceptos, conexión y operación básica.
- Azure Database for PostgreSQL.
- Azure Cosmos DB: modelo NoSQL, contenedores, particiones e índices.
- Criterios de selección entre bases de datos relacionales y NoSQL.
- Conectividad desde aplicaciones Python.
- Persistencia, configuración de conexiones y seguridad de acceso.
- Blob Storage: operaciones, metadata, niveles de acceso y ciclo de vida.

---

## Capítulo 4: Mensajería, integración y procesamiento asíncrono

**Objetivo:** Diseñar arquitecturas desacopladas y orientadas a eventos mediante colas, topics y servicios de integración de Azure.

**Temario:**
- Azure Service Bus: queues, topics, subscriptions y dead-letter queues.
- Azure Event Grid: eventos, publicadores y suscripciones.
- Azure Queue Storage.
- Integración de Azure Functions con servicios de mensajería.
- Procesamiento asíncrono y arquitecturas orientadas a eventos.
- Reintentos, idempotencia y manejo de errores.
- Criterios para elegir entre Service Bus, Event Grid y Queue Storage.

---

## Capítulo 5: Contenedores y aplicaciones cloud-native

**Objetivo:** Empaquetar y desplegar APIs y workers containerizados en Azure, comprendiendo cuándo utilizar contenedores administrados frente a otros modelos de ejecución.

**Temario:**
- Docker: imágenes, contenedores, Dockerfiles y redes.
- Azure Container Registry.
- Azure Container Apps.
- Despliegue de APIs y workers containerizados.
- Configuración, escalamiento y revisiones de aplicaciones.
- Publicación y versionado de imágenes.
- Comparación de escenarios de uso entre App Service, Functions y Container Apps.

---

## Capítulo 6: Kubernetes y Azure Kubernetes Service

**Objetivo:** Comprender los fundamentos de Kubernetes y operar una aplicación básica en Azure Kubernetes Service (AKS).

**Temario:**
- Arquitectura y conceptos fundamentales de Kubernetes.
- Pods, Deployments, Services y Namespaces.
- ConfigMaps y Secrets.
- Ingress y exposición de aplicaciones.
- Health probes, requests y limits.
- Horizontal Pod Autoscaler.
- AKS: arquitectura, creación, despliegue y operación básica.
- Publicación de imágenes desde Azure Container Registry hacia AKS.

---

## Capítulo 7: Seguridad, identidad y administración de APIs

**Objetivo:** Proteger aplicaciones y servicios mediante identidades administradas, autenticación, autorización, secretos, redes privadas y políticas de API.

**Temario:**
- Microsoft Entra ID: aplicaciones, autenticación y autorización.
- OAuth 2.0 y OpenID Connect.
- Managed Identities: system-assigned y user-assigned.
- Azure Key Vault: secretos, claves y certificados.
- Azure API Management: APIs, productos, políticas y autenticación.
- Azure Virtual Network, Private Endpoints y acceso privado a servicios.
- Principio de mínimo privilegio aplicado a aplicaciones y workloads.
- Protección de credenciales, certificados y configuraciones sensibles.

---

## Capítulo 8: CI/CD, observabilidad y confiabilidad

**Objetivo:** Automatizar la entrega de aplicaciones y establecer mecanismos de observabilidad, resiliencia y diseño confiable en Azure.

**Temario:**
- GitHub Actions: workflows, jobs, steps y secrets.
- Pipelines de build, test y deployment.
- Despliegues de aplicaciones y contenedores en Azure.
- Azure Monitor y Application Insights.
- Log Analytics Workspace y consultas con Kusto Query Language (KQL).
- OpenTelemetry y trazabilidad distribuida.
- Métricas, logs, alertas y dashboards.
- Resiliencia: timeouts, retries y circuit breaker.
- Alta disponibilidad y recuperación ante fallos.
- Azure Well-Architected Framework: pilares y principios de diseño.

---

## Capítulo 9: Desarrollo de soluciones con inteligencia artificial en Azure

**Objetivo:** Construir una solución de IA generativa integrada con una aplicación Python, utilizando Azure OpenAI, Azure AI Search y Azure AI Foundry de forma segura y evaluable.

**Temario:**
- Fundamentos de Azure OpenAI.
- Modelos, deployments, inferencia y consumo mediante APIs.
- Integración de Azure OpenAI con aplicaciones Python.
- Embeddings y búsqueda semántica.
- Azure AI Search: índices, indexadores, skillsets y consultas vectoriales.
- Retrieval-Augmented Generation (RAG).
- Azure AI Foundry: proyectos, modelos, evaluación y desarrollo de soluciones de IA.
- Seguridad, control de costes y límites de consumo.
- Evaluación, monitoreo y despliegue de aplicaciones con IA.

