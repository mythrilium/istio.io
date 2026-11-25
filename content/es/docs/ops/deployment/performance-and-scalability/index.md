---
title: Rendimiento y escalabilidad
description: Resumen del rendimiento y la escalabilidad de Istio.
weight: 30
keywords:
  - performance
  - scalability
  - scale
  - benchmarks
aliases:
  - /docs/performance-and-scalability/overview
  - /docs/performance-and-scalability/microbenchmarks
  - /docs/performance-and-scalability/performance-testing-automation
  - /docs/performance-and-scalability/realistic-app-benchmark
  - /docs/performance-and-scalability/scalability
  - /docs/performance-and-scalability/scenarios
  - /docs/performance-and-scalability/synthetic-benchmarks
  - /docs/concepts/performance-and-scalability
  - /docs/ops/performance-and-scalability
owner: istio/wg-environments-maintainers
test: n/a
---

Istio facilita la creación de una red de servicios desplegados con enrutamiento avanzado,
balanceo de carga, autenticación de servicio a servicio, monitoreo y más, todo
sin ningún cambio en el código de la aplicación. Istio se esfuerza por ofrecer
estos beneficios con una sobrecarga mínima de recursos y aspira a soportar meshes muy
grandes con altas tasas de solicitudes, añadiendo una latencia mínima.

Los componentes del data plane de Istio, los proxies Envoy, gestionan los datos que fluyen por
el sistema. El componente del control plane de Istio, Istiod, configura
el data plane. El data plane y el control plane tienen consideraciones de rendimiento distintas.

## Resumen de rendimiento de Istio 1.24

