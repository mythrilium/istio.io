---
title: Arquitectura de máquinas virtuales
description: Describe la arquitectura general de Istio para máquinas virtuales.
weight: 25
keywords:
- virtual-machine
test: n/a
owner: istio/wg-environments-maintainers
---

Antes de leer este documento, asegúrate de revisar la [arquitectura de Istio](/es/docs/ops/deployment/architecture/) y los [modelos de despliegue](/es/docs/ops/deployment/deployment-models/).
Esta página se basa en esos documentos para explicar cómo se puede extender Istio para incorporar máquinas virtuales a la mesh.

El soporte de Istio para máquinas virtuales permite conectar a la mesh workloads que se ejecutan fuera de un clúster de Kubernetes.
Esto permite que las aplicaciones heredadas, o las aplicaciones que no son adecuadas para ejecutarse en un entorno de contenedores, obtengan todos los beneficios que Istio proporciona a las aplicaciones que se ejecutan dentro de Kubernetes.

Para los workloads que se ejecutan en Kubernetes, la propia plataforma Kubernetes proporciona varias funcionalidades, como el descubrimiento de servicios, la resolución DNS y las comprobaciones de estado, que a menudo faltan en los entornos de máquinas virtuales.
Istio habilita estas funcionalidades para los workloads que se ejecutan en máquinas virtuales y, además, permite que estos workloads utilicen funcionalidades de Istio como TLS mutuo (mTLS), telemetría detallada y capacidades avanzadas de gestión de tráfico.

El siguiente diagrama muestra la arquitectura de una mesh con máquinas virtuales:

{{< tabset category-name="network-mode" >}}

{{< tab name="Single-Network" category-value="single" >}}

En esta mesh hay una única [red](/es/docs/ops/deployment/deployment-models/#network-models), en la que los pods y las máquinas virtuales pueden comunicarse directamente entre sí.

El tráfico del control plane, incluida la configuración XDS y la firma de certificados, se envía a través de un Gateway en el clúster.
Esto garantiza que las máquinas virtuales tengan una dirección estable a la que conectarse durante su arranque (bootstrapping). Los pods y las máquinas virtuales pueden comunicarse directamente entre sí sin necesidad de ningún Gateway intermedio.

{{< image width="75%"
    link="single-network.svg"
    alt="Una service mesh con una única red y máquinas virtuales"
    title="Red única"
    caption="Una service mesh con una única red y máquinas virtuales"
    >}}

{{< /tab >}}

{{< tab name="Multi-Network" category-value="multiple" >}}

En esta mesh hay varias [redes](/es/docs/ops/deployment/deployment-models/#network-models), en las que los pods y las máquinas virtuales no pueden comunicarse directamente entre sí.

El tráfico del control plane, incluida la configuración XDS y la firma de certificados, se envía a través de un Gateway en el clúster.
Del mismo modo, toda la comunicación entre pods y máquinas virtuales pasa por el gateway, que actúa como puente entre las dos redes.

{{< image width="75%"
    link="multi-network.svg"
    alt="Una service mesh con varias redes y máquinas virtuales"
    title="Varias redes"
    caption="Una service mesh con varias redes y máquinas virtuales"
    >}}

{{< /tab >}}

{{< /tabset >}}

## Asociación de servicios

Istio proporciona dos mecanismos para representar los workloads de máquinas virtuales:

* [`WorkloadGroup`](/es/docs/reference/config/networking/workload-group/) representa un grupo lógico de workloads de máquinas virtuales que comparten propiedades comunes. Es similar a un `Deployment` en Kubernetes.
* [`WorkloadEntry`](/es/docs/reference/config/networking/workload-entry/) representa una única instancia de un workload de máquina virtual. Es similar a un `Pod` en Kubernetes.

Crear estos recursos (`WorkloadGroup` y `WorkloadEntry`) no aprovisiona ningún recurso ni ejecuta ningún workload de máquina virtual.
En cambio, estos recursos simplemente hacen referencia a esos workloads e indican a Istio cómo configurar la mesh adecuadamente.

Al añadir un workload de máquina virtual a la mesh, necesitarás crear un `WorkloadGroup` que actúe como plantilla para cada instancia de `WorkloadEntry`:

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: WorkloadGroup
metadata:
  name: product-vm
spec:
  metadata:
    labels:
      app: product
  template:
    serviceAccount: default
  probe:
    httpGet:
      port: 8080
{{< /text >}}

Una vez que una máquina virtual se ha [configurado y añadido a la mesh](/es/docs/setup/install/virtual-machine/#configure-the-virtual-machine), el control plane de Istio creará automáticamente el `WorkloadEntry` correspondiente.
Por ejemplo:

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: WorkloadEntry
metadata:
  annotations:
    istio.io/autoRegistrationGroup: product-vm
  labels:
    app: product
  name: product-vm-1.2.3.4
spec:
  address: 1.2.3.4
  labels:
    app: product
  serviceAccount: default
{{< /text >}}

Este recurso `WorkloadEntry` describe una única instancia de un workload, similar a un pod en Kubernetes. Cuando el workload se elimina de la mesh, el recurso `WorkloadEntry`
se elimina automáticamente.  Además, si hay sondas (probes) configuradas en el recurso `WorkloadGroup`, el control plane de Istio actualiza automáticamente el estado de salud de las instancias de `WorkloadEntry` asociadas.

Para que los consumidores puedan llamar a tu workload de forma fiable, se recomienda declarar una asociación con un `Service`. Esto permite a los clientes acceder a un nombre de host estable, como `product.default.svc.cluster.local`, en lugar de a una dirección IP efímera. Esto también te permite usar las capacidades avanzadas de enrutamiento de Istio mediante las APIs `DestinationRule` y `VirtualService`.

Cualquier servicio de Kubernetes puede seleccionar de forma transparente workloads tanto de pods como de máquinas virtuales mediante los campos selector, que se comparan con las etiquetas de los pods y de los `WorkloadEntry`, respectivamente.

Por ejemplo, un `Service` llamado `product` está compuesto por un `Pod` y un `WorkloadEntry`:

{{< image width="30%"
    link="service-selector.svg"
    title="Selección de servicios"
    >}}

Con esta configuración, las solicitudes a `product` se balancearían entre las instancias de workload del pod y de la máquina virtual.

## DNS

Kubernetes proporciona resolución DNS en los pods para los nombres de `Service`, lo que permite que los pods se comuniquen fácilmente entre sí mediante nombres de host estables.

Para la expansión a máquinas virtuales, Istio proporciona una funcionalidad similar mediante un [proxy DNS](/es/docs/ops/configuration/traffic-management/dns-proxy/).
Esta funcionalidad redirige todas las consultas DNS del workload de la máquina virtual al proxy de Istio, que mantiene una correspondencia entre nombres de host y direcciones IP.

Como resultado, los workloads que se ejecutan en máquinas virtuales pueden llamar de forma transparente a los `Service`s (de forma similar a los pods) sin necesidad de configuración adicional.
