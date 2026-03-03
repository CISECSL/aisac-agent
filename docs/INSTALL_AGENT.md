# Guia de Instalacion - AISAC Agent (Endpoints)

Guia para instalar el agente AISAC en servidores y estaciones de trabajo Linux y Windows.

## Indice

1. [Que se instala](#1-que-se-instala)
2. [Requisitos previos](#2-requisitos-previos)
3. [Obtener credenciales](#3-obtener-credenciales)
4. [Instalacion en Linux](#4-instalacion-en-linux)
5. [Instalacion en Windows](#5-instalacion-en-windows)
6. [Verificacion post-instalacion](#6-verificacion-post-instalacion)
7. [Instalacion no interactiva](#7-instalacion-no-interactiva)
8. [Que se configura automaticamente](#8-que-se-configura-automaticamente)
9. [Desinstalacion](#9-desinstalacion)
10. [Resolucion de problemas](#10-resolucion-de-problemas)

---

## 1. Que se instala

El instalador configura automaticamente dos componentes en cada endpoint:

| Componente | Descripcion |
|------------|-------------|
| **Wazuh Agent** | Agente HIDS que reporta al Wazuh Manager centralizado |
| **AISAC Agent** | Heartbeat de estado + reenvio de logs locales a la plataforma AISAC |

### Arquitectura

```
  ┌──────────────────────────────────┐
  │         Endpoint (este equipo)   │
  │                                  │
  │  Wazuh Agent ──────────────────────> Wazuh Manager (1514/1515)
  │                                  │
  │  AISAC Agent ──────────────────────> Plataforma AISAC (HTTPS 443)
  │    - Heartbeat (estado)          │
  │    - Collector (logs locales)    │
  └──────────────────────────────────┘
```

---

## 2. Requisitos previos

### Sistemas operativos soportados

| SO | Versiones |
|----|-----------|
| **Ubuntu** | 20.04, 22.04, 24.04 LTS |
| **Debian** | 11, 12 |
| **CentOS / RHEL** | 7, 8, 9 |
| **Rocky Linux / AlmaLinux** | 8, 9 |
| **Windows Server** | 2016, 2019, 2022 |
| **Windows** | 10, 11 |

### Requisitos del sistema

| Requisito | Linux | Windows |
|-----------|-------|---------|
| Acceso | root / sudo | Administrador |
| Init system | systemd | Windows Services |
| Shell | bash | PowerShell 5.0+ |
| Descarga | curl | PowerShell (Invoke-WebRequest) |

### Arquitecturas soportadas

- **Linux**: amd64 (x86_64), arm64 (aarch64)
- **Windows**: amd64

### Conectividad de red

| Destino | Puerto | Uso |
|---------|--------|-----|
| Wazuh Manager | 1514/TCP | Comunicacion del agente Wazuh |
| Wazuh Manager | 1515/TCP | Registro del agente Wazuh |
| `api.aisac.cisec.es` | 443/TCP | Plataforma AISAC (heartbeat + logs) |

---

## 3. Obtener credenciales

Necesitas tres datos antes de instalar:

### 3.1 IP del Wazuh Manager

La IP (publica o privada) del servidor donde esta instalado el Wazuh Manager.

- Si el endpoint esta en la **misma red/VPC** que el Manager: usar la IP privada
- Si el endpoint esta en **otra red**: usar la IP publica del Manager

### 3.2 API Key del asset

1. Accede al Dashboard AISAC
2. Ve a **Assets** y crea el asset que representa a este endpoint
3. Copia la **API Key** (formato: `aisac_xxxx...`)


### 3.3 Auth Token (JWT)

Token JWT proporcionado por el administrador de la plataforma. Necesario para autenticarse contra el gateway de la API.

---

## 4. Instalacion en Linux

### Paso 1: Descargar el instalador

```bash
curl -sSL https://raw.githubusercontent.com/aisacAdmin/aisac-agent/main/scripts/install.sh -o install.sh
```

### Paso 2: Ejecutar el instalador

```bash
sudo bash install.sh -m <MANAGER_IP> -k <API_KEY> -t <AUTH_TOKEN>
```

**Ejemplo:**

```bash
sudo bash install.sh \
  -m 54.78.120.30 \
  -k aisac_abc123def456 \
  -t eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Parametros

| Parametro | Obligatorio | Descripcion |
|-----------|-------------|-------------|
| `-m <MANAGER_IP>` | Si | IP del Wazuh Manager |
| `-k <API_KEY>` | Si | API Key del asset en la plataforma AISAC |
| `-t <AUTH_TOKEN>` | No | Token JWT para autenticacion con el gateway |
| `-u <URL>` | No | URL del endpoint install-config (por defecto: produccion) |
| `-h` | No | Mostrar ayuda |

> Si no se pasa `-k`, el script lo solicita de forma interactiva.

### Que hace el instalador

```
  Step 1/2: Instalar Wazuh Agent
    - Llama al endpoint install-config para obtener la configuracion
    - Descarga e instala el paquete Wazuh Agent
    - Configura la conexion al Manager
    - Inicia el servicio wazuh-agent

  Step 2/2: Instalar AISAC Agent
    - Instala el binario aisac-agent
    - Genera la configuracion (heartbeat + collector)
    - Instala e inicia el servicio systemd
```

### Estructura de directorios

```
/opt/aisac/
  aisac-agent                    # Binario

/etc/aisac/
  agent.yaml                     # Configuracion

/var/lib/aisac/
  sincedb.json                   # Posicion de lectura de logs
  safety_state.json              # Estado interno

/var/log/aisac/
  agent.log                      # Logs del agente
```

---

## 5. Instalacion en Windows

### Paso 1: Descargar los scripts

Descargar los tres scripts de instalacion en una carpeta:

```powershell
# Crear carpeta temporal
New-Item -ItemType Directory -Path "$env:TEMP\aisac-install" -Force
cd "$env:TEMP\aisac-install"

# Descargar scripts
$baseUrl = "https://raw.githubusercontent.com/aisacAdmin/aisac-agent/main/scripts"
@("install.ps1", "install-wazuh-agent.ps1", "install-aisac-agent.ps1") | ForEach-Object {
    Invoke-WebRequest -Uri "$baseUrl/$_" -OutFile $_ -UseBasicParsing
}
```

### Paso 2: Ejecutar el instalador

Abrir PowerShell **como Administrador**:

```powershell
.\install.ps1 -ManagerIp <MANAGER_IP> -ApiKey <API_KEY> -AuthToken <AUTH_TOKEN>
```

**Ejemplo:**

```powershell
.\install.ps1 `
  -ManagerIp 54.78.120.30 `
  -ApiKey aisac_abc123def456 `
  -AuthToken eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Parametros

| Parametro | Obligatorio | Descripcion |
|-----------|-------------|-------------|
| `-ManagerIp` | Si | IP del Wazuh Manager |
| `-ApiKey` | No | API Key del asset (se solicita si no se pasa) |
| `-AuthToken` | No | Token JWT para autenticacion con el gateway |
| `-RegisterUrl` | No | URL del endpoint install-config |

### Estructura de directorios

```
C:\Program Files\AISAC\
  aisac-agent.exe                # Binario
  nssm.exe                      # Wrapper de servicios

C:\ProgramData\AISAC\
  agent.yaml                     # Configuracion
  data\                          # Datos persistentes
  logs\
    agent.log                    # Logs del agente
    service-stdout.log           # Salida del servicio
    service-stderr.log           # Errores del servicio
```

---

## 6. Verificacion post-instalacion

### 6.1 Comprobar servicios

**Linux:**

```bash
# Wazuh Agent
sudo systemctl status wazuh-agent

# AISAC Agent
sudo systemctl status aisac-agent
```

**Windows:**

```powershell
# Wazuh Agent
Get-Service WazuhSvc

# AISAC Agent
Get-Service AISACAgent
```

Ambos deben estar en estado **Running**.

### 6.2 Verificar conexion con el Manager

**Linux:**

```bash
# Ver estado del agente Wazuh
sudo /var/ossec/bin/agent_control -i 000

# Ver logs de conexion
sudo tail -20 /var/ossec/logs/ossec.log
```

**Windows:**

```powershell
# Ver logs de Wazuh
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 20
```

### 6.3 Verificar heartbeat AISAC

**Linux:**

```bash
tail -f /var/log/aisac/agent.log
```

**Windows:**

```powershell
Get-Content "C:\ProgramData\AISAC\logs\agent.log" -Wait
```

Buscar mensajes de heartbeat exitosos. En la plataforma AISAC, el asset debe aparecer como **Online**.

---

## 7. Instalacion no interactiva

Para despliegues automatizados con herramientas como Ansible, Puppet o scripts:

### Linux

```bash
# Usando variables de entorno
AISAC_API_KEY="aisac_abc123" \
AISAC_AUTH_TOKEN="eyJhbG..." \
sudo -E bash install.sh -m 54.78.120.30

# O con todos los parametros en linea
sudo bash install.sh \
  -m 54.78.120.30 \
  -k aisac_abc123 \
  -t eyJhbG...
```

### Windows

```powershell
# Usando variables de entorno
$env:AISAC_API_KEY = "aisac_abc123"
$env:AISAC_AUTH_TOKEN = "eyJhbG..."
.\install.ps1 -ManagerIp 54.78.120.30

# O con todos los parametros
.\install.ps1 -ManagerIp 54.78.120.30 -ApiKey aisac_abc123 -AuthToken eyJhbG...
```

---

## 8. Que se configura automaticamente

El instalador genera la configuracion en base a la respuesta del endpoint `install-config`:

| Componente | Configuracion |
|------------|---------------|
| **Wazuh Agent** | IP y puerto del Manager, nombre del agente, grupo del tenant |
| **Heartbeat** | URL, API Key, Asset ID, intervalo de 120s |
| **Collector** | Solo se habilita si detecta fuentes de logs locales (ej. Suricata) |
| **Acciones SOAR** | Configuradas pero deshabilitadas (servidor SOAR no conectado) |
| **Safety** | Auto-revert habilitado con TTL por accion |

### Fuentes de logs auto-detectadas

| Fuente | Ruta (Linux) | Ruta (Windows) |
|--------|-------------|----------------|
| Suricata EVE | `/var/log/suricata/eve.json` | `C:\Program Files\Suricata\log\eve.json` |

> Si no se detecta ninguna fuente, el collector queda deshabilitado. Se puede habilitar manualmente editando `agent.yaml`.

---

## 9. Desinstalacion

### Linux

```bash
# Parar y deshabilitar servicios
sudo systemctl stop aisac-agent
sudo systemctl disable aisac-agent
sudo systemctl stop wazuh-agent
sudo systemctl disable wazuh-agent

# Eliminar AISAC
sudo rm -f /etc/systemd/system/aisac-agent.service
sudo rm -rf /opt/aisac /etc/aisac /var/lib/aisac /var/log/aisac
sudo rm -f /usr/local/bin/aisac-agent

# Eliminar Wazuh Agent
sudo dpkg --purge wazuh-agent    # Debian/Ubuntu
# o
sudo rpm -e wazuh-agent          # CentOS/RHEL

sudo systemctl daemon-reload
```

### Windows

```powershell
# Parar servicios
Stop-Service AISACAgent -Force -ErrorAction SilentlyContinue
Stop-Service WazuhSvc -Force -ErrorAction SilentlyContinue

# Eliminar servicio AISAC (con NSSM)
& "C:\Program Files\AISAC\nssm.exe" remove AISACAgent confirm

# Eliminar archivos AISAC
Remove-Item "C:\Program Files\AISAC" -Recurse -Force
Remove-Item "C:\ProgramData\AISAC" -Recurse -Force

# Desinstalar Wazuh Agent
msiexec.exe /x wazuh-agent /q
```

---

## 10. Resolucion de problemas

### El instalador falla con HTTP 401

```
[ERROR] agent-register returned HTTP 401
```

**Causa**: La API Key o el Auth Token no son validos.

**Solucion**:
- Verificar la API Key en la plataforma (Assets > [Tu asset] > API Key)
- Verificar que el Auth Token (JWT) es correcto y no ha expirado
- Asegurar que se pasan ambos parametros: `-k` y `-t`

### Wazuh Agent no conecta al Manager

```bash
# Linux
sudo tail -30 /var/ossec/logs/ossec.log
```

**Causas posibles**:
- La IP del Manager (`-m`) es incorrecta
- Los puertos 1514/1515 no estan abiertos en el firewall del Manager
- El Manager no esta activo

**Solucion**:
1. Verificar que la IP del Manager es accesible: `telnet <MANAGER_IP> 1514`
2. Pedir al administrador que abra los puertos 1514/1515 en el firewall del Manager

### Heartbeat devuelve 401

```bash
grep -i "401\|unauthorized" /var/log/aisac/agent.log
```

**Causas posibles**:
- API Key incorrecta en la configuracion
- Auth Token ausente o invalido
- Version antigua del binario que no soporta auth_token

**Solucion**: Verificar `api_key` y `auth_token` en el fichero de configuracion:

```bash
# Linux
cat /etc/aisac/agent.yaml | grep -A2 "api_key\|auth_token"
```

```powershell
# Windows
Select-String -Path "C:\ProgramData\AISAC\agent.yaml" -Pattern "api_key|auth_token"
```

### El servicio AISAC no arranca

**Linux:**

```bash
# Ver logs detallados
sudo journalctl -u aisac-agent -n 50 --no-pager

# Ejecutar manualmente
sudo /opt/aisac/aisac-agent -c /etc/aisac/agent.yaml
```

**Windows:**

```powershell
# Ver logs
Get-Content "C:\ProgramData\AISAC\logs\service-stderr.log" -Tail 50

# Ejecutar manualmente
& "C:\Program Files\AISAC\aisac-agent.exe" -c "C:\ProgramData\AISAC\agent.yaml"
```

### El asset no aparece como Online en la plataforma

1. Verificar que el servicio `aisac-agent` esta activo
2. Buscar errores de heartbeat en los logs
3. Verificar conectividad HTTPS: `curl -v https://api.aisac.cisec.es`
4. Comprobar que el `asset_id` en la configuracion coincide con el de la plataforma
