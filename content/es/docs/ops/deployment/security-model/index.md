---
title: Modelo de seguridad
description: Describe el modelo de seguridad de Istio.
weight: 10
owner: istio/wg-security-maintainers
test: n/a
---

Este documento tiene como objetivo describir la postura de seguridad de los distintos componentes de Istio y cómo los posibles ataques pueden afectar al sistema.

## Componentes

Istio incluye una variedad de componentes opcionales que se tratarán aquí.
Para una visión general de alto nivel, consulta [Arquitectura de Istio](/es/docs/ops/deployment/architecture/).
Ten en cuenta que los despliegues de Istio son muy flexibles; a continuación, asumiremos principalmente los peores escenarios.

### Istiod

Istiod es el componente principal del control plane de Istio y a menudo desempeña el rol de [componente que sirve XDS](/es/docs/concepts/traffic-management/), así
como el de [Autoridad de Certificación de mTLS](/es/docs/concepts/security/) de la mesh.

Istiod se considera un componente altamente privilegiado, de forma similar al propio API server de Kubernetes.
* Tiene privilegios elevados de RBAC de Kubernetes, que normalmente incluyen acceso de lectura a `Secret` y acceso de escritura a webhooks.
* Cuando actúa como CA, puede aprovisionar certificados arbitrarios.
* Cuando actúa como control plane de XDS, puede programar los proxies para que tengan un comportamiento arbitrario.

