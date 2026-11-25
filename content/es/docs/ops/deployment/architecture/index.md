---
title: Arquitectura
description: Describe la arquitectura general de Istio y sus objetivos de diseño.
weight: 10
aliases:
  - /docs/concepts/architecture
  - /docs/ops/architecture
owner: istio/wg-environments-maintainers
test: n/a
---

Una service mesh de Istio se divide lógicamente en un **data plane** y un **control
plane**.

* El **data plane** está compuesto por un conjunto de proxies inteligentes
  ([Envoy](https://www.envoyproxy.io/)) desplegados como sidecars. Estos proxies
  median y controlan toda la comunicación de red entre los microservicios. También
  recopilan y reportan telemetría de todo el tráfico de la mesh.

* El **control plane** gestiona y configura los proxies para enrutar el tráfico.

El siguiente diagrama muestra los distintos componentes que forman el data plane y el control plane:

{{< image
    link="./arch.svg"
    alt="La arquitectura general de una aplicación basada en Istio."
    caption="Arquitectura de Istio en modo sidecar"
    >}}

## Componentes

Las siguientes secciones ofrecen una breve descripción general de cada uno de los componentes principales de Istio.

### Envoy

Istio usa una versión extendida del proxy
[Envoy](https://www.envoyproxy.io/). Envoy es un proxy de alto rendimiento
desarrollado en C++ para mediar todo el tráfico entrante y saliente de todos los
servicios de la service mesh.
Los proxies Envoy son los únicos componentes de Istio que interactúan con el tráfico
del data plane.

Los proxies Envoy se despliegan como sidecars de los servicios, lo que amplía
lógicamente los servicios con las numerosas funcionalidades integradas de Envoy,
por ejemplo:

* Descubrimiento dinámico de servicios
* Balanceo de carga
* Terminación TLS
* Proxies HTTP/2 y gRPC
* Interruptores de circuito (circuit breakers)
* Comprobaciones de estado (health checks)
* Despliegues por etapas con división de tráfico basada en porcentajes
* Inyección de fallos
* Métricas detalladas

Este despliegue de sidecars permite a Istio aplicar decisiones de políticas y extraer
telemetría detallada que puede enviarse a sistemas de monitoreo para proporcionar información
sobre el comportamiento de toda la mesh.

El modelo de proxy sidecar también te permite añadir capacidades de Istio a un
despliegue existente sin necesidad de rediseñar la arquitectura ni reescribir código.

Algunas de las funcionalidades y tareas de Istio que habilitan los proxies Envoy son:

* Funcionalidades de control de tráfico: aplica un control de tráfico detallado con reglas
  de enrutamiento avanzadas para tráfico HTTP, gRPC, WebSocket y TCP.

* Funcionalidades de resiliencia de red: configura reintentos, failovers, interruptores de circuito e
  inyección de fallos.

* Funcionalidades de seguridad y autenticación: aplica políticas de seguridad, así como control
  de acceso y limitación de tasa (rate limiting) definidos mediante la API de configuración.

* Modelo de extensiones conectables basado en WebAssembly que permite aplicar políticas
  personalizadas y generar telemetría para el tráfico de la mesh.

### Istiod

Istiod proporciona descubrimiento de servicios, configuración y gestión de certificados.

Istiod convierte las reglas de enrutamiento de alto nivel que controlan el comportamiento del tráfico en
configuraciones específicas de Envoy y las propaga a los sidecars en tiempo de ejecución.
Abstrae los mecanismos de descubrimiento de servicios específicos de cada plataforma y los sintetiza
en un formato estándar que cualquier sidecar compatible con la
[API de Envoy](https://www.envoyproxy.io/docs/envoy/latest/api/api) puede consumir.

Istio puede soportar el descubrimiento en múltiples entornos, como Kubernetes o VMs.

Puedes usar la
[API de gestión de tráfico](/es/docs/concepts/traffic-management/#introducing-istio-traffic-management)
de Istio para indicar a Istiod que refine la configuración de Envoy y así ejercer un control más granular
sobre el tráfico de tu service mesh.

La [seguridad](/es/docs/concepts/security/) de Istiod permite una autenticación sólida de servicio a servicio y
de usuario final con gestión integrada de identidades y credenciales. Puedes
usar Istio para cifrar el tráfico sin cifrar de la service mesh. Con
Istio, los operadores pueden aplicar políticas basadas en la identidad del servicio en lugar de
en identificadores de red de capa 3 o capa 4, relativamente inestables.
Además, puedes usar la [funcionalidad de autorización de Istio](/es/docs/concepts/security/#authorization)
para controlar quién puede acceder a tus servicios.

Istiod actúa como Autoridad de Certificación (CA) y genera certificados para permitir
una comunicación mTLS segura en el data plane.
