---
title: Requisitos de la plataforma
description: Requisitos de la plataforma para Istio.
weight: 1
skip_seealso: true
keywords: [platform-setup,prerequisites]
owner: istio/wg-environments-maintainers
test: no
aliases: docs/setup/platform-setup/prerequisites
---

## Requisitos de módulos del kernel en los nodos del clúster

Independientemente del modo de {{< gloss >}}data plane{{< /gloss >}} de Istio, en contextos de Kubernetes Istio generalmente requiere nodos de Kubernetes que ejecuten kernels de Linux con soporte para la interceptación y el enrutamiento de tráfico. Istio soporta dos backends para la gestión de tráfico: `iptables` (por defecto) y `nftables`.

La mayoría de los kernels de Linux publicados en la última década incluyen soporte integrado para las funcionalidades que usa Istio, ya sea como módulos del kernel que se cargan automáticamente cuando se necesitan o integrados en el propio kernel. Los módulos del kernel específicos que se requieren dependen del backend que elijas usar.

### Backend de iptables

Al usar el backend `iptables` (el backend por defecto), se requieren los siguientes módulos del kernel para que Istio funcione correctamente:

#### Módulos principales de iptables

| Módulo | Observación |
| --- | --- |
| `br_netfilter` |  |
| `ip6table_mangle` | Solo necesario para clústeres IPv6/dual-stack |
| `ip6table_nat` | Solo necesario para clústeres IPv6/dual-stack |
| `ip6table_raw` | Solo necesario para clústeres IPv6/dual-stack |
| `iptable_mangle` |  |
| `iptable_nat` |  |
| `iptable_raw` | Solo necesario para la interceptación de `DNS` en modo sidecar |
| `xt_REDIRECT` |  |
| `xt_connmark` | Necesario para el modo de data plane ambient y para el modo de data plane sidecar con interceptación `TPROXY` (por defecto) |
| `xt_conntrack` |  |
| `xt_CT` | Necesario para la interceptación de `DNS` con el backend `iptables` |
| `xt_mark` | Necesario para el modo de data plane ambient y para el modo de data plane sidecar con interceptación `TPROXY` (por defecto) |
| `xt_owner` |  |
| `xt_tcpudp` |  |
| `xt_multiport` |  |
| `ip_set` | Necesario para el modo de data plane ambient |

Los módulos listados arriba usan los siguientes módulos adicionales, que también deben cargarse en el nodo del clúster:

| Módulo | Observación |
| --- | --- |
| `bridge` |  |
| `ip6_tables` | Solo necesario para clústeres IPv6/dual-stack |
| `ip_tables` |  |
| `nf_conntrack` |  |
| `nf_conntrack_ipv4` |  |
| `nf_conntrack_ipv6` | Solo necesario para clústeres IPv6/dual-stack |
| `nf_nat` |  |
| `nf_nat_ipv4` |  |
| `nf_nat_ipv6` | Solo necesario para clústeres IPv6/dual-stack |
| `nf_nat_redirect` |  |
| `x_tables` |  |
| `ip_set_hash_ip` | Necesario para el modo de data plane ambient |

### Backend de nftables

El framework `nftables` es un reemplazo moderno de `iptables` que ofrece mejor rendimiento y flexibilidad.
Istio usa la herramienta CLI `nft` para configurar las reglas de `nftables`. El binario `nft` debe ser de la versión 1.0.1 o posterior, y
requiere la versión 5.13 o superior del kernel de Linux. Para que la CLI `nft` funcione correctamente, los siguientes módulos del kernel deben
estar disponibles en el sistema host.

| Módulo           | Observación                              |
|------------------|------------------------------------------|
| `nf_tables`      | Módulo principal de nftables             |
| `nf_conntrack`   | Necesario para el seguimiento de conexiones |
| `nft_ct`         |                                          |
| `nf_defrag_ipv4` |                                          |
| `nf_defrag_ipv6` | Solo necesario para clústeres IPv6/dual-stack |
| `nft_nat`        |                                          |
| `nft_socket`     |                                          |
| `nft_tproxy`     |                                          |
| `nft_redir`      |                                          |

### Problemas con los módulos del kernel

Aunque es poco común, el uso de kernels de Linux o distribuciones de Linux personalizados o no estándar puede dar lugar a escenarios en los que los módulos específicos listados arriba no estén disponibles en el host o no se puedan cargar automáticamente. Por ejemplo, el artículo [`selinux issue`](https://www.suse.com/support/kb/doc/?id=000020241) describe un escenario en algunas versiones de RHEL en el que la configuración de `selinux` puede impedir la carga automática de algunos de los módulos del kernel mencionados arriba.

Para más detalles sobre los componentes específicos de Istio que realizan la interceptación de tráfico y la configuración del enrutamiento, consulta la documentación del modo de data plane correspondiente.
