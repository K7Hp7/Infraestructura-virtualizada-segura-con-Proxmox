# 👤 Manual de Usuario — Infraestructura Libretotal_corp

> **Proyecto:** Despliegue de una infraestructura virtualizada segura con Proxmox  
> **Autor:** Francisco Javier Carballares Sánchez · 2º SMR Online  
> **Versión:** 1.0 · Marzo 2026

---

## 📋 Índice

1. [Acceso al Hipervisor — Proxmox VE](#1-acceso-al-hipervisor--proxmox-ve)
2. [Acceso al Firewall — OPNsense](#2-acceso-al-firewall--opnsense)
3. [Conexión VPN Remota — WireGuard](#3-conexión-vpn-remota--wireguard)
4. [Inicio de Sesión con Usuario de Dominio](#4-inicio-de-sesión-con-usuario-de-dominio)
5. [Panel de Monitorización — Zabbix](#5-panel-de-monitorización--zabbix)
6. [Backup y Restauración — PBS](#6-backup-y-restauración--pbs)
7. [Gestión de Usuarios en Active Directory](#7-gestión-de-usuarios-en-active-directory)
8. [Operaciones de Mantenimiento Habituales](#8-operaciones-de-mantenimiento-habituales)

---

## 1. Acceso al Hipervisor — Proxmox VE

### ¿Qué es Proxmox VE?

Proxmox VE es el **hipervisor** que gestiona todas las máquinas virtuales del entorno. Desde aquí puedes arrancar, parar, crear y supervisar las VMs.

### Acceso

| Campo | Valor |
|-------|-------|
| URL | `https://10.10.0.2:8006` |
| Usuario | `root` |
| Realm | `Linux PAM standard authentication` |

> ⚠️ Es necesario estar en la **red de gestión (VLAN 99)** o conectado mediante **VPN WireGuard** para acceder.

### Operaciones básicas con VMs

**Arrancar una VM:**
1. En el panel izquierdo, hacer clic sobre la VM deseada
2. Hacer clic en **Start** (▶️) en la barra superior

**Apagar una VM de forma controlada:**
1. Seleccionar la VM
2. Clic en **Shutdown** (no usar *Stop* si la VM tiene SO instalado)

**Ver el estado de los recursos:**
- En la vista del nodo, el panel **Summary** muestra CPU, RAM y almacenamiento en tiempo real

**Acceder a la consola de una VM:**
1. Seleccionar la VM
2. Clic en **Console** → se abre una terminal gráfica en el navegador

---

## 2. Acceso al Firewall — OPNsense

### ¿Qué es OPNsense?

OPNsense es el **firewall y router** del entorno. Gestiona el tráfico entre VLANs, las reglas de seguridad, el DHCP, el DNS y la VPN.

### Acceso

| Campo | Valor |
|-------|-------|
| URL | `https://172.16.99.1` (desde VLAN 99) o `https://10.10.0.1` |
| Usuario | `root` o `admin` |
| Contraseña | La configurada durante la instalación |

### Ver el estado del firewall

En el **Dashboard** se muestra:
- Interfaces de red activas y sus IPs
- Estado de los servicios (WireGuard, DHCP, DNS)
- Tráfico en tiempo real por interfaz
- Alertas activas

### Ver las concesiones DHCP

**Services → DHCPv4 → Leases**  
Aquí puedes ver qué IP ha sido asignada a cada equipo del entorno.

### Ver el estado de la VPN

**VPN → WireGuard → Status**  
Se muestra el estado del túnel y la fecha del último *handshake* de cada cliente.

### Añadir una regla de firewall

1. **Firewall → Rules → [seleccionar interfaz]**
2. Clic en el botón **+** (Agregar)
3. Configurar:
   - **Action:** Pass / Block
   - **Protocol:** TCP, UDP, ICMP, Any
   - **Source:** Red o IP origen
   - **Destination:** Red o IP destino
   - **Destination port:** Puerto específico o Any
4. Guardar y hacer clic en **Apply changes**

---

## 3. Conexión VPN Remota — WireGuard

### ¿Para qué sirve la VPN?

WireGuard permite **acceder a la red interna de Libretotal_corp desde cualquier lugar** (casa, móvil, teletrabajo) de forma cifrada y segura.

### Configuración en el cliente

#### En dispositivo móvil (iOS / Android)

1. Instalar la app **WireGuard** desde App Store o Google Play
2. Abrir la app → botón **+** → **Escanear código QR** o **Importar desde archivo**
3. Activar el interruptor de la conexión → La VPN está activa

#### En Windows / macOS / Linux

1. Instalar el cliente WireGuard desde [https://www.wireguard.com/install/](https://www.wireguard.com/install/)
2. Abrir WireGuard → **Importar túnel(es) desde archivo**
3. Seleccionar el archivo `.conf` proporcionado por el administrador
4. Hacer clic en **Activar**

### Verificar la conexión

Una vez activa la VPN podrás:
- Acceder a `https://10.10.0.2:8006` (Proxmox) desde cualquier red
- Hacer ping a `172.16.30.10` (servidor AD)
- Iniciar sesión en la consola de Zabbix

### Indicadores de estado

| Estado | Descripción |
|--------|-------------|
| 🟢 Conectado | Túnel activo, tráfico bidireccional |
| 🟡 Intentando | Buscando el servidor, sin handshake |
| 🔴 Desconectado | VPN inactiva |

---

## 4. Inicio de Sesión con Usuario de Dominio

### Equipos Windows 11 (WIN-100, WIN-200, WIN-300)

Los equipos cliente están unidos al dominio `lab.libretotal.space`. Para iniciar sesión:

1. En la pantalla de login, escribir el usuario con el formato:
   ```
   LAB\nombredeusuario
   ```
   o bien:
   ```
   nombredeusuario@lab.libretotal.space
   ```
2. Introducir la contraseña de dominio
3. El perfil se creará automáticamente en el primer inicio de sesión

### Equipos Debian 13

Los clientes Linux también están federados con el dominio. Para iniciar sesión por SSH o localmente:

```bash
ssh adminjavi@lab.libretotal.space@<IP del equipo>
# o simplemente:
ssh adminjavi@<IP del equipo>
```

### Cambiar contraseña de usuario de dominio

#### Desde Windows
- Pulsar `Ctrl+Alt+Supr` → **Cambiar contraseña**

#### Desde PowerShell (administrador)
```powershell
Set-ADAccountPassword -Identity "nombredeusuario" `
  -NewPassword (ConvertTo-SecureString "NuevaContraseña!" -AsPlainText -Force) `
  -Reset
```

---

## 5. Panel de Monitorización — Zabbix

### Acceso

| Campo | Valor |
|-------|-------|
| URL | `http://172.16.30.11:8080` |
| Usuario | `Admin` |
| Contraseña | `zabbix` (cambiar tras primer acceso) |

### Vista principal — Dashboard

El dashboard muestra:
- **Estado general** de todos los equipos monitorizados
- **Problemas activos** en tiempo real
- **Gráficas de rendimiento** (CPU, RAM, red)
- **Mapa topológico** de la infraestructura

### Interpretar el indicador de disponibilidad

| Color | Significado |
|-------|-------------|
| 🟢 Verde (ZBX) | Agente activo y respondiendo |
| 🔴 Rojo (ZBX) | Agente sin respuesta o host apagado |
| ⚪ Gris | Sin confirmar todavía |

### Ver problemas activos

**Monitorización → Problemas**  
Lista todos los eventos de alerta activos. Haciendo clic en cada uno se obtienen detalles del evento, hora de inicio y duración.

### Ver métricas de un equipo concreto

1. **Monitorización → Equipos** → Seleccionar un equipo
2. Pestaña **Últimos datos**: métricas en tiempo real
3. Pestaña **Gráficas**: historial visual de CPU, RAM, disco y red

### Crear un widget de monitorización personalizado

1. En el Dashboard, clic en **Editar tablero**
2. Clic en **Agregar widget**
3. Seleccionar tipo: *Gráfico*, *Indicador*, *Mapa*, etc.
4. Configurar los hosts y métricas a mostrar → **Aplicar**

---

## 6. Backup y Restauración — PBS

### Acceso a Proxmox Backup Server

| Campo | Valor |
|-------|-------|
| URL | `https://172.16.99.10:8007` |
| Usuario | `admin@pbs` |
| Contraseña | La configurada durante la instalación |

### Ver los backups disponibles

En la GUI de PBS:  
**Administración → Almacén de datos → BackupsLab → Contenido**

Se lista cada snapshot disponible con su fecha, tamaño y estado de verificación.

### Restaurar una VM desde PBS (desde Proxmox VE)

1. Acceder a Proxmox VE (`https://10.10.0.2:8006`)
2. En el panel izquierdo, ir a **Datacenter → Storage → PBS-BackupsLab**
3. Seleccionar el snapshot de la VM a restaurar
4. Clic en **Restore**:
   - Seleccionar el nodo de destino
   - Asignar el mismo VM ID (o uno nuevo)
   - Activar **Start after restore** si se desea arrancar automáticamente
5. Confirmar → La restauración comenzará y se puede seguir el progreso en la pestaña **Tasks**

### Crear un backup manual de una VM

En Proxmox VE:
1. Seleccionar la VM en el panel izquierdo
2. Pestaña **Backup** → **Backup Now**
3. Seleccionar almacenamiento: `PBS-BackupsLab`
4. Modo: `Snapshot`
5. Clic en **Backup**

---

## 7. Gestión de Usuarios en Active Directory

### Acceso a la consola de AD

En el servidor `SRV-AD01` (172.16.30.10), abrir:
- **Inicio → Herramientas administrativas → Usuarios y equipos de Active Directory**

### Crear un nuevo usuario

1. Expandir el dominio `lab.libretotal.space`
2. Clic derecho sobre la OU de destino (ej. `Usuarios`) → **Nuevo → Usuario**
3. Rellenar:
   - Nombre y apellidos
   - Nombre de inicio de sesión (ej. `juan.garcia`)
4. Establecer contraseña → Marcar **El usuario debe cambiar la contraseña en el siguiente inicio de sesión**
5. Finalizar

### Desactivar un usuario

1. Clic derecho sobre el usuario → **Deshabilitar cuenta**

> ℹ️ Es preferible **deshabilitar** en lugar de eliminar para mantener el historial y poder recuperarlo.

### Restablecer contraseña

1. Clic derecho sobre el usuario → **Restablecer contraseña**
2. Introducir la nueva contraseña
3. Marcar **El usuario debe cambiar la contraseña** → Aceptar

### Ver en qué equipos inició sesión un usuario

```powershell
# En PowerShell (en SRV-AD01)
Get-ADUser -Identity "nombredeusuario" -Properties LastLogonDate | Select-Object Name, LastLogonDate
```

---

## 8. Operaciones de Mantenimiento Habituales

### Actualizar OPNsense

```
System → Firmware → Updates → Check for Updates → Update
```
> ⚠️ Realizar siempre un snapshot de la VM antes de actualizar.

### Actualizar Proxmox VE

```bash
# Desde la shell del nodo Proxmox
apt update && apt dist-upgrade
```

### Actualizar el servidor Debian (Zabbix)

```bash
apt update && apt upgrade -y
systemctl restart zabbix-server zabbix-agent
```

### Hacer snapshot manual de una VM en Proxmox

1. Seleccionar la VM
2. Pestaña **Snapshots** → **Take Snapshot**
3. Dar un nombre descriptivo (ej. `pre-actualizacion-2026-04`)
4. Marcar **Include RAM** si se desea guardar el estado de la memoria

### Añadir un nuevo cliente al dominio

1. En el nuevo equipo Windows, configurar DNS primario: `172.16.30.10`
2. **Sistema → Configuración avanzada → Cambiar → Dominio:** `lab.libretotal.space`
3. Credenciales: `LAB\adminjavi` y la contraseña de administrador de dominio
4. Reiniciar → El equipo aparecerá en la OU `Equipos` de AD

### Revocar acceso VPN a un usuario

1. En OPNsense: **VPN → WireGuard → Peers**
2. Seleccionar el peer del usuario
3. Desmarcar **Habilitado** → **Guardar → Aplicar cambios**

---

*Documento elaborado por Francisco Javier Carballares Sánchez — TFG 2º SMR Online 2025-2026*
