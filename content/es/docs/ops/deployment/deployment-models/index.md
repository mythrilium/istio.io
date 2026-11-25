---
title: Modelos de despliegue
description: Describe las opciones y consideraciones al configurar tu despliegue de Istio.
weight: 20
keywords:
  - single-cluster
  - multiple-clusters
  - control-plane
  - tenancy
  - networks
  - identity
  - trust
  - single-mesh
  - multiple-meshes
aliases:
  - /docs/concepts/multicluster-deployments
  - /docs/concepts/deployment-models
  - /docs/ops/prep/deployment-models
owner: istio/wg-environments-maintainers
test: n/a
---

Al configurar un despliegue de Istio en producción, necesitas responder a varias preguntas.
¿La mesh estará limitada a un único {{< gloss "cluster" >}}clúster{{< /gloss >}} o se distribuirá entre
varios clústeres? ¿Todos los servicios estarán ubicados en una única red completamente conectada, o se requerirán
gateways para conectar servicios entre varias redes? ¿Hay un único
{{< gloss >}}control plane{{< /gloss >}}, potencialmente compartido entre clústeres,
o hay varios control planes desplegados para garantizar la alta disponibilidad (HA)?
¿Se conectarán todos los clústeres en una única service mesh {{< gloss "multicluster" >}}multiclúster{{< /gloss >}}
o se federarán en un despliegue {{< gloss >}}multi-mesh{{< /gloss >}}?

Todas estas preguntas, entre otras, representan dimensiones independientes de configuración de un despliegue de Istio.

1. uno o varios clústeres
1. una o varias redes
1. uno o varios control planes
1. una o varias meshes

En un entorno de producción con varios clústeres, puedes usar una combinación
de modelos de despliegue. Por ejemplo, se recomienda tener más de un control plane para la HA,
pero podrías lograrlo en un despliegue de 3 clústeres desplegando 2 clústeres con
un único control plane compartido y luego añadiendo el tercer clúster con un segundo
control plane en una red diferente. Después, los tres clústeres podrían configurarse
para compartir ambos control planes, de modo que todos los clústeres tengan 2 fuentes de control
para garantizar la HA.

Elegir el modelo de despliegue adecuado depende de los requisitos de aislamiento, rendimiento
y HA de tu caso de uso. Esta guía describe las distintas opciones y
consideraciones al configurar tu despliegue de Istio.

## Modelos de clúster {#cluster-models}

Las instancias de workload de tu aplicación se ejecutan en uno o más
{{< gloss "cluster" >}}clústeres{{< /gloss >}}. Para lograr aislamiento, rendimiento y
alta disponibilidad, puedes limitar los clústeres a zonas de disponibilidad y regiones.

Los sistemas de producción, según sus requisitos, pueden ejecutarse en varios
clústeres que abarcan varias zonas o regiones, aprovechando los balanceadores de carga en la nube
para gestionar aspectos como la localidad y el failover zonal o regional.

En la mayoría de los casos, los clústeres representan límites para la configuración y el descubrimiento
de endpoints. Por ejemplo, cada clúster de Kubernetes tiene un API Server que gestiona
la configuración del clúster y, además, sirve la información de los
{{< gloss "service endpoint" >}}endpoints de servicio{{< /gloss >}} a medida que los pods se levantan
o se detienen. Como Kubernetes configura este comportamiento por clúster, este
enfoque ayuda a limitar los posibles problemas causados por configuraciones incorrectas.

En Istio, puedes configurar una única service mesh para que abarque cualquier número de
clústeres.

### Clúster único

