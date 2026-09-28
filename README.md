# Laboratorio de Virtualización: Windows Server 2022 (DHCP + IIS) y CachyOS

Laboratorio académico del curso de **Sistemas Operativos** (Ingeniería en Sistemas, Universidad Mariano Gálvez, 2026).

Se desplegó un entorno cliente-servidor virtualizado con **VMware Workstation 17**, compuesto por un host físico y dos máquinas virtuales: un servidor **Windows Server 2022** con los servicios **DHCP** e **IIS**, y un cliente **CachyOS Linux**. El servidor web se publicó además fuera de la red local mediante una VPN mesh con **Tailscale**.

> **Mi rol en el equipo:** despliegue y configuración base de Windows Server (instalación, red, firewall y servicios).

---

## Contenido

- [Tecnologías utilizadas](#tecnologías-utilizadas)
- [1. Configuración de las máquinas virtuales](#1-configuración-de-las-máquinas-virtuales)
- [2. Instalación de Windows Server 2022](#2-instalación-de-windows-server-2022)
- [3. Configuración post-instalación](#3-configuración-post-instalación)
- [4. Servicio DHCP](#4-servicio-dhcp)
- [5. Servicio IIS](#5-servicio-iis)
- [6. Acceso remoto por VPN (Tailscale)](#6-acceso-remoto-por-vpn-tailscale)
- [7. Instalación de CachyOS (cliente)](#7-instalación-de-cachyos-cliente)
- [8. Pruebas y resultados](#8-pruebas-y-resultados)
- [Problemas encontrados y soluciones](#problemas-encontrados-y-soluciones)
- [Limitaciones y mejoras futuras](#limitaciones-y-mejoras-futuras)
- [Lo que aprendí](#lo-que-aprendí)

---

| Equipo | Sistema operativo | Red | Dirección IP |
|---|---|---|---|
| Host | Windows 11 | VMnet1 (Host-only) | `192.168.57.1` |
| SRV-SO-2026 | Windows Server 2022 Standard Evaluation | Host-only + NAT | `192.168.57.10` (estática) · `100.105.131.15` (Tailscale) |
| cachyos-x8664 | CachyOS (base Arch Linux) | Host-only | `192.168.57.129` (asignada por DHCP) |

**Red del laboratorio:** `192.168.57.0/24` · Máscara `255.255.255.0` · Puerta de enlace `192.168.57.1`

---

## Tecnologías utilizadas

| Categoría | Herramientas |
|---|---|
| Virtualización | VMware Workstation Pro 17 |
| Sistemas operativos | Windows Server 2022 (Desktop Experience), CachyOS con KDE Plasma |
| Servicios | DHCP Server, Internet Information Services (IIS) 10 |
| Red | Redes virtuales Host-only y NAT, TCP/IP, Tailscale (VPN WireGuard) |
| Línea de comandos | PowerShell, CMD, fish shell, `nmcli`, `ip`, `curl` |

---

## 1. Configuración de las máquinas virtuales

### Servidor: SRV-SO-2026

| Parámetro | Valor | Justificación |
|---|---|---|
| Configuración | Custom (advanced) | Control total del hardware virtual |
| Firmware | UEFI + Secure Boot | Arranque seguro y más rápido que BIOS |
| CPU | 2 núcleos | Suficiente para servicios de red |
| RAM | ~3.7 GB | Desktop Experience requiere más memoria que Server Core |
| Disco | 60 GB NVMe, dividido en varios archivos | Tamaño recomendado por VMware para Windows Server 2022 |
| Controlador SCSI | LSI Logic SAS | Recomendado para Windows Server 2022 |
| Red | Host-only (VMnet1) + NAT (añadido después) | Red aislada del laboratorio + salida a Internet |

### Cliente: cachyos-x8664

| Parámetro | Valor | Justificación |
|---|---|---|
| Firmware | UEFI | Evolución de BIOS, arranque más rápido |
| CPU | 4 núcleos | No superar el 50 % de los núcleos del host |
| RAM | 4 GB | Entorno gráfico KDE Plasma |
| Disco | 45 GB SCSI, archivo único | CachyOS requiere mínimo 30 GB |
| Controlador SCSI | Paravirtualized SCSI | Mayor rendimiento de E/S con menor uso de CPU |
| Red | Host-only (VMnet1) | Misma red que el servidor |


---

## 2. Instalación de Windows Server 2022

1. Descargar la ISO de evaluación (180 días) desde el [Evaluation Center de Microsoft](https://www.microsoft.com/es-es/evalcenter/evaluate-windows-server-2022).
2. Montar la ISO en la unidad CD/DVD de la VM y arrancar.
3. Seleccionar idioma, formato de hora (Español - Guatemala) y teclado (Latinoamericano).
4. Elegir **Windows Server 2022 Standard Evaluation (Desktop Experience)**.
   - Se eligió la interfaz gráfica por ser la primera interacción con el sistema. En producción, **Server Core** reduce el consumo de recursos y la superficie de ataque.
5. Instalación **personalizada** sobre el disco sin asignar.
6. Crear la contraseña del usuario `Administrador`.
7. Ajustar zona horaria a **(UTC-06:00) América Central**.

<!-- ![Selección de edición](capturas/02-edicion.png) -->

---

## 3. Configuración post-instalación

### 3.1 Cambiar el nombre del servidor

**Administrador del servidor → Servidor local → Nombre de equipo → Cambiar** → `SRV-SO-2026` → Reiniciar.

```powershell
# Alternativa por PowerShell
Rename-Computer -NewName "SRV-SO-2026" -Restart
```

### 3.2 Revisar el estado de la red

```powershell
hostname
Get-NetAdapter
ipconfig
```

Desde el host se revisó el rango DHCP que VMware entrega en VMnet1 (**Edit → Virtual Network Editor → DHCP Settings**): `192.168.57.128 – 192.168.57.254`. La IP fija del servidor debe quedar **fuera** de ese rango.

### 3.3 Asignar IP estática

| Campo | Valor |
|---|---|
| Dirección IP | `192.168.57.10` |
| Máscara | `255.255.255.0` |
| Puerta de enlace | `192.168.57.1` |
| DNS preferido | `127.0.0.1` |
| DNS alternativo | `192.168.57.1` |

```powershell
# Alternativa por PowerShell
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.57.10 -PrefixLength 24 -DefaultGateway 192.168.57.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1,192.168.57.1
```

### 3.4 Permitir ping en el firewall

Windows Defender Firewall bloquea por defecto las solicitudes ICMP entrantes, por lo que el servidor no respondía al ping.

```powershell
# Crear regla para permitir ping (ICMPv4 Echo Request)
New-NetFirewallRule -DisplayName "Permitir ICMPv4-In (Ping)" `
  -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow

# Alternativa: habilitar la regla predefinida
Enable-NetFirewallRule -Name "FPS-ICMP4-ERQ-In"
```

---

## 4. Servicio DHCP

DHCP automatiza la entrega de parámetros de red (IP, máscara, puerta de enlace, DNS) mediante el proceso **DORA**: *Discover → Offer → Request → Acknowledge*. Usa los puertos UDP 67 (servidor) y 68 (cliente).

### 4.1 Desactivar el DHCP de VMware

Para evitar dos servidores DHCP en la misma red:

1. VMware → **Edit → Virtual Network Editor → Change Settings**.
2. Seleccionar **VMnet1 (Host-only)**.
3. Desmarcar **Use local DHCP service to distribute IP addresses to VMs**.
4. Aplicar y aceptar.

### 4.2 Instalar el rol

**Administrador del servidor → Agregar roles y características → Servidor DHCP** → Instalar → **Completar configuración de DHCP** → Confirmar (crea los grupos de seguridad *Administradores de DHCP* y *Usuarios de DHCP*).

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```

### 4.3 Crear el ámbito

**Herramientas → DHCP → IPv4 → Ámbito nuevo**

| Parámetro | Valor |
|---|---|
| Nombre | `LAB-SO-HostOnly` |
| Rango | `192.168.57.100` – `192.168.57.200` |
| Máscara | `255.255.255.0` (/24) |
| Exclusiones | `192.168.57.100` – `192.168.57.110` (reservado para IP fijas) |
| Rango efectivo | `192.168.57.111` – `192.168.57.200` (90 direcciones) |
| Concesión | 8 horas (se redujo de 8 días para observar renovaciones) |
| Puerta de enlace (003) | `192.168.57.1` |
| Dominio / DNS (006) | `lab.local` / `192.168.57.10` |
| Estado | Activado |

```powershell
# Alternativa por PowerShell
Add-DhcpServerv4Scope -Name "LAB-SO-HostOnly" -StartRange 192.168.57.100 -EndRange 192.168.57.200 `
  -SubnetMask 255.255.255.0 -LeaseDuration 08:00:00 -State Active
Add-DhcpServerv4ExclusionRange -ScopeId 192.168.57.0 -StartRange 192.168.57.100 -EndRange 192.168.57.110
Set-DhcpServerv4OptionValue -ScopeId 192.168.57.0 -Router 192.168.57.1 -DnsDomain "lab.local" -DnsServer 192.168.57.10
```

---

## 5. Servicio IIS

IIS es el servidor web de Microsoft. Soporta HTTP/HTTPS/FTP, tiene una arquitectura modular, aísla sitios mediante *Application Pools* y permite varios sitios en un mismo servidor mediante *bindings* (IP + puerto + nombre de host).

### 5.1 Instalar el rol

**Agregar roles y características → Servidor web (IIS)** → Instalar.

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
```

### 5.2 Verificar y publicar una página propia

1. Abrir `http://localhost` en el servidor: aparece la página por defecto de IIS.
2. Ir a `C:\inetpub\wwwroot` y agregar un `index.html` personalizado.
3. Recargar `http://localhost` para confirmar el cambio.

---

## 6. Acceso remoto por VPN (Tailscale)

Para acceder al sitio desde dispositivos fuera de la red local, sin abrir puertos en el router, se usó **Tailscale**, una VPN mesh basada en WireGuard.

1. Apagar el servidor y agregar un **segundo adaptador de red en modo NAT** (VM → Settings → Add → Network Adapter) para darle salida a Internet.
2. Verificar conectividad: `ping 8.8.8.8`.
3. Descargar e instalar Tailscale desde [tailscale.com/download/windows](https://tailscale.com/download/windows).
4. Iniciar sesión y unir el servidor a la *tailnet*. 
5. Desde otro dispositivo con Tailscale (celular en datos móviles), abrir la dirección IP asignada por la tailnet.

---

## 7. Instalación de CachyOS (cliente)

CachyOS es una distribución basada en Arch Linux, optimizada para rendimiento. Se instaló con el asistente **Calamares**:

| Paso | Elección |
|---|---|
| Idioma / zona | Español (México) / America/Guatemala |
| Teclado | Spanish (Latin American) |
| Bootloader | Limine (soporte nativo para snapshots Btrfs) |
| Particionado | Borrar disco · Btrfs · `/boot` FAT32 4 GB + raíz Btrfs 41 GB |
| Escritorio | KDE Plasma (interfaz familiar para usuarios de Windows) |
| Usuario | Usuario con privilegios de administrador |


---

## 8. Pruebas y resultados

| Prueba | Origen → Destino | Comando / método | Resultado |
|---|---|---|---|
| Conectividad | Host → Servidor | `ping 192.168.57.10` | 4/4 paquetes, 0 % pérdida |
| Conectividad | Servidor → Host | `ping 192.168.57.1` | 4/4 paquetes, 0 % pérdida |
| Conectividad | CachyOS → Host y Servidor | `ping -c4` | 0 % pérdida |
| Salida a Internet | Servidor → Internet | `ping 8.8.8.8` | 0 % pérdida |
| DHCP | CachyOS | `ip addr show` | Recibió `192.168.57.129` |
| DHCP | Servidor | Consola DHCP → Concesiones de direcciones | Concesión registrada para `cachyos-x8664` |
| IIS local | Servidor | `http://localhost` | Página personalizada |
| IIS en red | Host | Navegador → `http://192.168.57.10` | 
| IIS en red | CachyOS | `curl http://192.168.57.10` y navegador |
| IIS por VPN | Celular en red externa | Navegador → `http://IPAsignadaTailnet` |

---

## Problemas encontrados y soluciones

| Problema | Causa | Solución |
|---|---|---|
| El host no recibía respuesta al hacer ping al servidor | Windows Defender Firewall bloquea ICMP entrante por defecto | Crear regla de firewall ICMPv4 con `New-NetFirewallRule` |
| Riesgo de que el cliente recibiera IP del DHCP de VMware y no del servidor | Dos servidores DHCP activos en VMnet1 | Deshabilitar el DHCP local de VMware en Virtual Network Editor |
| La IP estática podía chocar con direcciones dinámicas | IP fija dentro del rango DHCP de VMware (`.128–.254`) | Asignar `192.168.57.10`, fuera del rango, y crear exclusiones en el ámbito |
| El servidor no tenía salida a Internet para instalar Tailscale | Host-only es una red aislada | Agregar un segundo adaptador en modo NAT |
| No se podía acceder al sitio desde fuera de la red local | Red privada virtual sin exposición externa | Publicar mediante VPN mesh (Tailscale), sin abrir puertos |

---

## Limitaciones y mejoras futuras

- **DNS:** el ámbito entrega `192.168.57.10` como servidor DNS y el servidor usa `127.0.0.1`, pero el rol **Servidor DNS no está instalado**. Siguiente paso: instalar DNS y crear la zona `lab.local`.
- **HTTPS:** el sitio se sirve por HTTP. Mejora: certificado autofirmado y binding en el puerto 443.
- **Active Directory:** unir el cliente a un dominio con AD DS.
- **Server Core:** repetir la configuración completa solo con PowerShell.
- **Automatización:** consolidar todos los comandos en un script `.ps1` reproducible.

---

## Lo que aprendí

- Diferencias entre las redes virtuales **Host-only, NAT y Bridged** y cuándo usar cada una.
- Planificar direccionamiento IP: separar IP estáticas del rango dinámico con exclusiones.
- El funcionamiento del proceso **DORA** y cómo verificarlo desde el cliente y el servidor.
- Diagnosticar conectividad por capas: adaptador (`Get-NetAdapter`), IP (`ipconfig`, `ip addr`), alcance (`ping`) y servicio (`curl`).
- Que el firewall es la primera sospecha cuando la red está bien configurada pero no hay respuesta.

---


## Autor

**Luis Tocay** – Estudiante de Ingeniería en Sistemas, Universidad Mariano Gálvez
[GitHub](https://github.com/[tu-usuario])

> Laboratorio realizado con fines académicos. Se usó la versión de evaluación gratuita de Windows Server 2022. No se incluyen imágenes ISO ni discos virtuales.
