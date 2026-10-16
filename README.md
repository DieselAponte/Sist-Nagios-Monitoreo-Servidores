# Enterprise Hybrid Monitoring & Alerting Infrastructure

Arquitectura de observabilidad on-premise implementada con **Nagios Core**, diseñada para la telemetría, auditoría de disponibilidad y notificación segura en entornos heterogéneos (**Linux Debian 13** y **Windows Server 2022**). 

El sistema integra un modelo de sondeo activo en dos fases (Capa 3 y Capa 7), desacoplando la conectividad de red de la salud de los servicios, e incorpora un pipeline redundante de alertas en tiempo real mediante **Telegram API** y **Postfix SMTP Relay** con mitigación anti-spoofing.

---

## 📦 Acceso a las Máquinas Virtuales del Laboratorio

Los appliances virtuales configurados (archivos `.ova` / `.vmx`) con el laboratorio completo preconfigurado, credenciales de prueba y servicios listos para levantar en VMware Workstation están disponibles en el siguiente enlace de almacenamiento:

[![Google Drive](https://img.shields.io/badge/Google%20Drive-Descargar%20Máquinas%20Virtuales-blue?logo=googledrive&style=for-the-badge)](https://drive.google.com/drive/folders/TU_ID_DE_CARPETA_AQUI?usp=sharing)

> **Nota:** Se recomienda importar las máquinas virtuales manteniendo la configuración de red en modo **Custom (VMnet con segmento 192.168.100.0/24)** para asegurar la interoperabilidad inmediata entre los tres nodos.

---

## 🗺️ Topología de Red y Asignación de Roles

| Dispositivo | Sistema Operativo | Dirección IP | Rol en la Arquitectura | Protocolo / Puerto de Telemetría |
| :--- | :--- | :--- | :--- | :--- |
| **M1** | Ubuntu Server 24.04 | `192.168.100.10/24` | Servidor Central de Monitoreo (Nagios Core + Postfix) | N/A (Origen del Polling) |
| **M2** | Windows Server 2022 | `192.168.100.40/24` | Nodo de Aplicaciones Windows | **TCP 12489** (`NSClient++ / check_nt`) |
| **M3** | Debian 13 | `192.168.100.30/24` | Nodo de Servicios Linux | **TCP 5666** (`NRPE Daemon`) |

---


## ⚙️ Arquitectura de Comunicación y Flujo de Datos

```text
[ NAGIOS CORE SERVER (M1: 192.168.100.10) ]
│
├─── 1. Chequeo L3 (ICMP Echo Request - Sin Puertos) ────► ¿Host Vivo?
│                                                              │ (Sí)
├─── 2. Chequeo L4/L7 (Sockets TCP) ───────────────────────────┤
│         │                                                    │
│         ├──► [TCP 5666]  ──► Debian 13 (NRPE Execution)      ▼
│         └──► [TCP 12489] ──► Windows Server (NSClient++ API)
│
[Detección de Falla: Estado HARD DOWN (2/2 intentos)]
│
├─► Canal Rápido: /usr/local/bin/notify_telegram.sh ──► HTTPS POST ──► [ Telegram Bot API ]
│
└─► Canal Formal: Postfix MTA (Local Spool)
│
▼
[ sender_canonical (Anti-Spoofing) ]
(nagios@local -> usuario@gmail.com)
│
▼
[ TLS 1.3 / SASL Auth - Puerto 587 ] ──► [ smtp.gmail.com ] ──► Bandeja Destinatario
```

### 1. Desacoplamiento de Capas (L3 vs L4/L7)
* **Disponibilidad de Red (Capa 3):** Se valida mediante `check-host-icmp-fast`. ICMP es un protocolo de capa de red que **no utiliza puertos**; sirve para determinar exclusivamente si la interfaz y el medio físico/virtual responden.
* **Telemetría de Recursos (Capa 4/7):** Solo si la Capa 3 responde, Nagios abre conexiones TCP hacia puertos específicos:
  * **TCP 12489 (`check_nt`):** Consulta directa a las APIs de contadores de rendimiento de Windows Server (`MEMUSE`, `CPULOAD`).
  * **TCP 5666 (`check_nrpe`):** Ejecución remota de scripts nativos en Debian, devolviendo el `stdout` y el código de salida numérico (`Exit Code`: 0=OK, 1=WARNING, 2=CRITICAL).

### 2. Pipeline de Notificación y Prevención Anti-Spoofing
* **Telegram Bot API:** Notificación inmediata mediante script en Bash optimizado con `curl`, enviando payloads con formato HTML para alertar a los administradores en sus dispositivos móviles.
* **Postfix Smart Host Relay:** Postfix actúa como cliente intermedio hacia `smtp.gmail.com:587`.
* **Mitigación de Rechazo SMTP:** Google exige que el usuario autenticado coincida con el remitente técnico del correo. Para solventar el rechazo de correos por políticas anti-spoofing (`Relay access denied`), se configuró `sender_canonical_maps` en Postfix, transformando el `Envelope From` local (`nagios@server-monitoreo.local`) en una identidad verificada ante los registros SPF/DKIM de Google.

---

## 🛠️ Archivos Clave del Repositorio

* `/usr/local/nagios/etc/objects/commands.cfg`: Definición de comandos de chequeo (`check_nt`, `check_nrpe`, `check-host-icmp-fast`) y directivas de notificación para Telegram y Postfix.
* `/usr/local/nagios/etc/objects/contacts.cfg`: Definición de grupos y asignación de correos receptores para el usuario `nagiosadmin`.
* `/usr/local/nagios/etc/objects/windows-server.cfg`: Configuración del host Windows Server 2022, aislamiento de hostgroup y umbrales de servicio.
* `/usr/local/bin/notify_telegram.sh`: Script en Bash que captura las macros de Nagios (`$NOTIFICATIONTYPE$`, `$HOSTNAME$`, `$HOSTSTATE$`, etc.) y las despacha vía HTTP POST.
* `/etc/postfix/main.cf`: Configuración del Relayhost, activación SASL y directivas TLS.
* `/etc/postfix/sender_canonical`: Tabla hash de mapeo de remitentes locales a direcciones externas autorizadas.

---

## 🔍 Comandos de Auditoría y Verificación en Vivo

### Monitoreo de Procesos y Logs (Servidor M1)
```bash
# Ver eventos de cambio de estado y disparo de alertas en Nagios
sudo tail -f /usr/local/nagios/var/nagios.log

# Monitorear transacciones SMTP y handshakes TLS en Postfix
sudo tail -f /var/log/mail.log

# Auditar la cola de salida de correos (debe mantenerse vacía)
sudo mailq

# Forzar vaciado de correos encolados
sudo postqueue -f

# Limpiar correos corruptos o rebotados en la cola
sudo postsuper -d ALL
