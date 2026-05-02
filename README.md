# 🖥️ Infraestructura Virtualizada Segura con Proxmox

> **TFG — 2º SMR Online | Francisco Javier Carballares Sánchez**  
> Tutor: Iker Iturbe Azcorra · Fecha: Mayo 2026

---

## 📋 Descripción del Proyecto

Este proyecto consiste en el **diseño e implementación de una infraestructura de red corporativa completa** para la empresa simulada **Libretotal_corp**, desplegada sobre un único nodo físico de bajo coste y gestionada íntegramente con software de código abierto.

El entorno virtualizado reproduce los servicios tecnológicos fundamentales de una pequeña o mediana empresa: identidad centralizada, conectividad remota segura, monitorización en tiempo real y continuidad de negocio.

---

## 🏗️ Arquitectura de la Solución

```
Internet
    │
┌───▼──────────────────────────────────────────┐
│          OPNsense (Firewall Perimetral)       │
│     VPN WireGuard · VLANs 802.1Q · DHCP      │
└───┬──────────────┬──────────────┬─────────────┘
    │              │              │
┌───▼───┐    ┌────▼────┐   ┌────▼────┐
│VLAN 10│    │ VLAN 30 │   │ VLAN 99 │
│Clientes│   │Servidores│  │ Gestión │
│172.16. │   │172.16.  │   │172.16.  │
│10.0/24 │   │30.0/24  │   │99.0/24  │
└────────┘   └────┬────┘   └────┬────┘
                  │             │
          ┌───────┴──┐    ┌─────┴──────┐
          │  AD DS   │    │    PBS     │
          │Win Server│    │(Backup Srv)│
          └──────────┘    └────────────┘
                  │
          ┌───────┴──────┐
          │    Zabbix    │
          │ (Monitoring) │
          └──────────────┘

     Todo sobre Proxmox VE (Hipervisor)
```

---

## 🛠️ Stack Tecnológico

| Categoría | Tecnología | Versión |
|-----------|-----------|---------|
| **Hipervisor** | Proxmox VE | 8.x |
| **Firewall / Router** | OPNsense | 24.x |
| **VPN** | WireGuard | — |
| **Directorio Activo** | Windows Server 2022 (AD DS) | — |
| **Monitorización** | Zabbix | 7.4 LTS |
| **Copias de seguridad** | Proxmox Backup Server (PBS) | — |
| **SO Clientes** | Windows 11, Debian 13 | — |
| **BBDD Zabbix** | MariaDB | — |

---

## 🌐 Plan de Direccionamiento

| VLAN | Nombre | Red | Gateway | Función |
|------|--------|-----|---------|---------|
| 10 | VLAN_CLIENTES | 172.16.10.0/24 | 172.16.10.1 | Estaciones de usuario |
| 30 | VLAN_SERVIDORES_INTERNOS | 172.16.30.0/24 | 172.16.30.1 | AD, Zabbix |
| 99 | VLAN_GESTION | 172.16.99.0/24 | 172.16.99.1 | Proxmox, PBS |
| — | VPN WireGuard | 10.20.20.0/24 | — | Acceso remoto |

---

## ✅ Requisitos Funcionales Cumplidos (Matriz RFTP)

- [x] **R01** — Acceso a la GUI de Proxmox VE (CPU/RAM verificados)
- [x] **R02** — Segmentación VLAN con aislamiento inter-VLAN
- [x] **R03** — Autenticación de usuarios de dominio en Windows 11
- [x] **R04** — VPN WireGuard operativa desde red 4G/5G externa
- [x] **R05** — Alertas de monitorización en Zabbix ante caída de servicio
- [x] **R06** — Restauración de VM desde snapshot en PBS

---

## 📁 Estructura del Repositorio

```
📦 tfg-proxmox-libretotal/
├── 📄 README.md                    ← Este archivo
├── 📘 MANUAL_DESPLIEGUE.md         ← Guía de instalación paso a paso
├── 👤 MANUAL_USUARIO.md            ← Guía de uso de la infraestructura
├── 📊 presentacion/
│   └── TFG_Presentacion.pptx       ← Diapositivas de defensa
└── 📝 memoria/
    └── TFG_SMR_Francisco_Javier_Carballares.docx
```

---

## 🚀 Inicio Rápido

Consulta el **[Manual de Despliegue](./MANUAL_DESPLIEGUE.md)** para reproducir el entorno completo desde cero.

---

## 👤 Autor

**Francisco Javier Carballares Sánchez**  
2º SMR Online · Curso 2025–2026  
Tutor: Iker Iturbe Azcorra

---

## 📜 Licencia

Este proyecto tiene carácter académico. Todo el software utilizado es de código abierto o libre uso educativo.