En el caso más simple, puedes limitar una mesh de Istio a un único
{{< gloss "cluster" >}}clúster{{< /gloss >}}. Un clúster normalmente opera sobre una
[red única](#single-network), aunque esto varía entre proveedores de
infraestructura. Un modelo de clúster único y red única incluye un control plane,
lo que da lugar al despliegue de Istio más simple.

{{< image width="50%"
    link="single-cluster.svg"
    alt="Una service mesh con un único clúster"
    title="Clúster único"
    caption="Una service mesh con un único clúster"
    >}}

Los despliegues de un único clúster ofrecen simplicidad, pero carecen de otras funcionalidades, por
ejemplo, el aislamiento de fallos y el failover. Si necesitas una mayor disponibilidad,
deberías usar varios clústeres.

### Varios clústeres {#multiple-clusters}

Puedes configurar una única mesh para que incluya
varios {{< gloss "cluster" >}}clústeres{{< /gloss >}}. Usar un despliegue
{{< gloss "multicluster" >}}multiclúster{{< /gloss >}} dentro de una única mesh ofrece
las siguientes capacidades, más allá de las de un despliegue de un único clúster:

- Aislamiento de fallos y failover: si `cluster-1` cae, se hace failover a `cluster-2`.
- Enrutamiento y failover según la ubicación: envía las solicitudes al servicio más cercano.
- Distintos [modelos de control plane](#control-plane-models): soportan distintos
  niveles de disponibilidad.
- Aislamiento por equipo o proyecto: cada equipo ejecuta su propio conjunto de clústeres.

{{< image width="75%"
    link="multi-cluster.svg"
    alt="Una service mesh con varios clústeres"
    title="Multiclúster"
    caption="Una service mesh con varios clústeres"
    >}}

Los despliegues multiclúster te ofrecen un mayor grado de aislamiento y
disponibilidad, pero aumentan la complejidad. Si tus sistemas tienen requisitos de alta
disponibilidad, probablemente necesites clústeres en varias zonas y regiones. Puedes
hacer despliegues canary de cambios de configuración o de nuevas versiones de binarios en un único clúster,
donde los cambios de configuración solo afectan a una pequeña parte del tráfico de los usuarios.
Además, si un clúster tiene un problema, puedes enrutar temporalmente el tráfico a
clústeres cercanos hasta que resuelvas el problema.

Puedes configurar la comunicación entre clústeres en función de la
[red](#network-models) y de las opciones que soporte tu proveedor de la nube. Por
ejemplo, si dos clústeres residen en la misma red subyacente, puedes habilitar
la comunicación entre clústeres simplemente configurando reglas de firewall.

Dentro de una mesh multiclúster, todos los servicios se comparten por defecto, de acuerdo con el
concepto de {{< gloss "namespace sameness" >}}namespace sameness{{< /gloss >}}.
Las [reglas de gestión de tráfico](/es/docs/ops/configuration/traffic-management/multicluster)
proporcionan un control detallado sobre el comportamiento del tráfico multiclúster.

### DNS con varios clústeres {#dns-with-multiple-clusters}

Cuando una aplicación cliente hace una solicitud a algún host, primero debe realizar una
búsqueda DNS del nombre de host para obtener una dirección IP antes de poder continuar con
la solicitud.
En Kubernetes, el servidor DNS que reside dentro del clúster normalmente gestiona
esta búsqueda DNS, según las definiciones de `Service` configuradas.

Istio usa la IP virtual devuelta por la búsqueda DNS para balancear la carga
entre la lista de endpoints activos del servicio solicitado, teniendo en cuenta las
reglas de enrutamiento configuradas en Istio.
Istio usa los `Service`/`Endpoint` de Kubernetes o los `ServiceEntry` de Istio para
configurar su correspondencia interna entre nombres de host y direcciones IP de workloads.

Este sistema de nombres de dos niveles se vuelve más complicado cuando tienes varios
clústeres. Istio está preparado de forma nativa para entornos multiclúster, pero Kubernetes no lo está
(hoy en día). Por ello, el clúster cliente debe tener una entrada DNS para el
servicio para que la búsqueda DNS tenga éxito y la solicitud se envíe
correctamente. Esto se cumple incluso si no hay instancias de los pods de ese
servicio ejecutándose en el clúster cliente.

Para garantizar que la búsqueda DNS tenga éxito, debes desplegar un `Service` de Kubernetes en
cada clúster que consuma ese servicio. Esto garantiza que, independientemente de
dónde se origine la solicitud, superará la búsqueda DNS y se entregará a Istio
para un enrutamiento adecuado.
Esto también se puede lograr con un `ServiceEntry` de Istio, en lugar de con un
`Service` de Kubernetes. Sin embargo, un `ServiceEntry` no configura el servidor DNS de Kubernetes.
Esto significa que el DNS tendrá que configurarse manualmente o
con herramientas automatizadas, como la funcionalidad de
[asignación automática de direcciones](/es/docs/ops/configuration/traffic-management/dns-proxy/#address-auto-allocation)
del [proxy DNS de Istio](/es/docs/ops/configuration/traffic-management/dns-proxy/).

{{< tip >}}
Hay algunas iniciativas en curso que ayudarán a simplificar la gestión del DNS:

- [Admiral](https://github.com/istio-ecosystem/admiral) es un proyecto de la comunidad
  de Istio que proporciona varias capacidades multiclúster. Si necesitas soportar topologías
  multi-network, gestionar esta configuración en varios clústeres a escala es un desafío.
  Admiral adopta un enfoque propio (opinionated) sobre esta configuración y proporciona aprovisionamiento y
  sincronización automáticos entre clústeres.

- [Kubernetes Multi-Cluster Services](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api)
  es una Kubernetes Enhancement Proposal (KEP) que define una API para exportar
  servicios a varios clústeres. En la práctica, esto traslada a Kubernetes la responsabilidad de
  la visibilidad de los servicios y la resolución DNS de todo el `clusterset`.
  También hay trabajo en curso para incorporar capas de soporte de `MCS`
  en Istio, lo que permitiría a Istio funcionar con cualquier controlador `MCS` de un proveedor de la nube
  o incluso actuar como el controlador `MCS` de toda la mesh.
{{< /tip >}}

## Modelos de red {#network-models}

Istio usa una definición simplificada de {{< gloss "network" >}}red{{< /gloss >}} para
referirse a las {{< gloss "workload instance" >}}instancias de workload{{< /gloss >}} que tienen
conectividad directa. Por ejemplo, por defecto todas las instancias de workload de un único
clúster están en la misma red.

Muchos sistemas de producción requieren varias redes o subredes para el aislamiento
y la alta disponibilidad. Istio permite extender una service mesh sobre una variedad de
topologías de red. Este enfoque te permite seleccionar el modelo de red que
se ajuste a tu topología de red existente.

### Red única {#single-network}

En el caso más simple, una service mesh opera sobre una única red completamente
conectada. En un modelo de red única, todas las
{{< gloss "workload instance" >}}instancias de workload{{< /gloss >}}
pueden comunicarse directamente entre sí sin un gateway de Istio.

Una red única permite a Istio configurar los consumidores de servicios de forma uniforme
en toda la mesh, con la capacidad de dirigirse directamente a las instancias de workload.
Sin embargo, ten en cuenta que, para una red única que abarque varios clústeres,
los servicios y los endpoints no pueden tener direcciones IP superpuestas.

{{< image width="50%"
    link="single-net.svg"
    alt="Una service mesh con una única red"
    title="Red única"
    caption="Una service mesh con una única red"
    >}}

### Varias redes

Puedes extender una única service mesh a través de varias redes; esta
configuración se conoce como **multi-network**.

Tener varias redes ofrece las siguientes capacidades, más allá de las de una red única:

- Rangos de IP o VIP superpuestos para los **endpoints de servicio**
- Cruce de límites administrativos
- Tolerancia a fallos
- Escalado de las direcciones de red
- Cumplimiento de estándares que requieren segmentación de red

En este modelo, las instancias de workload de redes diferentes solo pueden comunicarse entre
sí a través de uno o más [gateways de Istio](/es/docs/concepts/traffic-management/#gateways).
Istio usa el **descubrimiento de servicios particionado** para proporcionar a los consumidores una
vista diferente de los {{< gloss "service endpoint" >}}endpoints de servicio{{< /gloss >}}. La vista depende de la
red de los consumidores.

{{< image width="50%"
    link="multi-net.svg"
    alt="Una service mesh con varias redes"
    title="Despliegue multi-network"
    caption="Una service mesh con varias redes"
    >}}

Esta solución requiere exponer todos los servicios (o un subconjunto) a través del gateway.
Los proveedores de la nube pueden ofrecer opciones que no requieran exponer los servicios en
la internet pública. Una opción así, si existe y cumple tus requisitos,
probablemente será la mejor elección.

{{< tip >}}
Para garantizar comunicaciones seguras en un escenario multi-network, Istio
solo soporta la comunicación entre redes con workloads que tengan un proxy de Istio.
Esto se debe a que Istio expone los servicios en el Ingress Gateway con TLS
pass-through, lo que permite mTLS directamente hasta el workload. Sin embargo, es probable que un workload sin
un proxy de Istio no pueda participar en la autenticación
mutua con otros workloads. Por este motivo, Istio filtra los
endpoints que están fuera de la red para los servicios sin proxy (proxyless).
{{< /tip >}}

## Modelos de control plane {#control-plane-models}

Una mesh de Istio usa el {{< gloss >}}control plane{{< /gloss >}} para configurar toda
la comunicación entre las instancias de workload dentro de la mesh. Las instancias de workload
se conectan a una instancia del control plane para obtener su configuración.

En el caso más simple, puedes ejecutar tu mesh con un control plane en un único
clúster.

{{< image width="50%"
    link="single-cluster.svg"
    alt="Un único clúster con un control plane"
    title="Control plane único"
    caption="Un único clúster con un control plane"
    >}}

Un clúster como este, con su propio control plane local, se denomina
{{< gloss "primary cluster" >}}clúster primario{{< /gloss >}}.

Los despliegues multiclúster también pueden compartir instancias del control plane. En este caso,
las instancias del control plane pueden residir en uno o más clústeres primarios.
Los clústeres sin su propio control plane se denominan
{{< gloss "remote cluster" >}}clústeres remotos{{< /gloss >}}.

{{< image width="75%"
    link="shared-control.svg"
    alt="Una service mesh con un clúster primario y un clúster remoto"
    title="Control plane compartido"
    caption="Una service mesh con un clúster primario y un clúster remoto"
    >}}

Para soportar clústeres remotos en una mesh multiclúster, el control plane de
un clúster primario debe ser accesible a través de una IP estable (por ejemplo, una IP de clúster).
Para los clústeres que abarcan varias redes,
esto se puede lograr exponiendo el control plane a través de un gateway de Istio.
Los proveedores de la nube pueden ofrecer opciones, como balanceadores de carga internos, para
proporcionar esta capacidad sin exponer el control plane en
la internet pública. Una opción así, si existe y cumple tus requisitos,
probablemente será la mejor elección.

En los despliegues multiclúster con más de un clúster primario, cada clúster
primario recibe su configuración (es decir, `Service` y `ServiceEntry`,
`DestinationRule`, etc.) del API Server de Kubernetes que reside en el mismo
clúster. Por lo tanto, cada clúster primario tiene una fuente de
configuración independiente.
Esta duplicación de la configuración entre clústeres primarios sí requiere
pasos adicionales al desplegar cambios. Los grandes sistemas
de producción pueden automatizar este proceso con herramientas, como sistemas de CI/CD, para
gestionar el despliegue de la configuración.

En lugar de ejecutar control planes en clústeres primarios dentro de la mesh, una
service mesh compuesta íntegramente por clústeres remotos puede estar controlada por un
{{< gloss "external control plane" >}}control plane externo{{< /gloss >}}. Esto proporciona una gestión
aislada y una separación completa entre el despliegue del control plane y los
servicios del data plane que forman la mesh.

{{< image width="100%"
    link="single-cluster-external-control-plane.svg"
    alt="Un único clúster con un control plane externo"
    title="Control plane externo"
    caption="Un único clúster con un control plane externo"
    >}}

El {{< gloss >}}control plane gestionado{{< /gloss >}} de un proveedor de la nube es un
ejemplo típico de control plane externo.

Para lograr alta disponibilidad, deberías desplegar varios control planes en distintos
clústeres, zonas o regiones.

{{< image width="75%"
    link="multi-control.svg"
    alt="Una service mesh con instancias del control plane en cada región"
    title="Varios control planes"
    caption="Una service mesh con instancias del control plane en cada región"
    >}}

Este modelo ofrece los siguientes beneficios:

- Mayor disponibilidad: si un control plane deja de estar disponible, el alcance de
  la interrupción se limita solo a los workloads de los clústeres gestionados por ese control plane.

- Aislamiento de la configuración: puedes hacer cambios de configuración en un clúster,
  zona o región sin afectar a los demás.

- Despliegue controlado: tienes un control más detallado sobre el despliegue de la configuración
  (por ejemplo, un clúster a la vez). También puedes hacer despliegues canary de cambios de configuración en una subsección de la mesh
  controlada por un clúster primario determinado.

- Visibilidad selectiva de servicios: puedes restringir la visibilidad de los servicios a una parte
  de la mesh, lo que ayuda a establecer un aislamiento a nivel de servicio. Por ejemplo, un
  administrador puede optar por desplegar el servicio `HelloWorld` en el clúster A,
  pero no en el clúster B. Cualquier intento de llamar a `HelloWorld` desde el clúster B
  fallará en la búsqueda DNS.

La siguiente lista clasifica ejemplos de despliegue del control plane según su disponibilidad:

- Un clúster por región (**menor disponibilidad**)
- Varios clústeres por región
- Un clúster por zona
- Varios clústeres por zona
- Cada clúster (**mayor disponibilidad**)

### Descubrimiento de endpoints con varios control planes

Un control plane de Istio gestiona el tráfico dentro de la mesh proporcionando a cada proxy
la lista de endpoints de servicio. Para que esto funcione en un
escenario multiclúster, cada control plane debe observar los endpoints del API
Server de cada clúster.

Para habilitar el descubrimiento de endpoints de un clúster, un administrador genera un
`remote secret` y lo despliega en cada clúster primario de la mesh. El
`remote secret` contiene credenciales que otorgan acceso al API server del
clúster.
Los control planes se conectarán entonces y descubrirán los endpoints de servicio del
clúster, lo que habilita el balanceo de carga entre clústeres para estos servicios.

{{< image width="75%"
    link="endpoint-discovery.svg"
    caption="Clústeres primarios con descubrimiento de endpoints"
    >}}

Por defecto, Istio balancea las solicitudes de forma uniforme entre los endpoints de
cada clúster. En sistemas grandes que abarcan regiones geográficas, puede ser
deseable usar el [balanceo de carga por localidad](/es/docs/tasks/traffic-management/locality-load-balancing)
para preferir que el tráfico permanezca en la misma zona o región.

En algunos escenarios avanzados, puede que no se desee el balanceo de carga entre clústeres.
Por ejemplo, en un despliegue blue/green, puedes desplegar distintas versiones del
sistema en distintos clústeres. En este caso, cada clúster opera, en la práctica,
como una mesh independiente. Este comportamiento se puede lograr de un par de
formas:

- No intercambiar remote secrets entre los clústeres. Esto ofrece el
  aislamiento más fuerte entre los clústeres.

- Usar `VirtualService` y `DestinationRule` para impedir el enrutamiento entre dos
  versiones de los servicios.

En cualquiera de los casos, se impide el balanceo de carga entre clústeres. El tráfico externo
se puede enrutar a un clúster o al otro mediante un balanceador de carga externo.

## Modelos de identidad y confianza {#identity-and-trust-models}

Cuando se crea una instancia de workload dentro de una service mesh, Istio asigna al
workload una {{< gloss "identity" >}}identidad{{< /gloss >}}.

La Autoridad de Certificación (CA) crea y firma los certificados que se usan para verificar
las identidades usadas dentro de la mesh. Puedes verificar la identidad del remitente del mensaje
con la clave pública de la CA que creó y firmó el certificado
de esa identidad. Un **trust bundle** es el conjunto de todas las claves públicas de CA que usa
una mesh de Istio. Con el trust bundle de una mesh, cualquiera puede verificar el remitente de cualquier
mensaje que provenga de esa mesh.

### Confianza dentro de una mesh

Dentro de una única mesh de Istio, Istio garantiza que cada instancia de workload tenga un
certificado adecuado que represente su propia identidad, y el trust bundle
necesario para reconocer todas las identidades dentro de la mesh y de cualquier mesh federada.
La CA crea y firma los certificados de esas identidades. Este modelo
permite que las instancias de workload de la mesh se autentiquen entre sí al
comunicarse.

{{< image width="50%"
    link="single-trust.svg"
    alt="Una service mesh con una autoridad de certificación común"
    title="Confianza dentro de una mesh"
    caption="Una service mesh con una autoridad de certificación común"
    >}}

### Confianza entre meshes {#trust-between-meshes}

Para habilitar la comunicación entre dos meshes con CAs diferentes, debes
intercambiar los trust bundles de las meshes. Istio no proporciona ninguna herramienta
para intercambiar trust bundles entre meshes. Puedes intercambiar los trust bundles
manual o automáticamente mediante un protocolo como [SPIFFE Trust Domain Federation](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE_Federation.md).
Una vez que importas un trust bundle en una mesh, puedes configurar políticas locales para
esas identidades.

{{< image width="50%"
    link="multi-trust.svg"
    alt="Varias service meshes con distintas autoridades de certificación"
    title="Confianza entre meshes"
    caption="Varias service meshes con distintas autoridades de certificación"
    >}}

## Modelos de mesh

Istio permite tener todos tus servicios en una
{{< gloss "service mesh" >}}mesh{{< /gloss >}}, o federar varias meshes
entre sí, lo que también se conoce como {{< gloss >}}multi-mesh{{< /gloss >}}.

### Mesh única

El despliegue de Istio más simple es una única mesh. Dentro de una mesh, los nombres de servicio son
únicos. Por ejemplo, solo un servicio puede llamarse `mysvc` en el namespace `foo`.
Además, las instancias de workload comparten una identidad común, ya que los
nombres de las cuentas de servicio son únicos dentro de un namespace, igual que los nombres de servicio.

Una única mesh puede abarcar [uno o más clústeres](#cluster-models) y
[una o más redes](#network-models). Dentro de una mesh, los
[namespaces](#namespace-tenancy) se usan para la [tenencia](#tenancy-models).

### Varias meshes {#multiple-meshes}

Los despliegues de varias meshes son el resultado de la {{< gloss >}}mesh federation{{< /gloss >}}.

Tener varias meshes ofrece las siguientes capacidades, más allá de las de una única mesh:

- Límites organizativos: líneas de negocio
- Reutilización de nombres de servicio o de namespaces: el namespace `default`
  usado de varias formas distintas
- Mayor aislamiento: aislar los workloads de prueba de los workloads de producción

Puedes habilitar la comunicación entre meshes con {{< gloss >}}mesh federation{{<
/gloss >}}. Al federar, cada mesh puede exponer un conjunto de servicios e
identidades, que todas las meshes participantes pueden reconocer.

{{< image width="50%"
    link="multi-mesh.svg"
    alt="Varias service meshes"
    title="Multi-mesh"
    caption="Varias service meshes"
    >}}

Para evitar colisiones de nombres de servicio, puedes asignar a cada mesh un
**ID de mesh** globalmente único, para garantizar que el nombre de dominio completo
(FQDN) de cada servicio sea distinto.

Al federar dos meshes que no comparten el mismo
{{< gloss >}}trust domain{{< /gloss >}}, debes federar la
{{< gloss "identity" >}}identidad{{< /gloss >}} y los **trust bundles** entre ellas. Consulta la
sección [Confianza entre meshes](#trust-between-meshes) para más detalles.

## Modelos de tenencia {#tenancy-models}

En Istio, un **inquilino** (tenant) es un grupo de usuarios que comparten
acceso y privilegios comunes sobre un conjunto de workloads desplegados.
Los inquilinos se pueden usar para proporcionar un nivel de aislamiento entre distintos equipos.

Puedes configurar los modelos de tenencia para satisfacer los siguientes requisitos
organizativos de aislamiento:

- Seguridad
- Políticas
- Capacidad
- Costo
- Rendimiento

Istio soporta tres tipos de modelos de tenencia:

- [Tenencia por namespace](#namespace-tenancy)
- [Tenencia por clúster](#cluster-tenancy)
- [Tenencia por mesh](#mesh-tenancy)

### Tenencia por namespace {#namespace-tenancy}

Un clúster se puede compartir entre varios equipos, cada uno con un namespace diferente.
Puedes conceder a un equipo permiso para desplegar sus workloads solo en un namespace determinado
o en un conjunto de namespaces.

Por defecto, los servicios de varios namespaces pueden comunicarse entre sí,
pero puedes aumentar el aislamiento eligiendo de forma selectiva qué servicios exponer a otros
namespaces. Puedes configurar políticas de autorización para los servicios expuestos con el fin de restringir
el acceso solo a los clientes autorizados.

{{< image width="50%"
    link="exp-ns.svg"
    alt="Una service mesh con dos namespaces y un servicio expuesto"
    title="Namespaces con un servicio expuesto"
    caption="Una service mesh con dos namespaces y un servicio expuesto"
    >}}

La tenencia por namespace puede extenderse más allá de un único clúster.
Al usar [varios clústeres](#multiple-clusters), los namespaces de cada
clúster que comparten el mismo nombre se consideran, por defecto, el mismo namespace.
Por ejemplo, `Service B` en el namespace `Team-1` del clúster `West` y `Service B` en el
namespace `Team-1` del clúster `East` se refieren al mismo servicio, e Istio combina sus
endpoints para el descubrimiento de servicios y el balanceo de carga.

{{< image width="50%"
    link="cluster-ns.svg"
    alt="Una service mesh con dos clústeres con el mismo namespace"
    title="Namespaces multiclúster"
    caption="Una service mesh con clústeres con el mismo namespace"
    >}}

### Tenencia por clúster {#cluster-tenancy}

Istio permite usar los clústeres como unidad de tenencia. En este caso, puedes dar
a cada equipo un clúster dedicado o un conjunto de clústeres para desplegar sus
workloads. Los permisos de un clúster normalmente se limitan a los miembros del
equipo propietario. Puedes establecer varios roles para un control más detallado, por
ejemplo:

- Administrador del clúster
- Desarrollador

Para usar la tenencia por clúster con Istio, configuras el clúster de cada equipo con su
propio {{< gloss >}}control plane{{< /gloss >}}, lo que permite a cada equipo gestionar su propia configuración.
Como alternativa, puedes usar Istio para implementar un grupo de clústeres como un único inquilino
usando {{< gloss "remote cluster" >}}clústeres remotos{{< /gloss >}} o varios
{{< gloss "primary cluster" >}}clústeres primarios{{< /gloss >}} sincronizados.
Consulta los [modelos de control plane](#control-plane-models) para más detalles.

### Tenencia por mesh {#mesh-tenancy}

En un despliegue multi-mesh con {{< gloss >}}mesh federation{{< /gloss >}}, cada mesh
puede usarse como unidad de aislamiento.

{{< image width="50%"
    link="cluster-iso.svg"
    alt="Dos service meshes aisladas con dos clústeres y dos namespaces"
    title="Aislamiento de clústeres"
    caption="Dos service meshes aisladas con dos clústeres y dos namespaces"
    >}}

Como cada mesh la opera un equipo u organización diferente, la nomenclatura de los servicios
rara vez es distinta. Por ejemplo, un `Service C` en el namespace `foo` del
clúster `Team-1` y el servicio `Service C` en el namespace `foo` del clúster
`Team-2` no se referirán al mismo servicio. El ejemplo más común es el
escenario de Kubernetes en el que muchos equipos usan el namespace `default` para desplegar
sus workloads.

Cuando cada equipo tiene su propia mesh, la comunicación entre meshes sigue los
conceptos descritos en el modelo de [varias meshes](#multiple-meshes).
