# 📘 Manual de Despliegue — Infraestructura Virtualizada Libretotal_corp

> **Proyecto:** Despliegue de una infraestructura virtualizada segura con Proxmox  
> **Autor:** Francisco Javier Carballares Sánchez · 2º SMR Online  
> **Versión:** 1.0 · Mayo 2026

---

## 📋 Índice

1. [Requisitos Previos](#1-requisitos-previos)
2. [Fase 1 — Instalación de Proxmox VE](#2-fase-1--instalación-de-proxmox-ve)
3. [Fase 2 — Creación de Máquinas Virtuales](#3-fase-2--creación-de-máquinas-virtuales)
4. [Fase 3 — Configuración de OPNsense (Firewall y VLANs)](#4-fase-3--configuración-de-opnsense-firewall-y-vlans)
5. [Fase 4 — VPN WireGuard](#5-fase-4--vpn-wireguard)
6. [Fase 5 — Active Directory (Windows Server 2022)](#6-fase-5--active-directory-windows-server-2022)
7. [Fase 6 — Federación de Clientes Linux (Debian 13)](#7-fase-6--federación-de-clientes-linux-debian-13)
8. [Fase 7 — Monitorización con Zabbix](#8-fase-7--monitorización-con-zabbix)
9. [Fase 8 — Proxmox Backup Server (PBS)](#9-fase-8--proxmox-backup-server-pbs)
10. [Verificación Final](#10-verificación-final)
11. [Resolución de Problemas](#11-resolución-de-problemas)

---

## 1. Requisitos Previos

### Hardware mínimo recomendado

| Componente | Mínimo | Usado en el proyecto |
|-----------|--------|----------------------|
| CPU | 4 núcleos con VT-x/AMD-V | Intel Xeon E5-2690 v3 (24 vCPU) |
| RAM | 16 GB | 125 GB |
| Almacenamiento | 256 GB SSD | SSD NVMe |
| Red | 1 NIC (recomendado 2) | 1 NIC física |

> ⚠️ La CPU debe tener **virtualización por hardware activada en BIOS/UEFI** (Intel VT-x o AMD-V).

### Software necesario para la gestión

- Navegador web moderno (Firefox, Chrome) en el equipo de gestión
- Cliente WireGuard para la VPN remota
- Conexión de red con salida a Internet en el host físico

### Esquema de máquinas virtuales del entorno

| VM ID | Nombre | SO | VLAN | IP Estática | RAM | CPU |
|-------|--------|----|------|-------------|-----|-----|
| 100 | OPNsense-GW | OPNsense 24 | WAN/troncal | 172.16.1.1 | 2 GB | 2 |
| 101 | SRV-AD01 | Windows Server 2022 | 30 | 172.16.30.10 | 4 GB | 4 |
| 102 | SRV-Zabbix | Debian 13 | 30 | 172.16.30.11 | 4 GB | 2 |
| 103 | PBS | Debian/LXC | 99 | 172.16.99.10 | 2 GB | 2 |
| 200 | WIN-100 | Windows 11 | 10 | DHCP | 4 GB | 2 |
| 201 | WIN-200 | Windows 11 | 10 | DHCP | 4 GB | 2 |
| 207 | WIN-300 | Windows 11 | 10 | DHCP | 4 GB | 2 |

---

## 2. Fase 1 — Instalación de Proxmox VE

### 2.1 Descarga e instalación

1. Descargar la ISO de Proxmox VE desde [https://www.proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)
2. Crear un USB booteable con Rufus o `dd`:
   ```bash
   dd if=proxmox-ve_8.x.iso of=/dev/sdX bs=4M status=progress
   ```
3. Arrancar el servidor desde el USB y seguir el asistente:
   - **Disco de instalación:** Seleccionar el SSD/NVMe principal
   - **País y zona horaria:** Europe/Madrid
   - **Contraseña root:** Establecer una contraseña segura
   - **IP de gestión:** `172.16.1.1/16` — Gateway: `10.10.0.1`

### 2.2 Primer acceso

Desde el navegador de tu equipo de gestión, accede a:
```
https://172.16.1.1:8006
```
- **Usuario:** `root`
- **Contraseña:** la configurada durante la instalación
- **Realm:** `Linux PAM`

> ⚠️ Acepta el certificado autofirmado en el navegador.

### 2.3 Configuración del bridge de red

En el nodo Proxmox, navega a **Datacenter → Nodo → Network** y configura el bridge troncal `vmbr0` sobre la interfaz física:

```
Interfaz física: enp3s0 (adaptar según tu hardware)
Bridge: vmbr0 (VLAN aware: SÍ)
IP: 10.10.0.2/24
Gateway: 10.10.0.1
```

---

## 3. Fase 2 — Creación de Máquinas Virtuales

### 3.1 Subir ISOs al almacenamiento

En Proxmox, ve a **local → ISO Images → Upload** y sube:
- `OPNsense-24.x-dvd-amd64.iso`
- `Windows Server 2022.iso`
- `debian-13-amd64-netinst.iso`
- `Windows 11.iso`

También descarga los **drivers VirtIO para Windows**:
```
https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/
```

### 3.2 Crear VM de OPNsense (VM 100)

1. **Crear VM** con los parámetros:
   - OS: Other
   - Discos: VirtIO SCSI, 32 GB
   - CPU: 2 cores, VirtIO
   - RAM: 2048 MB
2. **Añadir segunda NIC** (para WAN y LAN separadas si dispones de dos interfaces)
3. Asignar la NIC al bridge `vmbr0` con **VLAN tag vacío** (troncal)

### 3.3 Crear VM de Windows Server 2022 (VM 101)

1. OS: Windows 2022
2. Discos: VirtIO SCSI, 80 GB + añadir CD de VirtIO
3. CPU: 4 cores
4. RAM: 4096 MB
5. Red: `vmbr0`, VLAN tag `30`

### 3.4 Crear VM de Debian 13 / Zabbix (VM 102)

1. OS: Linux, kernel 6.x
2. Discos: VirtIO SCSI, 40 GB
3. CPU: 2 cores
4. RAM: 4096 MB
5. Red: `vmbr0`, VLAN tag `30`

---

## 4. Fase 3 — Configuración de OPNsense (Firewall y VLANs)

### 4.1 Instalación inicial

1. Arrancar la VM de OPNsense desde la ISO
2. En el asistente de instalación seleccionar:
   - Teclado: `es`
   - Modo: **ZFS** o **UFS** según preferencia
3. Una vez instalado, acceder a la consola y configurar la interfaz:
   - `WAN`: interfaz conectada a la red de tu router (IP de tu red doméstica o ISP)
   - `LAN`: interfaz virtual (VLAN troncal sobre `vmbr0`)

### 4.2 Acceso a la interfaz web

```
URL:  https://172.16.1.1   (o la IP LAN asignada)
User: root / admin
Pass: opnsense (cambiar en el primer acceso)
```

### 4.3 Configuración de VLANs

En **Interfaces → Other Types → VLAN**, crear las siguientes VLANs sobre la interfaz LAN (vtnet1 o similar):

| VLAN Tag | Nombre | Descripción |
|----------|--------|-------------|
| 10 | VLAN_CLIENTES | Estaciones de usuario |
| 30 | VLAN_SERVIDORES_INTERNOS | AD y Zabbix |
| 99 | VLAN_GESTION | Gestión PBS y Proxmox |

Asignar cada VLAN como interfaz en **Interfaces → Assignments** y configurar:

```
VLAN 10 → IP: 172.16.10.1/24 — DHCP habilitado (rango: 172.16.10.10–172.16.10.100)
VLAN 30 → IP: 172.16.30.1/24 — DHCP habilitado (rango: 172.16.30.10–172.16.30.50)
VLAN 99 → IP: 172.16.99.1/24 — DHCP habilitado (rango: 172.16.99.10–172.16.99.20)
```

### 4.4 Reglas de Firewall

Navega a **Firewall → Rules** y configura según el principio de **mínimo privilegio**:

**VLAN_CLIENTES (10):**
```
ALLOW   TCP/UDP  VLAN10 → VLAN30:53    (DNS)
ALLOW   TCP      VLAN10 → VLAN30:389,636 (LDAP/AD)
ALLOW   TCP      VLAN10 → VLAN30:445  (SMB)
ALLOW   ANY      VLAN10 → WAN          (Internet)
BLOCK   ANY      VLAN10 → VLAN30       (resto)
BLOCK   ANY      VLAN10 → VLAN99       (gestión)
```

**VLAN_SERVIDORES (30):**
```
ALLOW   ANY      VLAN30 → WAN          (actualizaciones)
ALLOW   TCP      VLAN30 → VLAN10:10050 (Zabbix polling)
BLOCK   ANY      VLAN30 → VLAN99
```

**VLAN_GESTION (99):**
```
ALLOW   ANY      VLAN99 → ANY          (gestión total)
```

### 4.5 DNS

En **Services → Unbound DNS → General**, activar y configurar:
- **Listen Interfaces:** All
- **Forward Zones:** `lab.libretotal.space → 172.16.30.10` (AD)

---

## 5. Fase 4 — VPN WireGuard

### 5.1 Instalación del plugin en OPNsense

```
System → Firmware → Plugins → Buscar "wireguard" → Instalar os-wireguard
```

### 5.2 Configurar el servidor WireGuard

En **VPN → WireGuard → Local**:

```
Nombre: WG_LAB
Escuchar en puerto: 51820
Dirección túnel: 10.20.20.1/24
```

Generar par de claves → guardar la clave pública.

### 5.3 Añadir un peer (cliente externo)

En **VPN → WireGuard → Peers**:

```
Nombre: Cliente_Movil
Clave pública: <pegar clave pública del cliente>
IPs permitidas: 10.20.20.2/32
```

### 5.4 Configuración en el cliente (móvil/PC externo)

Crear archivo de configuración WireGuard:

```ini
[Interface]
PrivateKey = <clave privada del cliente>
Address = 10.20.20.2/24
DNS = 172.16.30.10

[Peer]
PublicKey = <clave pública del servidor OPNsense>
Endpoint = <IP pública de tu router>:51820
AllowedIPs = 172.16.0.0/16, 10.20.20.0/24
PersistentKeepalive = 25
```

> ℹ️ Importar este archivo en la app WireGuard para iOS/Android o el cliente de escritorio.

### 5.5 Regla de firewall para WireGuard

En **Firewall → Rules → WAN**:
```
ALLOW UDP cualquier → <IP WAN>:51820
```

---

## 6. Fase 5 — Active Directory (Windows Server 2022)

### 6.1 Configuración inicial del servidor

1. Arrancar la VM `SRV-AD01` e instalar Windows Server 2022 (con GUI)
2. Configurar IP estática:
   ```
   IP:       172.16.30.10
   Máscara:  255.255.255.0
   Gateway:  172.16.30.1
   DNS:      127.0.0.1 (se auto-configura al instalar AD)
   ```
3. Cambiar el nombre del equipo a `SRV-AD01` y reiniciar.

### 6.2 Instalación del rol AD DS

En **Administrador del servidor → Agregar roles y características**:

1. Seleccionar **Servicios de dominio de Active Directory (AD DS)**
2. También instalar **DNS Server** cuando se solicite
3. Una vez instalado, hacer clic en **Promover este servidor a controlador de dominio**:
   ```
   Tipo de implementación: Agregar un bosque nuevo
   Nombre de dominio raíz: lab.libretotal.space
   Nivel funcional de bosque: Windows Server 2016 o superior
   Contraseña DSRM: <contraseña segura>
   ```
4. Revisar los requisitos previos → **Instalar** → El servidor se reiniciará automáticamente.

### 6.3 Estructura organizativa

En **Herramientas → Usuarios y equipos de Active Directory**, crear las siguientes OUs:

```
lab.libretotal.space
├── OU=Admin_Sistemas
│   └── adminjavi (administrador de dominio)
├── OU=Usuarios
│   ├── usuario1
│   └── usuario2
└── OU=Equipos
    ├── WIN-100
    ├── WIN-200
    └── WIN-300
```

### 6.4 Crear usuarios

```powershell
# Ejecutar en PowerShell como administrador de dominio
New-ADUser -Name "adminjavi" -SamAccountName "adminjavi" `
  -UserPrincipalName "adminjavi@lab.libretotal.space" `
  -Path "OU=Admin_Sistemas,DC=lab,DC=libretotal,DC=space" `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force) `
  -Enabled $true

Add-ADGroupMember -Identity "Domain Admins" -Members "adminjavi"
```

### 6.5 Unir clientes Windows al dominio

En los equipos Windows 11 (WIN-100, WIN-200, WIN-300):

1. Configurar DNS primario: `172.16.30.10`
2. **Sistema → Cambiar configuración → Cambiar → Dominio:** `lab.libretotal.space`
3. Introducir credenciales de administrador de dominio → Reiniciar

---

## 7. Fase 6 — Federación de Clientes Linux (Debian 13)

### 7.1 Preparación del cliente Debian

```bash
# Configurar DNS apuntando al AD
echo "nameserver 172.16.30.10" > /etc/resolv.conf
echo "search lab.libretotal.space" >> /etc/resolv.conf

# Verificar resolución
ping lab.libretotal.space
```

### 7.2 Instalar paquetes necesarios

```bash
apt update
apt install -y realmd sssd sssd-tools adcli samba-common-bin oddjob oddjob-mkhomedir packagekit
```

### 7.3 Unirse al dominio

```bash
# Descubrir el dominio
realm discover lab.libretotal.space

# Unirse al dominio
realm join lab.libretotal.space -U adminjavi

# Verificar unión
realm list
```

### 7.4 Configurar SSSD

Editar `/etc/sssd/sssd.conf`:

```ini
[sssd]
domains = lab.libretotal.space
config_file_version = 2
services = nss, pam

[domain/lab.libretotal.space]
ad_domain = lab.libretotal.space
krb5_realm = LAB.LIBRETOTAL.SPACE
realmd_tags = manages-system joined-with-adcli
cache_credentials = True
id_provider = ad
krb5_store_password_if_offline = True
default_shell = /bin/bash
ldap_id_mapping = True
use_fully_qualified_names = False
fallback_homedir = /home/%u@%d
access_provider = ad
```

```bash
systemctl restart sssd
```

### 7.5 Habilitar creación automática de perfiles

```bash
pam-auth-update --enable mkhomedir

# Permitir login al dominio en el equipo
realm permit --all
```

### 7.6 Verificación

```bash
id adminjavi@lab.libretotal.space
getent passwd adminjavi
```

---

## 8. Fase 7 — Monitorización con Zabbix

### 8.1 Instalación del servidor Zabbix en Debian 13

```bash
# Añadir repositorio oficial de Zabbix 7.0 LTS
wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest+debian13_all.deb
dpkg -i zabbix-release_latest+debian13_all.deb
apt update

# Instalar servidor, frontend y agente
apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent

# Instalar MariaDB
apt install -y mariadb-server
```

### 8.2 Configurar la base de datos

```bash
mysql -uroot -p <<EOF
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'zabbix_pass';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
SET GLOBAL log_bin_trust_function_creators = 1;
FLUSH PRIVILEGES;
EOF

# Importar esquema
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix

mysql -uroot -e "SET GLOBAL log_bin_trust_function_creators = 0;"
```

Editar `/etc/zabbix/zabbix_server.conf`:
```
DBPassword=zabbix_pass
```

### 8.3 Iniciar servicios

```bash
systemctl enable --now zabbix-server zabbix-agent apache2
```

Acceder al asistente web en: `http://172.16.30.11:8080`  
Usuario por defecto: `Admin` / Contraseña: `zabbix`

### 8.4 Instalar agente en Windows Server (AD)

1. Descargar **Zabbix Agent 2 v7.0.23** para Windows desde:  
   [https://www.zabbix.com/download_agents](https://www.zabbix.com/download_agents)
2. Descomprimir en `C:\Zabbix\`
3. Editar `C:\Zabbix\zabbix_agent2.conf`:
   ```
   Server=172.16.30.11
   ServerActive=172.16.30.11
   Hostname=SERV-AD01
   ```
4. Abrir **PowerShell como administrador**:
   ```powershell
   # Instalar el servicio
   C:\Zabbix\zabbix_agent2.exe --config C:\Zabbix\zabbix_agent2.conf --install

   # Abrir puerto en el firewall de Windows
   New-NetFirewallRule -DisplayName "Zabbix Agent" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 10050

   # Arrancar el servicio
   Start-Service "Zabbix Agent 2"
   ```

### 8.5 Instalar agente en OPNsense

```
System → Firmware → Plugins → Buscar "zabbix-agent" → Instalar os-zabbix-agent
Services → Zabbix Agent → Settings:
  - Habilitado: ✓
  - Nombre host: OPNsense-GW
  - Servidores Zabbix: 172.16.30.11
  - Puerto de escucha: 10050
```

### 8.6 Aprovisionar hosts en Zabbix

En la consola web (`http://172.16.30.11:8080`):

1. **Recopilación de datos → Equipos → Crear equipo**
2. Para cada host configurar:
   - Nombre exacto (igual al `Hostname` del agente)
   - Plantilla adecuada:
     - Linux: `Linux by Zabbix agent`
     - Windows Server: `Windows by Zabbix agent`
     - OPNsense: `FreeBSD by Zabbix agent`
   - Interfaz: IP del equipo, puerto `10050`
3. Verificar que el indicador **ZBX** aparece en **verde** en la lista de equipos.

---

## 9. Fase 8 — Proxmox Backup Server (PBS)

### 9.1 Despliegue del contenedor LXC

En Proxmox, crear un contenedor LXC con:
- Plantilla: Debian 12 (del repositorio de plantillas de Proxmox)
- RAM: 2 GB, CPU: 2 cores
- Disco: 100+ GB (para almacenar backups)
- Red: `vmbr0`, VLAN tag `99`, IP `172.16.99.10/24`, Gateway `172.16.99.1`

### 9.2 Instalar PBS en el contenedor

```bash
# Dentro del contenedor LXC
echo "deb http://download.proxmox.com/debian/pbs bookworm pbs-no-subscription" > /etc/apt/sources.list.d/pbs.list
wget -qO - https://download.proxmox.com/debian/proxmox-release-bookworm.gpg | apt-key add -
apt update && apt install -y proxmox-backup-server

systemctl enable --now proxmox-backup
```

Acceder a la GUI de PBS en: `https://172.16.99.10:8007`

### 9.3 Crear almacén de datos (Datastore)

En la GUI de PBS:  
**Administración → Almacén de datos → Agregar**:
```
Nombre: BackupsLab
Directorio de respaldo: /mnt/backups
```

### 9.4 Vincular PBS con Proxmox VE

En la GUI de Proxmox VE:  
**Datacenter → Storage → Add → Proxmox Backup Server**:
```
ID:          PBS-BackupsLab
Server:      172.16.99.10
Username:    admin@pbs
Password:    <contraseña de PBS>
Datastore:   BackupsLab
Fingerprint: <copiar desde PBS: Dashboard → Fingerprint>
```

### 9.5 Programar backups automáticos

En Proxmox VE:  
**Datacenter → Backup → Add**:
```
Almacenamiento: PBS-BackupsLab
Programación:   0 2 * * *  (todos los días a las 2:00)
Modo:           Snapshot
VMs:            All
```

---

## 10. Verificación Final

Ejecutar las siguientes pruebas para validar el despliegue:

### Segmentación de red

```bash
# Desde WIN-100 (VLAN 10) — deben FALLAR (aislamiento correcto)
ping 172.16.30.10   # No debe llegar al AD
ping 172.16.99.1    # No debe llegar a gestión

# Desde WIN-100 — debe FUNCIONAR
ping 8.8.8.8        # Salida a Internet
```

### DNS centralizado

```bash
nslookup lab.libretotal.space 172.16.30.10
# Debe resolver correctamente
```

### VPN WireGuard

1. Conectar el cliente desde red 4G/5G externa
2. Verificar en OPNsense: **VPN → WireGuard → Status** → Handshake reciente
3. Hacer ping a `172.16.30.10` desde el túnel

### Monitorización

1. En Zabbix, comprobar que todos los hosts muestran indicador **ZBX en verde**
2. Apagar la VM `SRV-AD01` y verificar que aparece una alerta en el dashboard de Zabbix

### Backup y restauración

```bash
# Desde Proxmox VE:
# 1. Eliminar una VM de prueba (WIN-300)
# 2. Restaurar desde PBS → Storage → PBS-BackupsLab → Restore
# 3. Verificar que la VM arranca correctamente
```

---

## 11. Resolución de Problemas

### OPNsense no enruta entre VLANs

- Verificar que las reglas de firewall están en el orden correcto (las ALLOW antes que las DENY)
- Comprobar que el bridge `vmbr0` tiene **VLAN aware** activado en Proxmox
- Revisar que el VLAN tag de cada VM coincide con la VLAN configurada en OPNsense

### WireGuard no establece handshake

- Verificar que el puerto `51820/UDP` está abierto en el router doméstico (port forwarding)
- Ajustar el MTU a `1280` si hay problemas con el doble encapsulamiento PPPoE:
  ```
  VPN → WireGuard → Local → MTU: 1280
  ```

### Debian no se une al dominio

- Comprobar que el DNS apunta a `172.16.30.10` y resuelve `lab.libretotal.space`
- Verificar sincronización NTP entre el cliente y el AD (diferencia < 5 minutos)
- Reinstalar `krb5-user` si hay errores de Kerberos:
  ```bash
  apt install --reinstall krb5-user
  ```

### Agente Zabbix no aparece en verde

- Verificar que el servicio está corriendo: `systemctl status zabbix-agent2`
- Comprobar que el puerto `10050` está abierto en el firewall del equipo
- Confirmar que el `Hostname` en el `.conf` coincide exactamente con el registrado en Zabbix

---

*Documento elaborado por Francisco Javier Carballares Sánchez — TFG 2º SMR Online 2025-2026*
