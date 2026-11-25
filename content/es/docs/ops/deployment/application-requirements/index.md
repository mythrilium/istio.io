---
title: Requisitos de las aplicaciones
description: Requisitos de las aplicaciones desplegadas en un clúster con Istio habilitado.
weight: 40
keywords:
  - kubernetes
  - sidecar
  - sidecar-injection
  - deployment-models
  - pods
  - setup
aliases:
  - /docs/setup/kubernetes/spec-requirements/
  - /docs/setup/kubernetes/prepare/spec-requirements/
  - /docs/setup/kubernetes/prepare/requirements/
  - /docs/setup/kubernetes/additional-setup/requirements/
  - /docs/setup/additional-setup/requirements
  - /docs/ops/setup/required-pod-capabilities
  - /help/ops/setup/required-pod-capabilities
  - /docs/ops/prep/requirements
  - /docs/ops/deployment/requirements
owner: istio/wg-environments-maintainers
test: n/a
---

Istio proporciona una gran cantidad de funcionalidades a las aplicaciones con poco o ningún impacto en el propio código de la aplicación.
Muchas aplicaciones de Kubernetes se pueden desplegar en un clúster con Istio habilitado sin ningún cambio.
Sin embargo, el modelo sidecar de Istio tiene algunas implicaciones que pueden requerir una consideración especial al desplegar
una aplicación con Istio habilitado.
Este documento describe estas consideraciones para las aplicaciones y los requisitos específicos para habilitar Istio.

## Requisitos de los pods {#pod-requirements}

Para formar parte de una mesh, los pods de Kubernetes deben cumplir los siguientes requisitos:

- **UIDs de la aplicación**: Asegúrate de que tus pods **no** ejecuten aplicaciones como un usuario
  con el valor de ID de usuario (UID) `1337`, porque `1337` está reservado para el proxy sidecar.

- **Capacidades `NET_ADMIN` y `NET_RAW`**: A menos que uses el
    [plugin CNI de Istio](/es/docs/setup/additional-setup/cni/), el contenedor `istio-init` requiere
    las capacidades `NET_ADMIN` y `NET_RAW` para configurar la redirección de tráfico con iptables. El
    namespace debe usar el nivel `privileged` en el modo enforce de
    [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/).
    Tanto el nivel `baseline` como el `restricted` bloquean estas capacidades e
    impedirán que el contenedor `istio-init` se ejecute.

    Para comprobar la etiqueta `pod-security.kubernetes.io/enforce` de un namespace:

    {{< text bash >}}
    $ kubectl get namespace <your namespace> --show-labels
    NAME       STATUS   AGE   LABELS
    myapp      Active   3d    pod-security.kubernetes.io/enforce=privileged,...
    {{< /text >}}

    Para aplicar el nivel `privileged` en modo enforce al namespace:

    {{< text bash >}}
    $ kubectl label namespace <your namespace> pod-security.kubernetes.io/enforce=privileged --overwrite
    {{< /text >}}

    Si tu política de seguridad no permite aplicar el nivel `privileged` al namespace, usa en su lugar el
    [plugin CNI de Istio](/es/docs/setup/additional-setup/cni/), que gestiona la redirección
    del tráfico sin requerir capacidades elevadas en el pod.

- **Etiquetas de pod**: Recomendamos declarar explícitamente en los pods un identificador y una versión de la aplicación mediante una etiqueta de pod.
  Estas etiquetas añaden información de contexto a las métricas y la telemetría que recopila Istio.
  Cada uno de estos valores se lee de varias etiquetas, ordenadas de mayor a menor precedencia:

    - Nombre de la aplicación: `service.istio.io/canonical-name`, `app.kubernetes.io/name` o `app`.
    - Versión de la aplicación: `service.istio.io/canonical-revision`, `app.kubernetes.io/version` o `version`.

- **Puertos de servicio con nombre**: Opcionalmente, los puertos de servicio pueden tener un nombre para especificar explícitamente un protocolo.
  Consulta [Selección de protocolo](/es/docs/ops/configuration/traffic-management/protocol-selection/) para
  más detalles. Si un pod pertenece a varios [servicios de Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service/),
  esos servicios no pueden usar el mismo número de puerto para protocolos distintos, por
  ejemplo, HTTP y TCP.

## Puertos usados por Istio {#ports-used-by-istio}

El proxy sidecar de Istio (Envoy) usa los siguientes puertos y protocolos.

{{< warning >}}
Para evitar conflictos de puertos con los sidecars, las aplicaciones no deben usar ninguno de los puertos que usa Envoy.
{{< /warning >}}

