# IPSec VPN — Basado en Enrutamiento (IKEv2 / VTI)

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Marco Teórico](#2-marco-teórico)
   - [IKEv2 con VTI — La combinación moderna](#21-ikev2-con-vti--la-combinación-moderna)
   - [Virtual Tunnel Interface con IKEv2](#22-virtual-tunnel-interface-con-ikev2)
   - [Comparativa de los cuatro modelos VPN vistos](#23-comparativa-de-los-cuatro-modelos-vpn-vistos)
   - [Parámetros Criptográficos Utilizados](#24-parámetros-criptográficos-utilizados)
3. [Documentación de la Red](#3-documentación-de-la-red)
   - [Topología](#31-topología)
   - [Tabla de Dispositivos y Direccionamiento IP](#32-tabla-de-dispositivos-y-direccionamiento-ip)
4. [Scripts de Configuración](#4-scripts-de-configuración)
   - [Router R1 — Site A](#41-router-r1--site-a)
   - [Router R2 — Site B](#42-router-r2--site-b)
   - [Configuración de PCs (Hosts)](#43-configuración-de-pcs-hosts)
5. [Verificación del Túnel](#5-verificación-del-túnel)
6. [Capturas de Pantalla](#6-capturas-de-pantalla)
7. [Consideraciones de Seguridad](#7-consideraciones-de-seguridad)
8. [Video Demostrativo](#8-video-demostrativo)
9. [Referencias](#9-referencias)

---

## 1. Objetivo del Laboratorio

El objetivo de este laboratorio es **implementar y verificar una VPN Site-to-Site Point-to-Point basada en enrutamiento utilizando IKEv2 con Virtual Tunnel Interface (VTI)** sobre infraestructura Cisco IOS. A través de esta práctica se busca demostrar:

* La configuración del protocolo IKEv2 mediante `crypto ikev2 proposal`, `crypto ikev2 policy`, `crypto ikev2 keyring` y `crypto ikev2 profile`, vinculado directamente a una interfaz de túnel lógica (`Tunnel0`) mediante `tunnel protection ipsec profile` — sin uso de Crypto Maps ni ACLs de tráfico interesante.
* La combinación de las ventajas del modelo *route-based* (enrutamiento dinámico sobre el túnel, interfaz visible, una sola SA por túnel) con las ventajas de IKEv2 (negociación en 4 mensajes, Dead Peer Detection integrado, mayor resistencia a DoS).
* La propagación automática de rutas entre las subredes LAN privadas (`20.25.37.128/25` y `20.25.37.0/25`) mediante OSPF corriendo directamente sobre la VTI protegida por IKEv2.
* La diferencia de configuración entre este modelo y los tres anteriores (IKEv1 policy-based, IKEv1 route-based, IKEv2 policy-based), cerrando el ciclo comparativo de los cuatro modelos VPN Site-to-Site.

---

## 2. Marco Teórico

### 2.1 IKEv2 con VTI

Los cuatro laboratorios de VPN Site-to-Site cubiertos en este curso representan una progresión lógica:

```
IKEv1 + Policy-Based  →  IKEv1 + Route-Based  →  IKEv2 + Policy-Based  →  IKEv2 + Route-Based
  (legacy, ACL)             (VTI, OSPF)            (moderno, ACL)           (moderno + VTI)
                                                                                 ↑ Este lab
```

La combinación **IKEv2 + VTI** representa el modelo recomendado por Cisco para nuevas implementaciones de VPN Site-to-Site. Es la arquitectura base de **FlexVPN** — la solución VPN empresarial de Cisco que unifica acceso remoto y site-to-site en un único framework. También es el modelo que implementan AWS VPN Gateway, Azure VPN Gateway y Google Cloud VPN cuando se conectan a equipos Cisco.

Las razones para elegir IKEv2+VTI sobre las otras combinaciones:

* Frente a IKEv1+Policy-Based: IKEv2 negocia más rápido (4 vs 9 mensajes), no tiene Aggressive Mode inseguro, y la VTI elimina las ACLs propensas a error.
* Frente a IKEv1+Route-Based: misma arquitectura VTI pero con el protocolo de negociación moderno — Dead Peer Detection integrado, soporte EAP, MOBIKE.
* Frente a IKEv2+Policy-Based: la VTI permite enrutamiento dinámico (OSPF/EIGRP/BGP) directamente sobre el túnel, imposible con Crypto Map.

### 2.2 Virtual Tunnel Interface con IKEv2

En el modelo route-based con IKEv1, la VTI se protegía así:

```cisco
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256    ! IKEv1 transform set

interface Tunnel0
 tunnel protection ipsec profile IPSEC_PROFILE_VTI
```

Con IKEv2, el IPSec Profile referencia adicionalmente el `ikev2-profile`:

```cisco
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256
 set ikev2-profile PROF_IKEv2          ! ← única línea nueva vs IKEv1 route-based

interface Tunnel0
 tunnel protection ipsec profile IPSEC_PROFILE_VTI
```

Esta es la diferencia de configuración entre IKEv1 route-based e IKEv2 route-based: **una sola línea** en el IPSec Profile. El resto de la configuración de la VTI — `tunnel source`, `tunnel destination`, `tunnel mode`, MTU, OSPF — es idéntico.

La diferencia principal vs. IKEv2 policy-based es que aquí **no hay Crypto Map** — el IPSec Profile se aplica directamente sobre `Tunnel0`, y el enrutamiento (OSPF) decide qué tráfico entra al túnel.

### 2.3 Comparativa de los cuatro modelos VPN vistos

| Característica | IKEv1 Policy | IKEv1 Route (VTI) | IKEv2 Policy | IKEv2 Route (VTI) — Este lab |
|---|---|---|---|---|
| **Protocolo IKE** | IKEv1 | IKEv1 | IKEv2 | IKEv2 |
| **Selección de tráfico** | ACL (Crypto Map) | Tabla de rutas | ACL (Crypto Map) | Tabla de rutas |
| **Interfaz de túnel** | No (WAN directa) | Tunnel0 (VTI) | No (WAN directa) | Tunnel0 (VTI) |
| **Routing dinámico** | ❌ | ✅ OSPF/EIGRP | ❌ | ✅ OSPF/EIGRP |
| **Mensajes negociación** | 9 | 9 | 4 | 4 |
| **DPD integrado** | ❌ | ❌ | ✅ | ✅ |
| **Comando clave** | `crypto map` en e0/0 | `tunnel protection` en Tunnel0 | `crypto map` + `set ikev2-profile` | `tunnel protection` + `set ikev2-profile` |
| **Recomendado** | Legacy | Aceptable | Moderno | ✅ Estándar actual |

### 2.4 Parámetros Criptográficos Utilizados

| Parámetro | Valor configurado | Propósito |
|---|---|---|
| **Cifrado IKE SA** | AES-256-CBC | Cifrado del canal IKE |
| **Integridad IKE SA** | SHA-256 | Integridad de mensajes IKE |
| **Grupo Diffie-Hellman** | Grupo 14 (2048-bit MODP) | Intercambio seguro de material de clave |
| **Autenticación** | Pre-Shared Key (PSK) | Autenticación mutua de los peers |
| **Lifetime IKE SA** | 86400 segundos (24 horas) | Duración del canal IKE antes de renegociar |
| **Cifrado ESP (Child SA)** | AES-256 | Cifrado del tráfico de datos |
| **Autenticación ESP (Child SA)** | SHA-256 HMAC | Integridad del tráfico de datos |
| **Modo IPSec** | Tunnel | Encapsulación completa del paquete IP original |
| **Lifetime Child SA** | 3600 segundos (1 hora) | Duración del túnel de datos antes de renegociar |
| **Enrutamiento sobre el túnel** | OSPF (Área 0) | Propagación dinámica de rutas LAN entre sitios |
| **Subnet del túnel VTI** | 10.25.37.0/30 | Enlace lógico punto a punto entre R1 y R2 |

> **Nota sobre PRF:** En IOSv/IOS 15.x el comando `prf` dentro de `crypto ikev2 proposal` no está disponible. IOS deriva automáticamente la PRF del algoritmo de integridad configurado (SHA-256). No es necesario declararlo.

---

## 3. Documentación de la Red

### 3.1 Topología

El laboratorio simula la interconexión segura de dos sitios corporativos a través de Internet. El segmento `192.168.1.0/24` representa la nube pública/ISP. Las subredes LAN privadas están derivadas de la matrícula `20250737`. La interfaz `Tunnel0` en cada router establece el enlace virtual IKEv2/IPSec punto a punto usando la subred `10.25.37.0/30`.

```
                              [ INTERNET / ISP ]
                              192.168.1.0/24
                              (Router ISP: 192.168.1.2)
                                    │
                   ┌────────────────┴────────────────┐
                   │ e0/0: 192.168.1.10              │ e0/0: 192.168.1.20
           ┌───────┴───────┐                 ┌───────┴───────┐
           │  Router R1    │◄═══ TÚNEL ══════►  Router R2    │
           │  (Site A)     │  IKEv2 + IPSec  │  (Site B)     │
           │  Tunnel0:     │  Route-Based    │  Tunnel0:     │
           │  10.25.37.1   │                 │  10.25.37.2   │
           └───────┬───────┘                 └───────┬───────┘
                   │ e0/1: 20.25.37.129/25           │ e0/1: 20.25.37.1/25
                   │                                 │
           ┌───────┴───────┐                 ┌───────┴───────┐
           │  Switch SW1   │                 │  Switch SW2   │
           └───┬───────┬───┘                 └───┬───────┬───┘
               │       │                         │       │
           ┌───┴──┐ ┌──┴───┐               ┌────┴─┐ ┌───┴──┐
           │ PC1  │ │ PC2  │               │ PC3  │ │ PC4  │
           │.130  │ │.131  │               │ .2   │ │ .3   │
           └──────┘ └──────┘               └──────┘ └──────┘
            20.25.37.128/25                 20.25.37.0/25
               SITE A                          SITE B

  ════════════════════════════════════════════════════════════════
  Flujo del túnel IKEv2 Route-Based:
    1. PC1 (20.25.37.130) hace ping a PC3 (20.25.37.2).
    2. R1 consulta su tabla de rutas: 20.25.37.0/25 apunta
       a Tunnel0 (aprendida por OSPF).
    3. El paquete entra a Tunnel0 → IPSec Profile lo cifra.
    4. IKEv2 negocia en background (4 mensajes) si la SA
       no existe aún — IKE_SA_INIT + IKE_AUTH.
    5. R1 envía el paquete ESP cifrado a 192.168.1.20.
    6. R2 descifra y entrega a PC3. Sin ACL de por medio.
  ════════════════════════════════════════════════════════════════
```

### 3.2 Tabla de Dispositivos y Direccionamiento IP

| Dispositivo | Tipo / Modelo | Interfaz | Dirección IP | Máscara | Gateway | Rol |
|---|---|---|---|---|---|---|
| **ISP** | Router ISP | e0/0 | 192.168.1.2 | /24 | — | Enlace hacia R1 |
| | | e0/1 | 192.168.1.2 | /24 | — | Enlace hacia R2 |
| **R1** | Cisco IOS (Router) | e0/0 | 192.168.1.10 | /24 | 192.168.1.2 | Gateway WAN — Site A |
| | | e0/1 | 20.25.37.129 | /25 | — | Gateway LAN — Site A |
| | | Tunnel0 | 10.25.37.1 | /30 | — | VTI — Endpoint local del túnel IKEv2 |
| **R2** | Cisco IOS (Router) | e0/0 | 192.168.1.20 | /24 | 192.168.1.2 | Gateway WAN — Site B |
| | | e0/1 | 20.25.37.1 | /25 | — | Gateway LAN — Site B |
| | | Tunnel0 | 10.25.37.2 | /30 | — | VTI — Endpoint remoto del túnel IKEv2 |
| **SW1** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN Site A |
| **SW2** | Cisco IOS (Switch L2) | — | — | — | — | Conmutación LAN Site B |
| **PC1** | Host Linux / VPC | eth0 | 20.25.37.130 | /25 | 20.25.37.129 | Host Site A |
| **PC2** | Host Linux / VPC | eth0 | 20.25.37.131 | /25 | 20.25.37.129 | Host Site A |
| **PC3** | Host Linux / VPC | eth0 | 20.25.37.2 | /25 | 20.25.37.1 | Host Site B |
| **PC4** | Host Linux / VPC | eth0 | 20.25.37.3 | /25 | 20.25.37.1 | Host Site B |

---

## 4. Scripts de Configuración

### 4.1 Router R1 — Site A

```cisco
! ══════════════════════════════════════════════════════
! R1 — Site A | IPSec IKEv2 Route-Based VPN (VTI)
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R1

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-ISP
 ip address 192.168.1.10 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteA
 ip address 20.25.37.129 255.255.255.128
 no shutdown

! ─── Ruta por defecto hacia Internet (ISP) ─────────────
ip route 0.0.0.0 0.0.0.0 192.168.1.2

! ══════════════════════════════════════════════════════
! BLOQUE IKEv2
! ══════════════════════════════════════════════════════

! ─── PASO 1: IKEv2 Proposal ────────────────────────────
! Define los algoritmos del canal IKE SA.
! NOTA: no incluir "prf sha256" — no soportado en IOSv.
! IOS deriva el PRF automáticamente del integrity algorithm.
crypto ikev2 proposal PROP_IKEv2
 encryption aes-cbc-256
 integrity sha256
 group 14

! ─── PASO 2: IKEv2 Policy ──────────────────────────────
! Vincula el Proposal a la política global IKEv2.
crypto ikev2 policy POL_IKEv2
 proposal PROP_IKEv2

! ─── PASO 3: IKEv2 Keyring ─────────────────────────────
! Define la pre-shared key para el peer remoto (R2).
crypto ikev2 keyring KR_SITEB
 peer R2_SITEB
  address 192.168.1.20
  pre-shared-key local  JordyITLA2026!
  pre-shared-key remote JordyITLA2026!

! ─── PASO 4: IKEv2 Profile ─────────────────────────────
! Une la identidad del peer, la autenticación y el keyring.
! Este profile se referenciará desde el IPSec Profile.
crypto ikev2 profile PROF_IKEv2
 match identity remote address 192.168.1.20 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_SITEB

! ══════════════════════════════════════════════════════
! BLOQUE IPSEC
! ══════════════════════════════════════════════════════

! ─── PASO 5: Transform Set ─────────────────────────────
! Define cifrado y autenticación del tráfico de datos (ESP).
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ─── PASO 6: IPSec Profile ─────────────────────────────
! Vincula el Transform Set y el IKEv2 Profile.
! Se aplica a la VTI — NO hay Crypto Map en este modelo.
! Diferencia vs IKEv1 route-based: se agrega "set ikev2-profile".
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256
 set ikev2-profile PROF_IKEv2
 set security-association lifetime seconds 3600

! ─── PASO 7: Virtual Tunnel Interface (VTI) ────────────
! La interfaz Tunnel0 actúa como enlace punto a punto.
! tunnel protection aplica el cifrado IKEv2/IPSec.
interface Tunnel0
 description IKEv2-VTI-Route-Based-hacia-R2
 ip address 10.25.37.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel destination 192.168.1.20
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROFILE_VTI
 no shutdown

! ─── PASO 8: OSPF sobre el túnel ───────────────────────
! El enrutamiento dinámico reemplaza a las ACLs de
! tráfico interesante del modelo policy-based.
router ospf 1
 network 20.25.37.128 0.0.0.127 area 0    ! LAN local Site A
 network 10.25.37.0 0.0.0.3 area 0        ! Subred del túnel VTI
```

### 4.2 Router R2 — Site B

```cisco
! ══════════════════════════════════════════════════════
! R2 — Site B | IPSec IKEv2 Route-Based VPN (VTI)
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R2

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-ISP
 ip address 192.168.1.20 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteB
 ip address 20.25.37.1 255.255.255.128
 no shutdown

! ─── Ruta por defecto hacia Internet (ISP) ─────────────
ip route 0.0.0.0 0.0.0.0 192.168.1.2

! ─── PASO 1: IKEv2 Proposal ────────────────────────────
crypto ikev2 proposal PROP_IKEv2
 encryption aes-cbc-256
 integrity sha256
 group 14

! ─── PASO 2: IKEv2 Policy ──────────────────────────────
crypto ikev2 policy POL_IKEv2
 proposal PROP_IKEv2

! ─── PASO 3: IKEv2 Keyring ─────────────────────────────
crypto ikev2 keyring KR_SITEA
 peer R1_SITEA
  address 192.168.1.10
  pre-shared-key local  JordyITLA2026!
  pre-shared-key remote JordyITLA2026!

! ─── PASO 4: IKEv2 Profile ─────────────────────────────
crypto ikev2 profile PROF_IKEv2
 match identity remote address 192.168.1.10 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local KR_SITEA

! ─── PASO 5: Transform Set ─────────────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ─── PASO 6: IPSec Profile ─────────────────────────────
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256
 set ikev2-profile PROF_IKEv2
 set security-association lifetime seconds 3600

! ─── PASO 7: Virtual Tunnel Interface (VTI) ────────────
interface Tunnel0
 description IKEv2-VTI-Route-Based-hacia-R1
 ip address 10.25.37.2 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel destination 192.168.1.10
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROFILE_VTI
 no shutdown

! ─── PASO 8: OSPF sobre el túnel ───────────────────────
router ospf 1
 network 20.25.37.0 0.0.0.127 area 0      ! LAN local Site B
 network 10.25.37.0 0.0.0.3 area 0        ! Subred del túnel VTI
```

### 4.3 Configuración de PCs (Hosts)

**PC1 y PC2 — Site A (`20.25.37.128/25`)**

```bash
# PC1
ip 20.25.37.130 255.255.255.128 20.25.37.129

# PC2
ip 20.25.37.131 255.255.255.128 20.25.37.129
```

**PC3 y PC4 — Site B (`20.25.37.0/25`)**

```bash
# PC3
ip 20.25.37.2 255.255.255.128 20.25.37.1

# PC4
ip 20.25.37.3 255.255.255.128 20.25.37.1
```

> **Nota:** Si los hosts son VPCs de PNETLab se usa el comando `ip` como se muestra. Si son máquinas Linux se configura con `ip addr add` e `ip route add default via`.

---

## 5. Verificación del Túnel

---

### 5.1 Verificar el estado de la interfaz Tunnel0

```cisco
R1# show interface Tunnel0
```

*Salida esperada:*

```
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Description: IKEv2-VTI-Route-Based-hacia-R2
  Internet address is 10.25.37.1/30
  MTU 1400 bytes, BW 100 Kbit/sec, DLY 50000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation TUNNEL, loopback not set
  Tunnel source 192.168.1.10 (Ethernet0/0), destination 192.168.1.20
  Tunnel protocol/transport IPSEC/IP
  Tunnel protection via IPSec (profile "IPSEC_PROFILE_VTI")
```

> La línea `Tunnel protection via IPSec (profile "IPSEC_PROFILE_VTI")` confirma que la VTI está protegida por el perfil IKEv2/IPSec. Si el estado es `down/down`, verificar primero que haya conectividad WAN: `ping 192.168.1.20` desde R1.

---

### 5.2 Verificar el estado de la IKEv2 SA

```cisco
R1# show crypto ikev2 sa
```

*Salida esperada:*

```
 IPv4 Crypto IKEv2  SA

Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         192.168.1.10/500      192.168.1.20/500      none/none            READY
      Encr: AES-CBC, keysize: 256, Hash: SHA256, DH Grp:14, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/XX sec
```

> Estado `READY` confirma que la IKE SA está establecida. La ausencia de la sección `Child SA` en este output es normal en IOSv — los detalles de la Child SA se ven con `show crypto ipsec sa`.

---

### 5.3 Verificar las IPSec SAs (Child SAs)

```cisco
R1# show crypto ipsec sa
```

*Salida esperada (fragmento):*

```
interface: Tunnel0
    Crypto map tag: Tunnel0-head-0, local addr 192.168.1.10

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   remote ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   current_peer 192.168.1.20 port 500

    #pkts encaps: 10, #pkts encrypt: 10, #pkts digest: 10
    #pkts decaps: 10, #pkts decrypt: 10, #pkts verify: 10
```

> **Diferencia clave vs. policy-based:** Los identifiers muestran `0.0.0.0/0.0.0.0` — correcto. La VTI protege todo el tráfico que entra a `Tunnel0`, no un subconjunto definido por ACL. La interfaz referenciada es `Tunnel0`, no `Ethernet0/0` como en el modelo policy-based.

---

### 5.4 Verificar la tabla de enrutamiento OSPF

```cisco
R1# show ip route ospf
```

*Salida esperada:*

```
      20.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
O        20.25.37.0/25 [110/1001] via 10.25.37.2, 00:01:20, Tunnel0
```

> La ruta hacia la LAN del Site B aprendida por OSPF a través de `Tunnel0` es la demostración del modelo *route-based*: el enrutamiento dinámico sobre el túnel IKEv2/IPSec funciona sin ninguna ACL adicional.

---

### 5.5 Verificar adyacencia OSPF sobre el túnel

```cisco
R1# show ip ospf neighbor
```

*Salida esperada:*

```
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.25.37.2        1   FULL/DR         00:00:35    10.25.37.2      Tunnel0
```

> Estado `FULL` confirma que OSPF estableció adyacencia completa sobre el túnel IKEv2. Esto es posible porque IPSec en modo túnel sobre VTI transporta el tráfico multicast de OSPF (224.0.0.5), a diferencia de IPSec policy-based puro.

---

### 5.6 Verificar conectividad extremo a extremo

Ejecutar desde **PC1** hacia **PC3**:

```bash
PC1> ping 20.25.37.2
```

*Resultado esperado:*

```
84 bytes from 20.25.37.2 icmp_seq=1 ttl=62 time=X ms
84 bytes from 20.25.37.2 icmp_seq=2 ttl=62 time=X ms
84 bytes from 20.25.37.2 icmp_seq=3 ttl=62 time=X ms
```

---

### 5.7 Tabla de comandos de verificación

| Comando | Qué muestra |
|---|---|
| `show interface Tunnel0` | Estado `up/up` y perfil IPSec vinculado. |
| `show crypto ikev2 sa` | Estado IKEv2 SA. Debe ser `READY`. |
| `show crypto ikev2 sa detail` | Parámetros negociados, SPIs IKE, identidades. |
| `show crypto ipsec sa` | Child SAs, contadores `#pkts encaps/decaps`. |
| `show ip route ospf` | Rutas aprendidas dinámicamente a través del túnel. |
| `show ip ospf neighbor` | Adyacencia OSPF sobre `Tunnel0` en estado `FULL`. |
| `show crypto ikev2 session` | Resumen de la sesión IKEv2 activa. |
| `show crypto ipsec profile` | Confirma el perfil IPSec y el IKEv2 profile vinculado. |
| `debug crypto ikev2` | Debug de la negociación IKEv2 en tiempo real. |

---

## 6. Capturas de Pantalla

A continuación se detalla el índice de evidencias correspondientes a las fases de configuración, verificación y funcionamiento de la VPN IKEv2 Route-Based, alojadas en la carpeta [`screenshots/`](screenshots/README.md):

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, todos los dispositivos encendidos. |
| 2 | [`02_ping_wan_previo.png`](screenshots/02_ping_wan_previo.png) | Ping desde R1 (`192.168.1.10`) hacia R2 (`192.168.1.20`) confirmando conectividad WAN base antes de configurar el túnel. |
| 3 | [`03_config_r1_ikev2.png`](screenshots/03_config_r1_ikev2.png) | Consola de R1 mostrando la configuración de `crypto ikev2 proposal`, `crypto ikev2 policy` y `crypto ikev2 keyring`. |
| 4 | [`04_config_r1_profile_vti.png`](screenshots/04_config_r1_profile_vti.png) | Consola de R1 mostrando el `crypto ikev2 profile`, el `crypto ipsec profile` con `set ikev2-profile` y la interfaz `Tunnel0` con `tunnel protection`. |
| 5 | [`05_config_r1_ospf.png`](screenshots/05_config_r1_ospf.png) | Consola de R1 mostrando la configuración de OSPF con las redes LAN y la subred del túnel en el Área 0. |
| 6 | [`06_config_r2_completa.png`](screenshots/06_config_r2_completa.png) | Configuración completa de R2 (Site B) mostrando la simetría con R1: keyring apuntando a `192.168.1.10`, VTI con `tunnel destination 192.168.1.10`. |
| 7 | [`07_tunnel0_up.png`](screenshots/07_tunnel0_up.png) | Salida de `show interface Tunnel0` en R1 mostrando estado `up/up` y `Tunnel protection via IPSec (profile "IPSEC_PROFILE_VTI")`. |
| 8 | [`08_ikev2_sa_ready.png`](screenshots/08_ikev2_sa_ready.png) | Salida de `show crypto ikev2 sa` mostrando estado `READY` con parámetros AES-256/SHA256/DH14 después de disparar la negociación con ping extendido. |
| 9 | [`09_ospf_neighbor_full.png`](screenshots/09_ospf_neighbor_full.png) | Salida de `show ip ospf neighbor` mostrando adyacencia en estado `FULL` sobre `Tunnel0` — prueba del enrutamiento dinámico sobre IKEv2. |
| 10 | [`10_ping_extremo_a_extremo.png`](screenshots/10_ping_extremo_a_extremo.png) | Ping exitoso desde PC1 (`20.25.37.130`) hacia PC4 (`20.25.37.3`) con el túnel IKEv2/IPSec activo, TTL=62. |

---

## 7. Consideraciones de Seguridad

### 7.1 Ventajas de seguridad de IKEv2 + VTI sobre los modelos anteriores

La combinación IKEv2 + VTI acumula las ventajas de seguridad de ambos componentes:

* **Sin Aggressive Mode:** IKEv2 no tiene modo equivalente — toda la autenticación viaja cifrada desde el segundo mensaje.
* **Dead Peer Detection integrado:** IKEv2 detecta automáticamente cuando el peer remoto deja de responder y puede renegociar o cerrar el túnel, sin configuración adicional de DPD como en IKEv1.
* **Sin ACLs de tráfico interesante:** La eliminación de las ACLs reduce el riesgo de misconfiguraciones que dejen tráfico sin cifrar — todo lo que entre a `Tunnel0` es cifrado sin excepción.
* **Una sola SA por túnel:** Reduce la superficie de negociación y simplifica la auditoría de seguridad.
* **Interfaz monitoreable:** `Tunnel0` aparece en `show interface` y puede ser integrada a sistemas de monitoreo (SNMP, Zabbix) como cualquier otro enlace.

### 7.2 Hardening recomendado

```cisco
! Deshabilitar IKEv1 completamente si solo se usa IKEv2
no crypto isakmp enable

! Eliminar la policy ISAKMP legacy por defecto de Cisco
no crypto isakmp policy 65535

! Forzar Perfect Forward Secrecy en el IPSec Profile
crypto ipsec profile IPSEC_PROFILE_VTI
 set pfs group14

! Restringir tráfico IKE y ESP solo al peer conocido en la WAN
ip access-list extended ACL_WAN_IKEv2
 permit udp host 192.168.1.20 host 192.168.1.10 eq 500
 permit udp host 192.168.1.20 host 192.168.1.10 eq 4500
 permit esp host 192.168.1.20 host 192.168.1.10
 deny ip any any log

interface Ethernet0/0
 ip access-group ACL_WAN_IKEv2 in
```

### 7.3 Diferencia de una línea vs. IKEv1 Route-Based

La configuración de IKEv2 route-based es prácticamente idéntica a la de IKEv1 route-based — la única diferencia real está en el bloque IKEv2 (que reemplaza `crypto isakmp`) y en esta línea dentro del IPSec Profile:

```cisco
! IKEv1 Route-Based:
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256
 ! ← sin ikev2-profile

! IKEv2 Route-Based (este lab):
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256
 set ikev2-profile PROF_IKEv2          ! ← esta es la única diferencia
```

Sin `set ikev2-profile` en el perfil, IOS intentará negociar con IKEv1 aunque el bloque `crypto ikev2` esté completo — ese es el error de configuración más común al migrar de IKEv1 a IKEv2 en el modelo route-based.

---

## 8. Video Demostrativo

🎥 **[Ver demostración en YouTube](#)**

**Duración:** máximo 8 minutos

**Contenido del video:**

* ✅ Topología funcional en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica del laboratorio.
* ✅ Demostración de los scripts de configuración aplicados en R1 y R2.
* ✅ Verificación de `show interface Tunnel0` mostrando `up/up` y el perfil IPSec activo.
* ✅ Verificación de `show crypto ikev2 sa` mostrando estado `READY`.
* ✅ Verificación de `show ip ospf neighbor` mostrando adyacencia `FULL` sobre `Tunnel0`.
* ✅ Ping exitoso extremo a extremo entre PC1 (`20.25.37.130`) y PC3/PC4 (`20.25.37.2/3`).

---

## 9. Referencias

* Kaufman, C. et al. (2014). *RFC 7296 — Internet Key Exchange Protocol Version 2 (IKEv2)*. IETF.
* Kent, S. & Seo, K. (2005). *RFC 4301 — Security Architecture for the Internet Protocol*. IETF.
* Eronen, P. & Laganier, J. (2007). *RFC 4555 — IKEv2 Mobility and Multihoming Protocol (MOBIKE)*. IETF.
* Cisco Systems. (2024). *Cisco IOS Security Configuration Guide: Secure Connectivity — IKEv2 with Virtual Tunnel Interface*.
* Cisco Systems. (2024). *Cisco IOS Security Command Reference — crypto ikev2 / tunnel protection*.
* Doraswamy, N. & Harkins, D. (2003). *IPSec: The New Security Standard for the Internet, Intranets, and Virtual Private Networks (2nd Ed.)*. Prentice Hall.