Por ello, la seguridad del clúster está estrechamente ligada a la seguridad de Istiod.
Seguir las [buenas prácticas de seguridad de Kubernetes](https://kubernetes.io/docs/concepts/security/) en torno al acceso a Istiod es fundamental.

### Plugin CNI de Istio {#istio-cni-plugin}

Istio se puede desplegar opcionalmente con el [`DaemonSet` del plugin CNI de Istio](/es/docs/setup/additional-setup/cni/).
Este `DaemonSet` es responsable de configurar las reglas de red de Istio para garantizar que el tráfico se redirija de forma transparente según sea necesario.
Es una alternativa al contenedor `istio-init` que se describe [más adelante](#sidecar-proxies).

Como el `DaemonSet` de CNI modifica las reglas de red del nodo, requiere un `securityContext` elevado.
Sin embargo, a diferencia de [Istiod](#istiod), se trata de un privilegio **local al nodo**.
Las implicaciones de esto se describen [más adelante](#node-compromise).

Como esto concentra los privilegios elevados necesarios para configurar la red en un único pod, en lugar de en *cada* pod,
generalmente se recomienda esta opción.

### Proxies sidecar {#sidecar-proxies}

Istio puede desplegar [opcionalmente](/es/docs/overview/dataplane-modes/) un proxy sidecar junto a una aplicación.

El proxy sidecar necesita que la red esté programada para dirigir todo el tráfico a través del proxy.
Esto se puede hacer con el [plugin CNI de Istio](#istio-cni-plugin) o desplegando un `initContainer` (`istio-init`) en el pod (esto se hace automáticamente si el plugin CNI no está desplegado).
El contenedor `istio-init` requiere las capacidades `NET_ADMIN` y `NET_RAW`.
Sin embargo, estas capacidades solo están presentes durante la inicialización; el contenedor sidecar principal no tiene ningún privilegio.

Además, el proxy sidecar no requiere ningún privilegio de RBAC de Kubernetes asociado.

Cada proxy sidecar está autorizado a solicitar un certificado para la cuenta de servicio (Service Account) del Pod asociado.

### Gateways y waypoints {#gateways-and-waypoints}

Los {{< gloss "gateway" >}}gateways{{< /gloss >}} y los {{< gloss "waypoint">}}waypoints{{< /gloss >}} actúan como despliegues de proxy independientes.
A diferencia de los [sidecars](#sidecar-proxies), no requieren ninguna modificación de red y, por tanto, no requieren ningún privilegio.

Estos componentes se ejecutan con sus propias cuentas de servicio, distintas de las identidades de las aplicaciones.

### Ztunnel

{{< gloss "ztunnel" >}}Ztunnel{{< /gloss >}} actúa como un proxy a nivel de nodo.
Esta tarea requiere las capacidades `NET_ADMIN`, `SYS_ADMIN` y `NET_RAW`.
Al igual que en el [plugin CNI de Istio](#istio-cni-plugin), se trata únicamente de privilegios **locales al nodo**.
Ztunnel no tiene ningún privilegio de RBAC de Kubernetes asociado.

Ztunnel está autorizado a solicitar certificados para cualquier cuenta de servicio de los pods que se ejecutan en el mismo nodo.
Al igual que con el [kubelet](https://kubernetes.io/docs/reference/access-authn-authz/node/), esto explícitamente no permite solicitar certificados
arbitrarios.
De nuevo, esto garantiza que estos privilegios sean únicamente **locales al nodo**.

## Propiedades de la captura de tráfico

Cuando un pod se incorpora a la mesh, todo el tráfico TCP entrante se redirige al proxy.
Esto incluye tanto el tráfico mTLS/{{< gloss >}}HBONE{{< /gloss >}} como el tráfico en texto plano.
Las [políticas](/es/docs/tasks/security/authorization/) que correspondan al workload se aplican antes de reenviar el tráfico al workload.

Sin embargo, Istio actualmente no garantiza que el tráfico _saliente_ se redirija al proxy.
Consulta las [limitaciones de la captura de tráfico](/es/docs/ops/best-practices/security/#understand-traffic-capture-limitations).
Por ello, si se requieren políticas para el tráfico saliente, hay que seguir con cuidado los pasos para [proteger el tráfico de salida](/es/docs/ops/best-practices/security/#securing-egress-traffic).

## Propiedades de TLS mutuo

El [TLS mutuo](/es/docs/concepts/security/#mutual-tls-authentication) es la base de gran parte de la postura de seguridad de Istio.
A continuación se explican varias propiedades que el TLS mutuo aporta a la postura de seguridad de Istio.

### Autoridad de certificación

Istio incluye de serie su propia Autoridad de Certificación.

Por defecto, la CA permite autenticar a los clientes mediante cualquiera de las siguientes opciones:
* Un token JWT de Kubernetes, con la audiencia `istio-ca`, verificado con un `TokenReview` de Kubernetes. Este es el método por defecto en los Pods de Kubernetes.
* Un certificado de TLS mutuo existente.
* Tokens JWT personalizados, verificados mediante OIDC (requiere configuración).

La CA solo emitirá los certificados solicitados para identidades para las que el cliente esté autenticado.

Istio también se puede integrar con una variedad de CAs de terceros; consulta la documentación de seguridad de cada una para obtener más información sobre su comportamiento.

### mTLS del cliente

{{< tabset category-name="dataplane" >}}
{{< tab name="Sidecar mode" category-value="sidecar" >}}
En modo sidecar, el sidecar del cliente [usará TLS automáticamente](/es/docs/ops/configuration/traffic-management/tls-configuration/#auto-mtls) al conectarse a un servicio
que se detecte que soporta mTLS. Esto también se puede [configurar explícitamente](/es/docs/ops/configuration/traffic-management/tls-configuration/#sidecars).
Ten en cuenta que esta detección automática depende de que Istio asocie el tráfico a un Service.
Los [tipos de tráfico no soportados](/es/docs/ops/configuration/traffic-management/traffic-routing/#unmatched-traffic) o la [limitación del ámbito de la configuración](/es/docs/ops/configuration/mesh/configuration-scoping/) pueden impedirlo.

Al [conectarse a un backend](/es/docs/concepts/security/#secure-naming), el conjunto de identidades permitidas se calcula, a nivel de Service, a partir de la unión de las identidades de todos los backends.
{{< /tab >}}

{{< tab name="Ambient mode" category-value="ambient" >}}
En modo ambient, Istio usará automáticamente mTLS al conectarse a cualquier backend que soporte mTLS, y verificará que la identidad del destino coincida con la identidad con la que se espera que se ejecute el workload.

Estas propiedades difieren del modo sidecar en que son propiedades de workloads individuales, en lugar del servicio.
Esto permite comprobaciones de autenticación más detalladas, además de soportar una mayor variedad de workloads.
{{< /tab >}}
{{< /tabset >}}

### mTLS del servidor

Por defecto, Istio acepta tráfico mTLS y no mTLS (a menudo llamado "modo permisivo").
Los usuarios pueden optar por la aplicación estricta escribiendo reglas `PeerAuthentication` o `AuthorizationPolicy` que requieran mTLS.

Cuando se establecen conexiones mTLS, se verifica el certificado del par (peer).
Además, se verifica que la identidad del par pertenezca al mismo trust domain.
Para verificar que solo se permiten identidades específicas, se puede usar una `AuthorizationPolicy`.

## Tipos de compromiso analizados

A partir de la descripción general anterior, consideraremos el impacto en el clúster si se comprometen distintas partes del sistema.
En el mundo real, hay una variedad de variables distintas en torno a cualquier ataque de seguridad:

* Lo fácil que es ejecutarlo
* Qué privilegios previos se requieren
* Con qué frecuencia se puede explotar
* Cuál es el impacto (ejecución remota total, denegación de servicio, etc.).

En este documento, consideraremos principalmente el peor escenario: un componente comprometido significa que un atacante tiene capacidades completas de ejecución remota de código.

### Compromiso del workload {#workload-compromise}

En este escenario, se compromete un workload de aplicación (pod).

Un pod [*puede* tener acceso](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#opt-out-of-api-credential-automounting) a su token de cuenta de servicio.
Si es así, el compromiso de un workload puede moverse lateralmente desde un único pod hasta comprometer toda la cuenta de servicio.

{{< tabset category-name="dataplane" >}}
{{< tab name="Sidecar mode" category-value="sidecar" >}}
En el modelo sidecar, el proxy está ubicado junto con el pod y se ejecuta dentro del mismo límite de confianza.
Una aplicación comprometida puede manipular el proxy a través de la API de administración u otras superficies, incluida la exfiltración del material de clave privada, lo que permite que otro agente suplante al workload.
Se debe asumir que un workload comprometido también implica el compromiso del proxy sidecar.

Dado esto, un workload comprometido puede:
* Enviar tráfico arbitrario, con o sin TLS mutuo.
  Este tráfico puede eludir cualquier configuración del proxy, o incluso el proxy por completo.
  Ten en cuenta que Istio no ofrece políticas de autorización basadas en el tráfico de salida, por lo que no se elude ninguna política de autorización de salida.
* Aceptar tráfico que ya estaba destinado a la aplicación. Puede eludir las políticas configuradas en el proxy sidecar.

La conclusión clave es que, aunque el workload comprometido pueda comportarse de forma maliciosa, esto no le da la capacidad de eludir las políticas de _otros_ workloads.
{{< /tab >}}

{{< tab name="Ambient mode" category-value="ambient" >}}
En modo ambient, el proxy de nodo no está ubicado dentro del pod y se ejecuta en otro límite de confianza como parte de un pod independiente.

Una aplicación comprometida puede enviar tráfico arbitrario.
Sin embargo, no tiene control sobre el proxy de nodo, que decide cómo gestionar el tráfico entrante y saliente.

Además, como el propio pod no tiene acceso a un token de cuenta de servicio para solicitar un certificado de TLS mutuo, se reducen las posibilidades de movimiento lateral.
{{< /tab >}}
{{< /tabset >}}

Istio ofrece una variedad de funcionalidades que pueden limitar el impacto de un compromiso de este tipo:
* Las funcionalidades de [observabilidad](/es/docs/tasks/observability/) se pueden usar para identificar el ataque.
* Las [políticas](/es/docs/tasks/security/authorization/) se pueden usar para restringir qué tipo de tráfico puede enviar o recibir un workload.

### Compromiso del proxy - Sidecars {#proxy-compromise---sidecars}

En este escenario, se compromete un proxy sidecar.
Como el sidecar y la aplicación residen en el mismo trust domain, esto es funcionalmente equivalente al [compromiso del workload](#workload-compromise).

### Compromiso del proxy - Waypoint {#proxy-compromise---waypoint}

En este escenario, se compromete un [waypoint proxy](#gateways-and-waypoints).
Aunque los waypoints no tienen ningún privilegio que un atacante pueda explotar, sí sirven (potencialmente) a muchos servicios y workloads diferentes.
Un waypoint comprometido recibirá todo el tráfico de estos, que puede ver, modificar o descartar.

Istio ofrece la flexibilidad de [configurar la granularidad del despliegue de un waypoint](/es/docs/ambient/usage/waypoint/#useawaypoint).
Los usuarios pueden considerar desplegar waypoints más aislados si requieren un aislamiento más fuerte.

Como los waypoints se ejecutan con una identidad distinta a la de las aplicaciones a las que sirven, un waypoint comprometido no implica que se pueda suplantar a las aplicaciones del usuario.

### Compromiso del proxy - Ztunnel {#proxy-compromise---ztunnel}

En este escenario, se compromete un proxy [ztunnel](#ztunnel).

Un ztunnel comprometido le da al atacante el control de la red del nodo.

Ztunnel tiene acceso al material de clave privada de cada aplicación que se ejecuta en su nodo.
Si un ztunnel se ve comprometido, ese material podría exfiltrarse y usarse en otro lugar.
Sin embargo, no es posible el movimiento lateral hacia identidades más allá de los workloads ubicados en el mismo nodo; cada ztunnel solo está autorizado a acceder a los certificados de los workloads que se ejecutan en su nodo, lo que acota el radio de impacto (blast radius) de un ztunnel comprometido.

### Compromiso del nodo {#node-compromise}

En este escenario, se compromete el nodo de Kubernetes.
Tanto [Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/node/) como Istio están diseñados para limitar el radio de impacto del compromiso de un único nodo, de modo que
el compromiso de un único nodo no conduzca a un [compromiso de todo el clúster](#cluster-api-server-compromise).

Sin embargo, el atacante sí tiene control total sobre cualquier workload que se ejecute en ese nodo.
Por ejemplo, puede comprometer cualquier [waypoint](#proxy-compromise---waypoint) ubicado en el mismo nodo, el [ztunnel](#proxy-compromise---ztunnel) local, cualquier [sidecar](#proxy-compromise---sidecars), cualquier [instancia de Istiod](#istiod-compromise) ubicada en el mismo nodo, etc.

### Compromiso del clúster (API Server) {#cluster-api-server-compromise}

Un compromiso del API Server de Kubernetes significa, en la práctica, que todo el clúster y toda la mesh están comprometidos.
A diferencia de la mayoría de los demás vectores de ataque, Istio no puede hacer mucho para controlar el radio de impacto de un ataque así.
Un API Server comprometido le da a un atacante control total sobre el clúster, incluidas acciones como ejecutar `kubectl exec` en pods arbitrarios,
eliminar cualquiera de las `AuthorizationPolicies` de Istio o incluso desinstalar Istio por completo.

### Compromiso de Istiod {#istiod-compromise}

Un compromiso de Istiod generalmente conduce al mismo resultado que un [compromiso del API Server](#cluster-api-server-compromise).
Istiod es un componente altamente privilegiado que debe protegerse a conciencia.
Seguir las [buenas prácticas de seguridad](/es/docs/ops/best-practices/security) es crucial para mantener un clúster seguro.

### Compromiso de una cuenta de Kubernetes {#k8s-account-compromise}

En este escenario, se compromete una cuenta de Kubernetes con permisos para desplegar recursos de Istio (``networking.istio.io/v1``).

Una cuenta comprometida de este tipo le da al atacante el control para desplegar recursos que gestionan el tráfico en la service mesh. Aunque los CRDs de gestión de tráfico son _recursos con namespace_ (namespaced resources), afectan al comportamiento de enrutamiento del tráfico de servicios externos al namespace y, por tanto, a la seguridad de la service mesh.

Por ejemplo, un `VirtualService` desplegado con la [opción de gateway mesh](/es/docs/reference/config/networking/virtual-service/#VirtualService-gateways) permite ataques Man-in-the-Middle dentro de la service mesh. El servicio controlado por el atacante puede:

* actuar como el servicio de destino.
* redirigir el tráfico a destinos alternativos.
* descartar solicitudes para interrumpir la comunicación (denegación de servicio).

Aunque el atacante puede intentar reenviar el tráfico al servicio de destino para leer o modificar la comunicación entre los dos servicios, puede fallar, ya que no puede eludir las [funcionalidades de seguridad de capa 4 y capa 7](/es/docs/overview/dataplane-modes/#layer-4-vs-layer-7-features) de Istio, según la configuración de la mesh.

Puedes encontrar más detalles y posibles mitigaciones en la [entrada del blog sobre _Security Considerations on Istio's CRDs with Namespace-based Multi-Tenancy_](/blog/2026/security-considerations-on-namespace-based-multi-tenancy/).
