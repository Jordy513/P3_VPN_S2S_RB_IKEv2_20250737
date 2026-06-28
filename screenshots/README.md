# Capturas de pantalla — IPSec VPN Basado en Enrutamiento (IKEv2)

Capturas del laboratorio en orden de demostración.

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](/screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, todos los dispositivos encendidos. |
| 2 | [`02_ping_wan_previo.png`](/screenshots/02_ping_wan_previo.png) | Ping desde R1 (`192.168.1.10`) hacia R2 (`192.168.1.20`) confirmando conectividad WAN base antes de configurar el túnel. |
| 3 | [`03_config_r1_ikev2.png`](/screenshots/03_config_r1_ikev2.png) | Consola de R1 mostrando la configuración de `crypto ikev2 proposal`, `crypto ikev2 policy` y `crypto ikev2 keyring`. |
| 4 | [`04_config_r1_profile_vti.png`](/screenshots/04_config_r1_profile_vti.png) | Consola de R1 mostrando el `crypto ikev2 profile`, el `crypto ipsec profile` con `set ikev2-profile` y la interfaz `Tunnel0` con `tunnel protection`. |
| 5 | [`05_config_r1_ospf.png`](/screenshots/05_config_r1_ospf.png) | Consola de R1 mostrando la configuración de OSPF con las redes LAN y la subred del túnel en el Área 0. |
| 6 | [`06_config_r2_completa.png`](/screenshots/06_config_r2_completa.png) | Configuración completa de R2 (Site B) mostrando la simetría con R1: keyring apuntando a `192.168.1.10`, VTI con `tunnel destination 192.168.1.10`. |
| 7 | [`07_tunnel0_up.png`](/screenshots/07_tunnel0_up.png) | Salida de `show interface Tunnel0` en R1 mostrando estado `up/up` y `Tunnel protection via IPSec (profile "IPSEC_PROFILE_VTI")`. |
| 8 | [`08_ikev2_sa_ready.png`](/screenshots/08_ikev2_sa_ready.png) | Salida de `show crypto ikev2 sa` mostrando estado `READY` con parámetros AES-256/SHA256/DH14 después de disparar la negociación con ping extendido. |
| 9 | [`09_ospf_neighbor_full.png`](/screenshots/09_ospf_neighbor_full.png) | Salida de `show ip ospf neighbor` mostrando adyacencia en estado `FULL` sobre `Tunnel0` — prueba del enrutamiento dinámico sobre IKEv2. |
| 10 | [`10_ping_extremo_a_extremo.png`](/screenshots/10_ping_extremo_a_extremo.png) | Ping exitoso desde PC1 (`20.25.37.130`) hacia PC4 (`20.25.37.3`) con el túnel IKEv2/IPSec activo, TTL=62. |