La mesh de las [pruebas de carga de Istio](https://github.com/istio/tools/tree/{{< source_branch_name >}}/perf/load) consta
de **1000** servicios y **2000** pods en una mesh de Istio con 70 000 solicitudes por segundo en toda la mesh.

## Rendimiento del control plane

Istiod configura los proxies sidecar en función de los archivos de configuración creados por el usuario y del estado
actual del sistema. En un entorno de Kubernetes, las Custom Resource Definitions (CRDs) y los deployments
constituyen la configuración y el estado del sistema. Los objetos de configuración de Istio, como los gateways y los virtual
services, proporcionan la configuración creada por el usuario.
Para generar la configuración de los proxies, Istiod procesa la configuración y el estado del sistema combinados
del entorno de Kubernetes y la configuración creada por el usuario.

El control plane soporta miles de servicios, distribuidos en miles de pods, con un
número similar de virtual services creados por el usuario y otros objetos de configuración.
Los requisitos de CPU y memoria de Istiod escalan con la cantidad de configuraciones y de posibles estados del sistema.
El consumo de CPU escala con los siguientes factores:

- La tasa de cambios en los despliegues.
- La tasa de cambios de configuración.
- El número de proxies que se conectan a Istiod.

Sin embargo, esta parte es inherentemente escalable de forma horizontal.

Puedes aumentar el número de instancias de Istiod para reducir el tiempo que tarda la configuración
en llegar a todos los proxies.

A gran escala, se recomienda encarecidamente [limitar el ámbito de la configuración](/es/docs/ops/configuration/mesh/configuration-scoping).

## Rendimiento del data plane

El rendimiento del data plane depende de muchos factores, por ejemplo:

- Número de conexiones de cliente
- Tasa de solicitudes objetivo
- Tamaño de la solicitud y tamaño de la respuesta
- Número de hilos de trabajo (worker threads) del proxy
- Protocolo
- Núcleos de CPU
- Diversas funcionalidades del proxy. En particular, se sabe que los filtros de telemetría (registro, trazado y métricas) tienen un impacto moderado.

La latencia, el throughput y el consumo de CPU y memoria de los proxies se miden en función de dichos factores.

### Uso de recursos del sidecar y de ztunnel

Como el proxy sidecar realiza trabajo adicional en la ruta de datos, consume CPU
y memoria. En Istio 1.24, con 1000 solicitudes http por segundo con 1 KB de carga útil cada una
- un único proxy sidecar con 2 hilos de trabajo consume unos 0.20 vCPU y 60 MB de memoria.
- un único waypoint proxy con 2 hilos de trabajo consume unos 0.25 vCPU y 60 MB de memoria
- un único proxy ztunnel consume unos 0.06 vCPU y 12 MB de memoria.

El consumo de memoria del proxy depende del estado de configuración total que mantiene el proxy.
Un gran número de listeners, clusters y rutas puede aumentar el uso de memoria.

### Latencia

Como Istio añade un proxy sidecar o un proxy ztunnel en la ruta de datos, la latencia es una consideración
importante.
Cada funcionalidad que añade Istio también aumenta la longitud de la ruta dentro del proxy y puede afectar a la latencia.

El proxy Envoy recopila datos de telemetría sin procesar después de enviar una respuesta al
cliente.
El tiempo dedicado a recopilar la telemetría sin procesar de una solicitud no contribuye al tiempo total necesario para completar esa solicitud.
Sin embargo, como el hilo de trabajo está ocupado gestionando la solicitud, no empezará a gestionar la siguiente solicitud inmediatamente.
Este proceso aumenta el tiempo de espera en cola de la siguiente solicitud y afecta a las latencias media y de cola (tail).
La latencia de cola real depende del patrón de tráfico.

### Latencia de Istio 1.24

En modo sidecar, una solicitud pasa por el proxy sidecar del cliente y luego por el proxy sidecar del servidor antes de llegar al servidor, y viceversa.
En modo ambient, una solicitud pasa por el ztunnel del nodo cliente y luego por el ztunnel del nodo servidor antes de llegar al servidor.
Con waypoints configurados, una solicitud pasa por un waypoint proxy entre los ztunnels.
Los siguientes gráficos muestran la latencia P90 y P99 de las solicitudes http/1.1 que atraviesan distintos modos de data plane.
Para ejecutar las pruebas, usamos un clúster bare-metal de 5 máquinas [M3 Large](https://deploy.equinix.com/product/servers/m3-large/) y [Flannel](https://github.com/flannel-io/flannel) como CNI principal.
Obtuvimos estos resultados usando los [benchmarks de Istio](https://github.com/istio/tools/tree/{{< source_branch_name >}}/perf/benchmark) para el protocolo `http/1.1` con una carga útil de 1 KB a 500, 750, 1000, 1250 y 1500 solicitudes por segundo, usando 4 conexiones de cliente, 2 hilos de trabajo del proxy y TLS mutuo habilitado.

Nota: Estas pruebas se realizaron en el [CNCF Community Infrastructure Lab](https://github.com/cncf/cluster).
Un hardware diferente dará valores diferentes.

{{< image link="./istio-1.24.0-fortio-90.png" caption="Latencia P90 frente a conexiones de cliente" width="90%" >}}

{{< image link="./istio-1.24.0-fortio-99.png" caption="Latencia P99 frente a conexiones de cliente" width="90%" >}}

- `no mesh`: El pod cliente llama directamente al pod servidor, sin pods en la service mesh de Istio.
- `ambient: L4`: Modo ambient por defecto con el {{< gloss >}}secure L4 overlay{{< /gloss >}}
- `ambient: L4+L7` Modo ambient por defecto con el secure L4 overlay y waypoints habilitados para el namespace.
- `sidecar` Sidecars del cliente y del servidor.

### Herramientas de benchmarking

Istio usa las siguientes herramientas para las pruebas de rendimiento (benchmarking)

- [`fortio.org`](https://fortio.org/) - una herramienta de pruebas de carga con throughput constante.
- [`nighthawk`](https://github.com/envoyproxy/nighthawk) - una herramienta de pruebas de carga basada en Envoy.
- [`isotope`](https://github.com/istio/tools/tree/{{< source_branch_name >}}/isotope) - una aplicación sintética con topología configurable.
