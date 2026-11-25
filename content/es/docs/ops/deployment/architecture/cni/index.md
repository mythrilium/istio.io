---
title: Plugin CNI
description: Describe cómo funciona el plugin CNI de Istio.
weight: 10
owner: istio/wg-networking-maintainers
test: n/a
---

Kubernetes tiene un modelo de red único y permisivo. Para configurar la red L2-L4 entre Pods, [un clúster de Kubernetes requiere un plugin CNI (Container Network Interface) de _interfaz_](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/). Este plugin se ejecuta cada vez que se crea un nuevo pod y configura el entorno de red de ese pod.

Si usas un proveedor de Kubernetes gestionado, normalmente tienes pocas opciones sobre qué plugin CNI se usa en tu clúster: es un detalle de implementación del servicio gestionado.

Para configurar la redirección del tráfico de la mesh, independientemente del CNI que tú o tu proveedor elijan usar para la red L2-L4, Istio incluye un plugin CNI _encadenado_ (chained), que se ejecuta después de todos los plugins CNI de interfaz configurados. La API para definir plugins encadenados y de interfaz, y para compartir datos entre ellos, forma parte de la [especificación CNI](https://www.cni.dev/). Istio funciona con todas las implementaciones de CNI que siguen el estándar CNI, tanto en modo sidecar como en modo ambient.

El plugin CNI de Istio es opcional en modo sidecar y obligatorio en modo {{<gloss>}}ambient{{< /gloss >}}.

* [Aprende a instalar Istio con un plugin CNI](/es/docs/setup/additional-setup/cni/)
