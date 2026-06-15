# Guía Completa: RIP con FRR — Laboratorio 5
### Redes de Computadoras · UNS · Debian 12

---

## ÍNDICE
1. [Conceptos: Routing y protocolos de enrutamiento](#1-conceptos-routing-y-protocolos-de-enrutamiento)
2. [¿Qué es RIP y cómo funciona?](#2-qué-es-rip-y-cómo-funciona)
3. [¿Qué es FRR y qué es Zebra?](#3-qué-es-frr-y-qué-es-zebra)
4. [VLSM: Cálculo de subredes](#4-vlsm-cálculo-de-subredes)
5. [Habilitación de IP Forwarding](#5-habilitación-de-ip-forwarding)
6. [Instalación y habilitación de FRR](#6-instalación-y-habilitación-de-frr)
7. [Configuración de interfaces con vtysh](#7-configuración-de-interfaces-con-vtysh)
8. [Rutas estáticas en router1](#8-rutas-estáticas-en-router1)
9. [Configuración de RIP con unicast](#9-configuración-de-rip-con-unicast)
10. [NAT en router_internet](#10-nat-en-router_internet)
11. [Verificación de tablas de enrutamiento](#11-verificación-de-tablas-de-enrutamiento)
12. [Prueba de desconexión de Red6](#12-prueba-de-desconexión-de-red6)
13. [Configuración de Workstation 1](#13-configuración-de-workstation-1)
14. [Prueba con elinks](#14-prueba-con-elinks)
15. [Resumen de comandos rápidos](#15-resumen-de-comandos-rápidos)

---

## 1. Conceptos: Routing y protocolos de enrutamiento

### ¿Qué es el routing?

Cuando un paquete IP viaja de un origen a un destino, puede atravesar múltiples redes y múltiples routers. Cada router toma una decisión local: "¿por qué interfaz reenvío este paquete?". Esa decisión se basa en la **tabla de enrutamiento** (*routing table*):

```
Para llegar a 192.168.12.0/24 → enviar por eth1 hacia 192.168.13.202 (router3)
Para llegar a 0.0.0.0/0       → enviar por eth2 hacia 172.31.0.254  (internet)
```

### ¿Cómo se llena la tabla de enrutamiento?

**1. Rutas directamente conectadas**: cuando configurás una IP en una interfaz, el sistema agrega automáticamente una ruta para esa red. No hace falta configurarlas manualmente.

**2. Rutas estáticas**: el administrador las define a mano. Son fijas. No se adaptan automáticamente si la topología cambia.

**3. Rutas dinámicas (protocolos de enrutamiento)**: los routers intercambian información entre sí. Si algo cambia, las tablas se actualizan solas. En este labo usamos **RIP**.

En este labo: **router1 usa rutas estáticas** y **el resto usa RIP**.

---

## 2. ¿Qué es RIP y cómo funciona?

**RIP** (Routing Information Protocol) es un protocolo **distance-vector**: cada router conoce la distancia (en saltos) a cada red y comunica ese vector a sus vecinos directos.

### La métrica: número de saltos

```
router1 → router2 → router3 → router_internet

Desde router2:  router_internet está a 2 saltos (via router3 → Red31)
Desde router1:  router_internet está a 3 saltos (static → router2 → router3 → Red31)
```

**Métrica máxima = 16 = infinito**. Redes con métrica 16 se consideran inalcanzables. Por eso RIP solo funciona bien en redes pequeñas (máximo ~15 routers de diámetro).

### Los timers de RIP

| Timer              | Valor por defecto | Significado                                            |
|--------------------|-------------------|--------------------------------------------------------|
| Update             | 30 segundos       | Cada cuánto envía la tabla a vecinos                   |
| Timeout            | 180 segundos      | Sin actualización en 180s → ruta inválida (métrica=16) |
| Garbage collection | 120 segundos      | Tiempo que mantiene la ruta inválida antes de borrarla |

### RIPv1 vs RIPv2 — por qué usamos v2

| Característica          | RIPv1     | RIPv2                 |
|-------------------------|-----------|-----------------------|
| Tipo                    | Classful  | Classless             |
| Envía máscara de subred | No        | **Sí**                |
| Soporte VLSM            | No        | **Sí**                |
| Envío de updates        | Broadcast | Multicast (224.0.0.9) |

**Usamos RIPv2** porque nuestra red usa VLSM. Con RIPv1 los routers no sabrían qué máscara aplicar a cada red.

### RIP multicast vs unicast — por qué usamos unicast

Por defecto, RIPv2 envía sus updates a la dirección multicast `224.0.0.9`. En entornos virtualizados (LXD, contenedores) el multicast frecuentemente no funciona: los paquetes no llegan o son descartados.

Por eso configuramos RIP en modo **unicast**: cada router envía sus updates directamente a las IPs de sus vecinos. En FRR:

```
passive-interface default     ← deshabilita multicast/broadcast en TODAS las interfaces
neighbor X.X.X.X              ← habilita envío unicast a ese vecino específico
```

`passive-interface default` suprime todo envío multicast. `neighbor` lo anula selectivamente para unicast. La combinación logra unicast puro.

### Prevención de bucles de enrutamiento

**Split Horizon**: un router no anuncia una ruta por la misma interfaz por donde la aprendió.

**Poison Reverse**: variante más agresiva. En vez de no anunciar, anuncia la ruta con métrica 16. Converge más rápido.

**Count to infinity**: si hay un bucle, los routers incrementan la métrica hasta 16, limitando el daño.

---

## 3. ¿Qué es FRR y qué es Zebra?

### FRR (Free Range Routing)

FRR es el software que nos da la cátedra para implementar protocolos de enrutamiento dinámico en Linux. Implementa RIP, OSPF, BGP, entre otros.

**Punto crítico del labo**: toda la configuración se hace desde la **shell de FRR (vtysh)**. La evaluación se hace sobre la shell de FRR. El único archivo que editamos directamente es `/etc/frr/daemons` para habilitar qué daemons corren.

### La arquitectura de FRR

```
┌─────────────────────────────────────────────────────┐
│                  FRR Suite                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐              │
│  │  ripd   │  │  ospfd  │  │  bgpd   │  ← protocolos|
│  └────┬────┘  └────┬────┘  └────┬────┘              │
│       └────────────┴────────────┘                   │
│  ┌──────────────────────────────────┐               │
│  │             zebra                │ ← núcleo      │
│  └─────────────────┬────────────────┘               │
└────────────────────┼────────────────────────────────┘
                     │
           ┌─────────▼──────────┐
           │  Kernel de Linux   │ ← tabla de rutas real
           └────────────────────┘
```

**Zebra**: daemon central. Gestiona interfaces y actúa de puente entre los protocolos y el kernel. Cuando ripd aprende una ruta, se la comunica a zebra, que la instala en el kernel.

**ripd**: implementa RIP. Intercambia mensajes unicast con vecinos y entrega rutas a zebra.

En este labo leemos el manual de **Zebra** (interfaces y rutas estáticas) y **RIP** (protocolo de enrutamiento).

### vtysh — la shell de FRR

```bash
vtysh          # entra al intérprete
```

Modos dentro de vtysh:
```
router2#                      ← modo exec (consultas con show)
router2# configure terminal
router2(config)#              ← modo configuración global
router2(config)# interface eth0
router2(config-if)#           ← modo configuración de interfaz
router2(config-if)# exit
router2(config)# router rip
router2(config-router)#       ← modo configuración de protocolo
router2(config-router)# end
router2#
router2# write                ← GUARDAR SIEMPRE ANTES DE SALIR
router2# exit
```

> **`write` es obligatorio antes de salir de vtysh.** Sin él, los cambios se pierden al reiniciar FRR. Es equivalente a `write memory` o `copy running-config startup-config` de Cisco.

Atajo sin entrar interactivamente:
```bash
vtysh -c "show ip route"
```

### Diferencia entre configurar desde vtysh vs desde bash

| Aspecto                | `ip addr add` (bash) | `ip address` (vtysh) |
|------------------------|----------------------|----------------------|
| Persiste al reiniciar  | No                   | Sí (en frr.conf)     |
| Zebra tiene control    | No                   | Sí                   |
| ripd puede anunciarla  | No                   | Sí                   |
| Es evaluado en el labo | No                   | **Sí**               |

En este labo **toda la configuración de routers se hace desde vtysh**. Los comandos bash se incluyen como referencia, pero la configuración evaluable es la de vtysh.

---

## 4. VLSM: Cálculo de subredes

### Bloque disponible
```
192.168.10.0/24
192.168.11.0/24
192.168.12.0/24
192.168.13.0/24
```

### Topología — qué conecta cada enlace P2P

```
router1 ──(Red3)─────────────── router2
                                   │        \
                                 (Red6)    (Red7)
                                   │          \
                                router3       router4
                               /   │              │
                           (Red9) (Red31)       (Red8)
                           /       │
                       router4  router_internet
```

- **Red6** (P2P): router2 ↔ router3
- **Red7** (P2P): router2 ↔ router4
- **Red9** (P2P): router3 ↔ router4
- **Red31** (/24 LAN): router3 + router_internet

### Redes ordenadas de mayor a menor

| Orden | Red  | Hosts req. | Hosts útiles | Máscara | Dirección de red |
|-------|------|------------|--------------|---------|------------------|
| 1     | Red3 | 335        | 510          | /23     | 192.168.10.0     |
| 2     | Red8 | 128        | 254          | /24     | 192.168.12.0     |
| 3     | Red4 | 61         | 62           | /26     | 192.168.13.0     |
| 4     | Red1 | 48         | 62           | /26     | 192.168.13.64    |
| 5     | Red5 | 30         | 30           | /27     | 192.168.13.128   |
| 6     | Red2 | 15         | 30           | /27     | 192.168.13.160   |
| 7     | Red6 | 2 (P2P)    | 2            | /30     | 192.168.13.192   |
| 8     | Red7 | 2 (P2P)    | 2            | /30     | 192.168.13.196   |
| 9     | Red9 | 2 (P2P)    | 2            | /30     | 192.168.13.200   |

> **¿Por qué Red3 necesita /23?** 335 > 254 (/24 da solo 254 útiles). Con /23 tenés 2⁹−2=510. Consume 192.168.10.x y 192.168.11.x.
>
> **¿Por qué Red8 necesita /24?** 128 > 126 (/25 da solo 126 útiles).
>
> **¿Por qué Red2 con 15 hosts usa /27?** /28 da 14 útiles (insuficiente). /27 da 30 (suficiente).

### Detalle de cada subred con IPs asignadas

---

**Red3 — 192.168.10.0/23** — 335 equipos (sw_red3) — router1 ↔ router2

| Campo      | Valor                              |
|------------|------------------------------------|
| Máscara    | `255.255.254.0`                    |
| Broadcast  | `192.168.11.255`                   |
| Rango útil | `192.168.10.1` → `192.168.11.254`  |
| router1    | `192.168.10.1`                     |
| router2    | `192.168.10.2`                     |

---

**Red8 — 192.168.12.0/24** — 128 equipos (sw_red8) — router4

| Campo      | Valor                              |
|------------|------------------------------------|
| Máscara    | `255.255.255.0`                    |
| Broadcast  | `192.168.12.255`                   |
| Rango útil | `192.168.12.1` → `192.168.12.254`  |
| router4    | `192.168.12.1`                     |

---

**Red4 — 192.168.13.0/26** — 61 equipos (sw_red4) — router2

| Campo      | Valor                            |
|------------|----------------------------------|
| Máscara    | `255.255.255.192`                |
| Broadcast  | `192.168.13.63`                  |
| Rango útil | `192.168.13.1` → `192.168.13.62` |
| router2    | `192.168.13.1`                   |

---

**Red1 — 192.168.13.64/26** — 48 equipos (sw_red1) — router1 + ws1

| Campo             | Valor                              |
|-------------------|------------------------------------|
| Máscara           | `255.255.255.192`                  |
| Broadcast         | `192.168.13.127`                   |
| Rango útil        | `192.168.13.65` → `192.168.13.126` |
| Gateway / router1 | `192.168.13.65`                    |
| Workstation 1     | `192.168.13.66`                    |

---

**Red5 — 192.168.13.128/27** — 30 equipos (sw_red5) — router2

| Campo      | Valor                                |
|------------|--------------------------------------|
| Máscara    | `255.255.255.224`                    |
| Broadcast  | `192.168.13.159`                     |
| Rango útil | `192.168.13.129` → `192.168.13.158`  |
| router2    | `192.168.13.129`                     |

---

**Red2 — 192.168.13.160/27** — 15 equipos (sw_red2) — router1

| Campo      | Valor                                |
|------------|--------------------------------------|
| Máscara    | `255.255.255.224`                    |
| Broadcast  | `192.168.13.191`                     |
| Rango útil | `192.168.13.161` → `192.168.13.190`  |
| router1    | `192.168.13.161`                     |

---

**Red6 — 192.168.13.192/30** — P2P router2 ↔ router3 (sw_red6)

| Campo     | Valor             |
|-----------|-------------------|
| Máscara   | `255.255.255.252` |
| Broadcast | `192.168.13.195`  |
| router2   | `192.168.13.193`  |
| router3   | `192.168.13.194`  |

---

**Red7 — 192.168.13.196/30** — P2P router2 ↔ router4 (sw_red7)

| Campo     | Valor             |
|-----------|-------------------|
| Máscara   | `255.255.255.252` |
| Broadcast | `192.168.13.199`  |
| router2   | `192.168.13.197`  |
| router4   | `192.168.13.198`  |

---

**Red9 — 192.168.13.200/30** — P2P router3 ↔ router4 (sw_red9)

| Campo     | Valor             |
|-----------|-------------------|
| Máscara   | `255.255.255.252` |
| Broadcast | `192.168.13.203`  |
| router3   | `192.168.13.201`  |
| router4   | `192.168.13.202`  |

---

**Red31 — 172.31.0.0/24** — router3 ↔ router_internet (sw_red31)

| Campo           | Valor           |
|-----------------|-----------------|
| Máscara         | `255.255.255.0` |
| router3         | `172.31.0.1`    |
| router_internet | `172.31.0.254`  |

---

### Resumen de IPs por equipo

| Equipo          | Red   | Switch   | IP               | Máscara |
|-----------------|-------|----------|------------------|---------|
| router1         | Red3  | sw_red3  | `192.168.10.1`   | /23     |
| router1         | Red1  | sw_red1  | `192.168.13.65`  | /26     |
| router1         | Red2  | sw_red2  | `192.168.13.161` | /27     |
| router2         | Red3  | sw_red3  | `192.168.10.2`   | /23     |
| router2         | Red4  | sw_red4  | `192.168.13.1`   | /26     |
| router2         | Red5  | sw_red5  | `192.168.13.129` | /27     |
| router2         | Red6  | sw_red6  | `192.168.13.193` | /30     |
| router2         | Red7  | sw_red7  | `192.168.13.197` | /30     |
| router3         | Red6  | sw_red6  | `192.168.13.194` | /30     |
| router3         | Red9  | sw_red9  | `192.168.13.201` | /30     |
| router3         | Red31 | sw_red31 | `172.31.0.1`     | /24     |
| router4         | Red7  | sw_red7  | `192.168.13.198` | /30     |
| router4         | Red8  | sw_red8  | `192.168.12.1`   | /24     |
| router4         | Red9  | sw_red9  | `192.168.13.202` | /30     |
| router_internet | Red31 | sw_red31 | `172.31.0.254`   | /24     |
| ws1             | Red1  | sw_red1  | `192.168.13.66`  | /26     |

---

## 5. Habilitación de IP Forwarding

Por defecto Linux descarta paquetes que llegan a una interfaz con destino distinto a la IP propia. Un router necesita lo opuesto: recibir por una interfaz y reenviar por otra. Sin IP forwarding el router es inútil.

Debe activarse en **todos los routers**: router1, router2, router3, router4, router_internet.

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

> `sysctl -p` aplica el cambio inmediatamente y persiste al reiniciar.

Verificar:
```bash
sysctl net.ipv4.ip_forward
```
Salida esperada: `net.ipv4.ip_forward = 1`

---

## 6. Instalación y habilitación de FRR

**En todos los routers.**

```bash
apt update
apt install -y frr frr-pythontools
```

### Habilitar los daemons — único archivo que editamos directamente

```bash
nano /etc/frr/daemons
```

Cambiar los valores correspondientes:

```
zebra=yes    ← siempre necesario en todos los routers
ripd=yes     ← en router2, router3, router4, router_internet
             ← en router1 dejar ripd=no
```

> **¿Por qué router1 no tiene ripd?** Al no correr el daemon ripd, router1 es incapaz de participar en RIP: no envía updates, no procesa los que recibe, no tiene tabla RIP. No hace falta ninguna configuración adicional para "bloquearlo".

```bash
systemctl enable frr
systemctl start frr
systemctl status frr
```

Verificar daemons corriendo:
```bash
ps aux | grep -E "zebra|ripd"
```
> En router1 solo debe aparecer `zebra`. En el resto, `zebra` y `ripd`.

---

## 7. Configuración de interfaces con vtysh

### Antes de configurar: identificar interfaces

```bash
ip link show
```
> Identifica los nombres reales (eth0, eth1, ens3, etc.) y cuál conecta a cada red.

---

### router1 — 3 interfaces

```bash
vtysh
```

```
configure terminal

interface eth0
 description Red3-sw_red3
 ip address 192.168.10.1/23
 no shutdown
exit

interface eth1
 description Red1-sw_red1
 ip address 192.168.13.65/26
 no shutdown
exit

interface eth2
 description Red2-sw_red2
 ip address 192.168.13.161/27
 no shutdown
exit

end
write
```

---

### router2 — 5 interfaces

```bash
vtysh
```

```
configure terminal

interface eth0
 description Red3-sw_red3
 ip address 192.168.10.2/23
 no shutdown
exit

interface eth1
 description Red4-sw_red4
 ip address 192.168.13.1/26
 no shutdown
exit

interface eth2
 description Red5-sw_red5
 ip address 192.168.13.129/27
 no shutdown
exit

interface eth3
 description Red6-sw_red6-P2P-router3
 ip address 192.168.13.193/30
 no shutdown
exit

interface eth4
 description Red7-sw_red7-P2P-router4
 ip address 192.168.13.197/30
 no shutdown
exit

end
write
```

---

### router3 — 3 interfaces

```bash
vtysh
```

```
configure terminal

interface eth0
 description Red6-sw_red6-P2P-router2
 ip address 192.168.13.194/30
 no shutdown
exit

interface eth1
 description Red9-sw_red9-P2P-router4
 ip address 192.168.13.201/30
 no shutdown
exit

interface eth2
 description Red31-sw_red31-router_internet
 ip address 172.31.0.1/24
 no shutdown
exit

end
write
```

---

### router4 — 3 interfaces

```bash
vtysh
```

```
configure terminal

interface eth0
 description Red7-sw_red7-P2P-router2
 ip address 192.168.13.198/30
 no shutdown
exit

interface eth1
 description Red8-sw_red8
 ip address 192.168.12.1/24
 no shutdown
exit

interface eth2
 description Red9-sw_red9-P2P-router3
 ip address 192.168.13.202/30
 no shutdown
exit

end
write
```

---

### router_internet — 2 interfaces

router_internet tiene dos interfaces:
- `eth1`: Red31 → conecta con router3. **Esta la configuramos nosotros.**
- `eth0`: interfaz virtual del entorno LXD que da acceso a Internet real (se conecta a lxdbr0). **Ya existe y ya tiene IP asignada por el entorno del labo. No la tocamos.**

```bash
vtysh
```

```
configure terminal

interface eth0
 description Red31-sw_red31-router3
 ip address 172.31.0.254/24
 no shutdown
exit

end
write
```

> `lxdbr0` la deja el entorno preconfigurada. Es la salida real a Internet y es la interfaz que usamos en la regla de NAT (`-o lxdbr0`). Podés verla con `ip addr show lxdbr0`.

---

### Verificar conectividad entre vecinos directos

```bash
# Desde router1:
ping -c3 192.168.10.2      # router2 en Red3

# Desde router2:
ping -c3 192.168.10.1      # router1 en Red3
ping -c3 192.168.13.194    # router3 por Red6
ping -c3 192.168.13.198    # router4 por Red7

# Desde router3:
ping -c3 192.168.13.193    # router2 por Red6
ping -c3 192.168.13.202    # router4 por Red9
ping -c3 172.31.0.254      # router_internet por Red31

# Desde router4:
ping -c3 192.168.13.197    # router2 por Red7
ping -c3 192.168.13.201    # router3 por Red9

# Desde router_internet:
ping -c3 172.31.0.1        # router3 por Red31
```

Salida esperada de un ping exitoso:
```
PING 192.168.10.2 (192.168.10.2) 56(84) bytes of data.
64 bytes from 192.168.10.2: icmp_seq=1 ttl=64 time=0.4 ms
3 packets transmitted, 3 received, 0% packet loss
```

---

## 8. Rutas estáticas en router1

router1 no corre ripd, así que no aprende rutas automáticamente. Dado que todas las redes remotas y la salida a Internet se alcanzan cruzando su único vecino, router2 (192.168.10.2), **la configuración mínima indispensable consiste en una sola ruta por defecto**. Esto hace redundantes las rutas estáticas específicas para cada subred.

### Rutas directamente conectadas (automáticas)

FRR/zebra las agrega solo al configurar interfaces:
- `192.168.10.0/23` (Red3)
- `192.168.13.64/26` (Red1)
- `192.168.13.160/27` (Red2)

### Ruta estática a configurar (mínima indispensable)

```bash
vtysh
```

```
configure terminal

# Una sola ruta por defecto reemplaza y resume todas las demás
ip route 0.0.0.0/0 192.168.10.2

end
write
```

### Verificar

```bash
vtysh -c "show ip route static"
```

Salida esperada:
```
S>* 0.0.0.0/0 [1/0] via 192.168.10.2, eth0
```

`S` = estática. `>*` = seleccionada e instalada en el kernel. `[1/0]` = distancia administrativa 1 / métrica 0.

---

## 9. Configuración de RIP con unicast

RIP se configura en: **router2, router3, router4, router_internet**. No en router1.

### Conceptos clave

**`version 2`**: Obligatorio para VLSM. Sin esto usa RIPv1, que no envía máscaras de subred. **Nota de portabilidad:** Aunque algunas versiones de FRR usan la versión 2 por defecto, es indispensable explicitarlo en la configuración para evitar comportamientos mixtos o fallback automático a la versión 1, y para garantizar la compatibilidad si la topología se migra a otros entornos (como Cisco IOS, donde el valor por defecto es RIPv1).

**`passive-interface default`**: deshabilita envío de updates multicast/broadcast en todas las interfaces. Primer paso para unicast puro.

**`neighbor X.X.X.X`**: habilita envío unicast a esa IP específica, aunque la interfaz esté en modo pasivo. Solo se declaran los vecinos RIP directos.

**`network X.X.X.X/YY`**: declara qué redes participan en RIP y se anuncian a vecinos. Se ponen solo las redes directamente conectadas al router. No se repiten redes de otros routers.

**`default-information originate`**: solo en router_internet. Anuncia `0.0.0.0/0` a los vecinos para que aprendan el camino a Internet.

---

### router2 — RIP

Vecinos RIP: router3 (192.168.13.194 por Red6) y router4 (192.168.13.198 por Red7).

> **Nota de interconexión con router1 (estático):** Dado que router1 no habla RIP, router2 debe conocer estáticamente las redes detrás de router1 (Red1 y Red2) y redistribuirlas en su proceso RIP para que el resto de los routers puedan alcanzarlas.

```bash
vtysh
```

```
configure terminal

# Rutas estáticas hacia las redes de router1 (Red1 y Red2) vía Red3
ip route 192.168.13.64/26 192.168.10.1
ip route 192.168.13.160/27 192.168.10.1

router rip
 version 2
 passive-interface default
 network 192.168.10.0/23
 network 192.168.13.0/26
 network 192.168.13.128/27
 network 192.168.13.192/30
 network 192.168.13.196/30
 neighbor 192.168.13.194
 neighbor 192.168.13.198
 redistribute static
exit

end
write
```

---

### router3 — RIP

Vecinos RIP: router2 (192.168.13.193 por Red6), router4 (192.168.13.202 por Red9), router_internet (172.31.0.254 por Red31).

```bash
vtysh
```

```
configure terminal

router rip
 version 2
 passive-interface default
 network 192.168.13.192/30
 network 192.168.13.200/30
 network 172.31.0.0/24
 neighbor 192.168.13.193
 neighbor 192.168.13.202
 neighbor 172.31.0.254
exit

end
write
```

---

### router4 — RIP

Vecinos RIP: router2 (192.168.13.197 por Red7) y router3 (192.168.13.201 por Red9).

```bash
vtysh
```

```
configure terminal

router rip
 version 2
 passive-interface default
 network 192.168.13.196/30
 network 192.168.12.0/24
 network 192.168.13.200/30
 neighbor 192.168.13.197
 neighbor 192.168.13.201
exit

end
write
```

---

### router_internet — RIP + ruta por defecto

Vecino RIP: router3 (172.31.0.1 por Red31). Es el único router RIP que conecta con router_internet.

```bash
vtysh
```

```
configure terminal

router rip
 version 2
 passive-interface default
 network 172.31.0.0/24
 neighbor 172.31.0.1
 default-information originate
exit

end
write
```

> **`default-information originate`**: router_internet anuncia `0.0.0.0/0` a router3. La cadena de propagación:
> ```
> router_internet → router3 (Red31)
> router3 → router2 (Red6) y router4 (Red9)
> router4 → router2 (Red7) [redundante, ya llegó por Red6]
> ```
> Todos los routers RIP aprenden la ruta por defecto. router1 la tiene estática apuntando a router2.

### El proceso de convergencia

Tras configurar RIP en todos los routers, en los primeros 30-60 segundos las tablas se estabilizan. Para monitorearlo:

```bash
watch -n2 "vtysh -c 'show ip rip'"
```
> Ejecuta el comando cada 2 segundos. Mirá cómo aparecen las rutas progresivamente.

---

## 10. NAT en router_internet

### ¿Qué es NAT y por qué es necesario?

Las IPs de nuestras redes (192.168.x.x) son **privadas**: los routers de Internet no saben cómo llegar a ellas y descartarían las respuestas. Con NAT (Network Address Translation), router_internet "disfraza" los paquetes salientes: reemplaza la IP privada origen por su propia IP pública (la de lxdbr0). Cuando llega la respuesta, hace la traducción inversa.

Sin NAT, los paquetes salen a Internet pero las respuestas nunca regresan.

### Identificar la interfaz externa

```bash
ip link show
```
La interfaz `eth0` es la que conecta a Internet real.

### Configurar MASQUERADE (en router_internet)

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

> `-t nat`: tabla NAT del kernel.
> `-A POSTROUTING`: aplica después de decidir el enrutamiento, justo antes de enviar.
> `-j MASQUERADE`: reemplaza IP origen con la IP dinámica de lxdbr0.

Verificar:
```bash
iptables -t nat -L POSTROUTING -n -v
```

Salida esperada:
```
Chain POSTROUTING (policy ACCEPT)
 pkts bytes target     prot opt in  out     source    destination
    0     0 MASQUERADE all  --  *   eth0  0.0.0.0/0 0.0.0.0/0
```

### Hacer el NAT persistente

```bash
apt install -y iptables-persistent
netfilter-persistent save
```

> Sin esto las reglas se pierden al apagar.

---

## 11. Verificación de tablas de enrutamiento

### show ip route — tabla completa

```bash
vtysh -c "show ip route"
```

Cómo leer la salida:
```
Codes: K - kernel, C - connected, S - static, R - RIP
       > - selected route, * - FIB route

K>* 0.0.0.0/0 [0/0] via 10.0.3.1, lxdbr0           ← default vía Internet (kernel)
C>* 172.31.0.0/24 is directly connected, eth0       ← conectada directamente
R>* 192.168.10.0/23 [120/2] via 172.31.0.1, eth0    ← aprendida por RIP
R>* 192.168.12.0/24 [120/3] via 172.31.0.1, eth0    ← aprendida por RIP (3 saltos)
C>* 192.168.13.200/30 is directly connected, ...
```

- `K` = kernel (puesta por el SO)
- `C` = connected (directamente conectada, zebra la agrega al configurar la interfaz)
- `S` = static
- `R` = RIP
- `>*` = seleccionada e instalada en el kernel
- `[120/2]` = distancia administrativa 120 / métrica 2 (2 saltos)

> La **distancia administrativa** (120 para RIP) indica la confianza del protocolo. Menor número = mayor confianza. Las rutas conectadas tienen distancia 0 y siempre ganan.

### show ip rip — tabla RIP

```bash
vtysh -c "show ip rip"
```

Salida esperada (router3 después de convergencia):
```
     Network            Next Hop         Metric From            Tag Time
R(d) 0.0.0.0/0          172.31.0.254          2 172.31.0.254    0  02:15
R(n) 192.168.10.0/23    192.168.13.193        2 192.168.13.193  0  02:10
R(n) 192.168.13.0/26    192.168.13.193        2 192.168.13.193  0  02:10
R(n) 192.168.13.128/27  192.168.13.193        2 192.168.13.193  0  02:10
C(i) 192.168.13.192/30  0.0.0.0               1 self            0
R(n) 192.168.13.196/30  192.168.13.193        2 192.168.13.193  0  02:10
R(n) 192.168.12.0/24    192.168.13.202        2 192.168.13.202  0  02:05
C(i) 192.168.13.200/30  0.0.0.0               1 self            0
C(i) 172.31.0.0/24      0.0.0.0               1 self            0
```

- `C(i)` = connected via interface (directamente conectada, anunciada por RIP)
- `R(n)` = RIP normal (aprendida de un vecino)
- `R(d)` = RIP default route
- `From` = IP del router que envió esta información
- `Time` = cuenta regresiva hasta el timeout (180s)
- `Metric` = número de saltos

### show ip rip status — estado del daemon

```bash
vtysh -c "show ip rip status"
```

Salida esperada (router3):
```
Routing Protocol is "rip"
  Sending updates every 30 seconds with +/-5 seconds jitter
  Timeout after 180 seconds, garbage collected after 120 seconds
  Default version control: send version 2, receive version 2
  Routing for Networks:
    192.168.13.192/30
    192.168.13.200/30
    172.31.0.0/24
  Routing Information Sources:
    Gateway          BadPackets BadRoutes Distance Last Update
    192.168.13.193            0         0      120   00:00:18
    192.168.13.202            0         0      120   00:00:12
    172.31.0.254              0         0      120   00:00:25
```

> `Routing Information Sources` muestra los vecinos que están enviando updates activamente. Si un vecino no aparece, el unicast no está funcionando hacia ese vecino.

### Qué esperar en cada router tras convergencia

**router2** debe ver:
- `C`: Red3, Red4, Red5, Red6, Red7
- `R`: Red8 (via router4 o router3), Red9, Red31, `0.0.0.0/0`

**router3** debe ver:
- `C`: Red6, Red9, Red31
- `R`: Red3, Red4, Red5, Red7, Red8, `0.0.0.0/0`

**router4** debe ver:
- `C`: Red7, Red8, Red9
- `R`: Red3, Red4, Red5, Red6, Red31, `0.0.0.0/0`

**router_internet** debe ver:
- `C`: Red31
- `R`: Red3, Red4, Red5, Red6, Red7, Red8, Red9
- `K`: `0.0.0.0/0` via lxdbr0

**router1** debe ver:
- `C`: Red3, Red1, Red2
- `S`: todo lo demás

### Prueba end-to-end desde router1

```bash
ping -c2 192.168.13.1      # router2-Red4
ping -c2 192.168.13.129    # router2-Red5
ping -c2 192.168.13.193    # router2-Red6
ping -c2 192.168.13.194    # router3-Red6
ping -c2 192.168.13.201    # router3-Red9
ping -c2 172.31.0.1        # router3-Red31
ping -c2 192.168.13.197    # router2-Red7
ping -c2 192.168.13.198    # router4-Red7
ping -c2 192.168.13.202    # router4-Red9
ping -c2 192.168.12.1      # router4-Red8
ping -c2 172.31.0.254      # router_internet-Red31
ping -c2 8.8.8.8            # Internet

traceroute 192.168.12.1    # ver el camino hacia Red8
traceroute 8.8.8.8          # ver el camino hacia Internet
```

---

## 12. Prueba de desconexión de Red6

Red6 es el enlace P2P entre router2 (192.168.13.193) y router3 (192.168.13.194).

### a) Análisis teórico — qué debería pasar al desconectar Red6

**Topología completa:**
```
router1 ──(Red3)── router2 ──(Red6)── router3 ──(Red31)── router_internet
                      │                  │
                    (Red7)             (Red9)
                      │                  │
                   router4 ─────────────
```

**Caminos hacia router_internet desde router2:**
- Camino 1 (2 saltos): router2 → router3 (Red6) → router_internet (Red31)
- Camino 2 (3 saltos): router2 → router4 (Red7) → router3 (Red9) → router_internet (Red31)

RIP prefiere el camino 1 por tener menor métrica.

**Cuando Red6 cae:**

1. La interfaz eth3 (Red6) en router2 cae. Zebra lo detecta en milisegundos.
2. Zebra notifica a ripd: "perdí eth3".
3. ripd en router2:
   - Marca como inválidas las rutas que llegaban vía 192.168.13.194 (métrica=16)
   - Envía un **triggered update** inmediato (no espera los 30s)
4. router3 también detecta su eth0 caída y hace lo mismo.
5. router2 pierde la ruta directa a Red31 y `0.0.0.0/0` vía router3.
6. **Pero router2 aún tiene camino alternativo**: router2 → router4 (Red7) → router3 (Red9) → router_internet.
7. RIP reconverge: la ruta `0.0.0.0/0` en router2 ahora va vía router4 (192.168.13.198) con métrica 3.

**¿Qué queda sin acceso?**
- Red6 misma queda fuera de uso (obvia).
- router3 pierde la conexión directa con router2, pero mantiene conectividad via router4 (Red9) y con router_internet (Red31). Las rutas hacia Red3/Red4/Red5 llegan a router3 via router4 → router2.

**¿Qué sigue funcionando?**
- router1 → Internet: sí, por router2 → router4 → router3 → router_internet (3 saltos)
- router4 → Internet: sí, directo por router3 (Red9 → Red31)
- Toda la red sigue conectada, solo cambia el camino

**Convergencia**: si la interfaz cae físicamente (link down), es casi instantánea por el triggered update. Si se simula sin bajar la interfaz, puede tardar hasta 180s (timeout).

### b) Prueba práctica

**Paso 1: capturar estado antes**

```bash
# En router2:
vtysh -c "show ip route" > /tmp/antes.txt
vtysh -c "show ip rip"   >> /tmp/antes.txt
cat /tmp/antes.txt
```

```bash
# Conectividad antes (desde router1):
ping -c3 172.31.0.1        # router3 debe responder
traceroute 172.31.0.254    # debe pasar por router3 directo (Red6)
```

Traceroute esperado **antes**:
```
1  192.168.10.2              ← router2
2  192.168.13.194            ← router3 (por Red6)
3  172.31.0.254              ← router_internet
```

**Paso 2: desconectar Red6**

> [!IMPORTANT]
> En entornos virtualizados con LXD (puentes virtuales), si solo tiras la interfaz de router2, la interfaz de router3 seguirá en estado `UP` de su lado. Para simular la caída del enlace real de forma instantánea sin esperar el timeout de RIP (180 segundos), debes **bajar la interfaz en ambos extremos**.

En router2, bajar la interfaz de Red6:
```bash
ip link set eth3 down
```

Y en router3, bajar la interfaz correspondiente:
```bash
ip link set eth0 down
```

O desde vtysh en cada router:
```bash
# En router2:
vtysh -c "configure terminal" -c "interface eth3" -c "shutdown"

# En router3:
vtysh -c "configure terminal" -c "interface eth0" -c "shutdown"
```

**Paso 3: observar convergencia**

```bash
# Ver logs en tiempo real:
journalctl -u frr -f
```

Verás líneas como:
```
ripd[xxx]: ZEBRA: interface eth3 index 4 <down>
ripd[xxx]: rip_if_down: turns off 192.168.13.192/30
ripd[xxx]: Sending triggered update
```

```bash
# Ver tabla actualizada (~5 segundos después):
vtysh -c "show ip rip"
vtysh -c "show ip route"
```

**Paso 4: verificar el cambio**

```bash
vtysh -c "show ip route" > /tmp/despues.txt
vtysh -c "show ip rip"   >> /tmp/despues.txt
diff /tmp/antes.txt /tmp/despues.txt
```

Cambios esperados:
- Desaparecen rutas con `via 192.168.13.194` (router3 directo por Red6)
- La ruta `0.0.0.0/0` ahora va `via 192.168.13.198` (router4) con métrica 3

```bash
# Probar desde router1:
ping -c3 172.31.0.1        # router3: SIGUE FUNCIONANDO (via router4 → router3)
ping -c3 8.8.8.8            # Internet: SIGUE FUNCIONANDO
traceroute 172.31.0.254    # ahora va por router4 antes de llegar a router3
```

Traceroute esperado **después** de la caída:
```
1  192.168.10.2              ← router2
2  192.168.13.198            ← router4 (cambió: ahora va por acá)
3  192.168.13.201            ← router3 por Red9
4  172.31.0.254              ← router_internet
```

### c) Reconectar Red6 y verificar

```bash
ip link set eth3 up
```

O desde vtysh:
```bash
vtysh -c "configure terminal" -c "interface eth3" -c "no shutdown"
```

```bash
# Esperar un ciclo de update (~30s):
sleep 35
vtysh -c "show ip rip"
```

Las rutas vía router3 por Red6 deben reaparecer con métrica 2 (más corta que la alternativa por router4 con métrica 3). RIP converge de vuelta al camino óptimo.

```bash
traceroute 172.31.0.254    # debe volver a ir directo por router3 (Red6)
```

---

## 13. Configuración de Workstation 1

ws1 se configura **sin FRR**, con comandos del sistema operativo.

- IP: `192.168.13.66/26`
- Máscara: `255.255.255.192`
- Gateway: `192.168.13.65` (router1 en Red1)

```bash
# Verificar nombre de interfaz:
ip link show

# Asignar IP (reemplazá eth0 por el nombre real):
ip addr add 192.168.13.66/26 dev eth0
ip link set eth0 up
ip route add default via 192.168.13.65
```

### Verificar

```bash
ip addr show eth0
```
Salida esperada:
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.13.66/26 scope global eth0
```

```bash
ip route show
```
Salida esperada:
```
default via 192.168.13.65 dev eth0
192.168.13.64/26 dev eth0 proto kernel scope link src 192.168.13.66
```

```bash
ping -c3 192.168.13.65    # gateway (router1)
ping -c3 192.168.10.1     # router1 otra interfaz
ping -c3 192.168.13.1     # router2-Red4
ping -c3 8.8.8.8           # Internet
```

---

## 14. Prueba con elinks

`elinks` es un navegador web para la terminal.

```bash
apt update && apt install -y elinks
```

> Alternativa si hay problemas: `apt install -y lynx`

```bash
# Navegar a un sitio:
elinks http://info.cern.ch

# Verificar rápido sin interfaz interactiva:
elinks -dump http://example.com

# Con curl:
curl -I http://example.com
```

Controles de elinks: flechas para navegar, `Enter` para seguir links, `g` para ir a URL, `q` para salir.

### Si no puede conectar

```bash
ip route show | grep default          # verificar ruta por defecto
ping -c3 192.168.13.65                # verificar gateway
ping -c3 8.8.8.8                       # verificar Internet
echo "nameserver 8.8.8.8" > /etc/resolv.conf  # agregar DNS si falta
```

---

## 15. Resumen de comandos rápidos

### Flujo completo en todos los routers

```bash
# 1. IP Forwarding
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf && sysctl -p

# 2. Instalar FRR
apt update && apt install -y frr frr-pythontools

# 3. Habilitar daemons (único archivo que editamos directamente)
nano /etc/frr/daemons
# router1:        zebra=yes, ripd=no
# resto:          zebra=yes, ripd=yes

# 4. Iniciar FRR
systemctl enable frr && systemctl start frr

# 5. Configurar interfaces (vtysh — ver sección 7 para cada router)
vtysh

# 6. Ping a vecinos directos para verificar
ping -c3 <IP-vecino>

# 7. Configurar rutas:
#    router1: rutas estáticas (sección 8)
#    router2/3/4/internet: RIP unicast (sección 9)
vtysh

# 8. NAT en router_internet
iptables -t nat -A POSTROUTING -o lxdbr0 -j MASQUERADE
apt install -y iptables-persistent && netfilter-persistent save

# 9. Esperar convergencia RIP (~30-60s)
watch -n2 "vtysh -c 'show ip rip'"
```

### Comandos show para la evaluación

```bash
# 1. Tabla de rutas completa
vtysh -c "show ip route"          
# 2. Solo rutas estáticas (útil en router1)
vtysh -c "show ip route static"   
# 3. Solo rutas aprendidas por RIP
vtysh -c "show ip route rip"      
# 4. Base de datos del protocolo RIP
vtysh -c "show ip rip"            
# 5. Estado general de RIP, vecinos y timers
vtysh -c "show ip rip status"     
# 6. Estado físico e IPs de interfaces
vtysh -c "show interface"         
# 7. Configuración activa del router
vtysh -c "show running-config"    
```

---

#### 🔍 Guía de Interpretación de Salidas (Símbolos y Campos)

##### A) En `show ip route` / `show ip route static` / `show ip route rip`
* **`K` (Kernel):** Ruta agregada por el kernel del sistema operativo Linux (ej: loopback o autoconfiguración).
* **`C` (Connected):** Ruta directamente conectada. Zebra la crea automáticamente al asignar la IP a la interfaz.
* **`S` (Static):** Ruta estática configurada manualmente (`ip route ...`).
* **`R` (RIP):** Ruta dinámica aprendida a través del protocolo RIP.
* **`>` (Selected):** Indica que esta es la mejor ruta hacia el destino y ha sido la elegida para usarse.
* **`*` (FIB):** Indica que la ruta está activa e instalada en la tabla de reenvío del Kernel (Forwarding Information Base).
* **`[120/2]` (Distancia/Métrica):**
  * **`120` (Distancia Administrativa):** Indica el nivel de confianza del origen de la ruta (RIP es 120, Estática es 1, Conectada es 0). Menor número = mayor prioridad.
  * **`2` (Métrica):** En RIP representa el coste en saltos (cantidad de routers a cruzar) para alcanzar la red.
* **`via X.X.X.X`:** Dirección IP del siguiente salto (next-hop) al que se le enviará el paquete.
* **`ethX`:** Interfaz de salida física o lógica por la cual saldrá el paquete.

##### B) En `show ip rip`
* **`C(i)` (Connected interface):** Subred directamente conectada que está siendo anunciada mediante RIP.
* **`R(n)` (RIP normal):** Ruta normal aprendida de un vecino RIP adyacente.
* **`R(d)` (RIP default):** Ruta por defecto (`0.0.0.0/0`) aprendida o generada por RIP.
* **`Next Hop`:** IP del router de siguiente salto. Si indica `0.0.0.0` o `self`, la red es propia o directamente conectada.
* **`Metric`:** Coste en saltos (1 a 15). Si llega a `16`, la ruta se considera inalcanzable (infinito).
* **`From`:** IP del router vecino que nos envió la actualización de esta ruta.
* **`Time`:** Cuenta regresiva (inicia en 180 segundos). Si llega a 0 sin recibir actualizaciones, la ruta se marca con métrica 16.

##### C) En `show ip rip status`
* **`Sending updates every 30 seconds`:** Temporizador de actualizaciones periódicas de la tabla de enrutamiento.
* **`Timeout after 180 seconds`:** Tiempo de gracia antes de marcar una ruta como caída (métrica 16).
* **`Garbage collect after 120 seconds`:** Tiempo de permanencia de una ruta caída antes de ser borrada del sistema (sirve para notificar la caída a los vecinos).
* **`Routing for Networks`:** Subredes donde RIP está activo y habilitado para enviar/recibir.
* **`Passive Interface(s)`:** Interfaces en modo pasivo. Tienen desactivado el envío de actualizaciones automáticas por multicast, pero permiten envío por unicast (`neighbor`).
* **`Routing Information Sources`:**
  * **`Gateway`:** IP del router vecino que nos está enviando actualizaciones.
  * **`BadPackets / BadRoutes`:** Contador de paquetes corruptos o anuncios de rutas con errores (debe ser 0).
  * **`Last Update`:** Tiempo transcurrido (MM:SS) desde la última actualización recibida (debe actualizarse cada ~30s).

##### D) En `show interface`
* **`Interface ethX is up, line protocol is up`:**
  * El primer **`up`** indica que la interfaz está activa físicamente (Capa 1).
  * El segundo **`up`** (line protocol) indica que la trama de datos de red está activa (Capa 2).
* **`inet X.X.X.X/YY`:** Dirección IP principal de la interfaz y su máscara de red.
* **`secondary`:** Indica una IP adicional configurada en la misma interfaz. **¡Advertencia!** Si no es deliberado, provocará problemas de ruteo local.
* **`flags: <UP,BROADCAST,RUNNING,MULTICAST>`:** Indicadores lógicos del enlace (UP: interfaz habilitada, RUNNING: cable conectado).

---
### Guardar configuración

```bash
# Dentro de vtysh, SIEMPRE antes de salir:
write

# Desde bash:
vtysh -c "write"
```

### Prueba de desconexión Red6

```bash
# Estado antes (en router2):
vtysh -c "show ip route" > /tmp/antes.txt && vtysh -c "show ip rip" >> /tmp/antes.txt

# Desconectar:
ip link set eth3 down

# Ver logs:
journalctl -u frr -f

# Verificar cambio:
vtysh -c "show ip route" > /tmp/despues.txt
diff /tmp/antes.txt /tmp/despues.txt

# Probar (desde router1):
ping -c3 172.31.0.1          # sigue funcionando (via router4)
traceroute 172.31.0.254      # ahora va router2 → router4 → router3

# Reconectar:
ip link set eth3 up

# Verificar reconvergencia:
sleep 35 && vtysh -c "show ip rip"
traceroute 172.31.0.254      # vuelve a ir directo por router3 (Red6)
```

### Workstation 1

```bash
ip addr add 192.168.13.66/26 dev eth0
ip link set eth0 up
ip route add default via 192.168.13.65
apt install -y elinks
elinks http://info.cern.ch
```
### Habilitación de DNS
```bash
sudo systemctl disable --now systemd-resolved
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

---

## 16. Resolución de Problemas Comunes (Troubleshooting)

### IPs Duplicadas (Secundarias) en Interfaces
En FRR, si intentas cambiar la IP de una interfaz sin remover la anterior primero (`no ip address ...`), la nueva IP se guardará como **secundaria**. Esto genera problemas donde el router enruta los paquetes localmente a sí mismo en lugar de mandarlos por el enlace de red.
* **Cómo verificar:**
  ```bash
  vtysh -c "show interface ethX"
  ```
  Si ves una IP marcada como `secondary`, debes eliminarla.
* **Cómo solucionarlo (desde Bash de la máquina afectada):**
  ```bash
  sudo ip addr del <IP-incorrecta>/<prefijo> dev ethX
  sudo systemctl restart frr
  ```

### Convergencia lenta o nula al simular caídas (LXD Bridge)
Al apagar la interfaz de un extremo de una conexión P2P, el extremo opuesto puede seguir con su interfaz en estado `UP` debido al switch virtual (puente bridge). RIP no se enterará de la desconexión hasta que expire el timer de timeout (**180 segundos**).
* **Solución:** Apagar manualmente la interfaz en ambos extremos del enlace para forzar la convergencia inmediata por la ruta alternativa.