| Puerto | Protocolo | Descripción | Solo interno del pod |
|----|----|----|----|
| 15000 | TCP | Puerto de administración de Envoy (comandos/diagnóstico) | Sí |
| 15001 | TCP | Tráfico saliente de Envoy | No |
| 15002 | TCP | Puerto de escucha para la detección de fallos | Sí |
| 15004 | HTTP | Puerto de depuración | Sí |
| 15006 | TCP | Tráfico entrante de Envoy | No |
| 15008 | HTTP2 | Puerto del túnel mTLS {{< gloss >}}HBONE{{</ gloss >}} | No |
| 15020 | HTTP | Telemetría de Prometheus combinada del agente de Istio, Envoy y la aplicación | No |
| 15021 | HTTP | Comprobaciones de estado (health checks) | No |
| 15053 | DNS  | Puerto DNS, si la captura está habilitada | Sí |
| 15090 | HTTP | Telemetría de Prometheus de Envoy | No |

El control plane de Istio (istiod) usa los siguientes puertos y protocolos.

| Puerto | Protocolo | Descripción | Solo host local |
|----|----|----|----|
| 443 | HTTPS | Puerto del servicio de webhooks | No |
| 8080 | HTTP | Interfaz de depuración (obsoleta, solo puerto del contenedor) | No |
| 15010 | GRPC | Servicios XDS y de CA (texto plano, solo para redes seguras) | No |
| 15012 | GRPC | Servicios XDS y de CA (TLS y mTLS, recomendado para uso en producción) | No |
| 15014 | HTTP | Monitoreo del control plane | No |
| 15017 | HTTPS | Puerto del contenedor del webhook, reenviado desde el 443 | No |

## Protocolos en los que el servidor habla primero (server-first) {#server-first-protocols}

Algunos protocolos son protocolos "server-first", lo que significa que el servidor envía los primeros bytes. Esto puede afectar al
mTLS [`PERMISSIVE`](/es/docs/reference/config/security/peer_authentication/#PeerAuthentication-MutualTLS-Mode) y a la [selección automática de protocolo](/es/docs/ops/configuration/traffic-management/protocol-selection/#automatic-protocol-selection).

Ambas funcionalidades inspeccionan los bytes iniciales de una conexión para determinar el protocolo, lo cual es incompatible con los protocolos server-first.

Para soportar estos casos, sigue los pasos de [selección explícita de protocolo](/es/docs/ops/configuration/traffic-management/protocol-selection/#explicit-protocol-selection) para declarar el protocolo de la aplicación como `TCP`.

Se sabe que los siguientes puertos suelen transportar protocolos server-first, y se asume automáticamente que son `TCP`:

|Protocolo|Puerto|
|--------|----|
| SMTP   |25  |
| DNS    |53  |
| MySQL  |3306|
| MongoDB|27017|

Como la comunicación TLS no es server-first, el tráfico server-first cifrado con TLS funcionará con la detección automática de protocolo siempre que te asegures de que todo el tráfico sujeto al sniffing de TLS esté cifrado:

1. Configura el modo `mTLS` `STRICT` para el servidor. Esto aplicará el cifrado TLS a todas las solicitudes.
1. Configura el modo `mTLS` `DISABLE` para el servidor. Esto deshabilitará el sniffing de TLS, lo que permite usar protocolos server-first.
1. Configura todos los clientes para que envíen tráfico `TLS`, generalmente mediante una [`DestinationRule`](/es/docs/reference/config/networking/destination-rule/#ClientTLSSettings) o usando auto mTLS.
1. Configura tu aplicación para que envíe tráfico TLS directamente.

## Tráfico saliente

Para soportar las capacidades de enrutamiento de tráfico de Istio, el tráfico que sale de un pod puede enrutarse de forma diferente
a cuando no hay un sidecar desplegado.

Para el tráfico basado en HTTP, el tráfico se enruta según el encabezado `Host`. Esto puede provocar un comportamiento inesperado si la IP de destino
y el encabezado `Host` no coinciden. Por ejemplo, una solicitud como `curl 1.2.3.4 -H "Host: httpbin.default"` se enrutará al servicio `httpbin`,
en lugar de a `1.2.3.4`.

Para el tráfico no basado en HTTP (incluido HTTPS), Istio no tiene acceso a un encabezado `Host`, por lo que las decisiones de enrutamiento se basan en la dirección IP del Service.

Una implicación de esto es que las llamadas directas a pods (por ejemplo, `curl <POD_IP>`), en lugar de a Services, no coincidirán con ningún Service. Aunque el tráfico puede
[dejarse pasar (passthrough)](/es/docs/tasks/traffic-management/egress/egress-control/#envoy-passthrough-to-external-services), no obtendrá toda la funcionalidad de Istio,
incluido el cifrado mTLS, el enrutamiento de tráfico y la telemetría.

Consulta la página de [Enrutamiento de tráfico](/es/docs/ops/configuration/traffic-management/traffic-routing) para más información.
