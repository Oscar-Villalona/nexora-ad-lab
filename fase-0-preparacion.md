# Fase 0 — Planificación y preparación del host

> Laboratorio Active Directory que simula una empresa pequeña para **atacarla, defenderla y documentarla** (Suricata, Wireshark, IDS/IPS, SIEM y endpoints).

**Estado:** 🟡 En curso · **Siguiente:** [Fase 1 — Diseño de red y firewall](fase-1-red.md)

---

## 🎯 Objetivo

Dejar el host listo y el proyecto planificado antes de crear ninguna VM: recursos, Hyper-V, ISOs verificadas, convenciones y estructura de documentación.

**Reglas del laboratorio**
1. El tráfico de ataque nunca sale de los switches privados de Hyper-V.
2. Solo software libre o licencias de evaluación legales.
3. Cada cambio queda registrado en la bitácora con evidencia.

## 🖥️ Host

| Componente | Detalle |
|---|---|
| CPU | Intel i5-14600K (14 núcleos / 20 hilos) |
| RAM | 32 GB DDR5 |
| GPU | RTX 3070 8 GB |
| Disco | SSD M.2 512 GB |
| SO | Windows 11 Pro + Hyper-V |

El límite real es la **RAM y el disco**, así que se trabaja por perfiles (no se encienden todas las VMs a la vez).

## 🧱 VMs planificadas

| VM | Rol | SO | vCPU | RAM máx. | Segmento |
|---|---|---|---|---|---|
| FW01 | Firewall / router | OPNsense | 2 | 2 GB | Todos |
| DC01 | Controlador de dominio, DNS | Windows Server (eval.) | 2 | 4 GB | Servidores |
| SRV01 | Archivos / web | Windows Server (eval.) | 2 | 3 GB | Servidores |
| WS01 | Equipo de empleado | Windows 11 Ent. (eval.) | 2 | 4 GB | Corp |
| WS02 | Segundo empleado (opcional) | Windows 11 Ent. (eval.) | 2 | 4 GB | Corp |
| SENSOR01 | Suricata IDS/IPS | Ubuntu Server LTS | 2 | 4 GB | SOC |
| SIEM01 | SIEM (por decidir) | Ubuntu Server LTS | 4 | 8 GB | SOC |
| KALI01 | Atacante | Kali Linux | 2 | 4 GB | Atacante |

**Perfiles de uso:** Construcción AD (~13 GB) · Ataque + detección (~26 GB) · Completo sin WS02 (~29 GB, solo pruebas cortas).
**Disco estimado:** ~175–200 GB + checkpoints y capturas.

## 🌐 Segmentos de red (avance de la Fase 1)

| Segmento | Red | Switch Hyper-V | Gateway (FW01) |
|---|---|---|---|
| SOC | 10.10.99.0/24 | `vSW-SOC` | 10.10.99.1 |
| Servidores | 10.10.20.0/24 | `vSW-SRV` | 10.10.20.1 |
| Corp | 10.10.10.0/24 | `vSW-CORP` | 10.10.10.1 |
| Atacante | 10.10.66.0/24 | `vSW-ATK` | 10.10.66.1 |

La WAN de FW01 usa el *Default Switch* (NAT) de Hyper-V; no se crea switch externo.

## ⚙️ Preparación del host

```powershell
# 1. Comprobar Hyper-V
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All | Select-Object FeatureName, State

# 2. Carpetas del lab
New-Item -ItemType Directory -Force -Path C:\Lab\ISOs, C:\Lab\VMs, C:\Lab\VHDX, C:\Lab\Base, C:\Lab\Capturas

# 3. Rutas por defecto de Hyper-V
Set-VMHost -VirtualMachinePath "C:\Lab\VMs" -VirtualHardDiskPath "C:\Lab\VHDX"

# 4. Switches privados (uno por segmento)
"vSW-SOC","vSW-SRV","vSW-CORP","vSW-ATK" | ForEach-Object { New-VMSwitch -Name $_ -SwitchType Private }

# 5. (Opcional) Exclusión de Defender solo para los discos del lab
Add-MpPreference -ExclusionPath "C:\Lab\VHDX","C:\Lab\VMs"
```

## 💿 Software

| Software | Uso | Verificación |
|---|---|---|
| Windows Server (eval. ~180 días) | DC01, SRV01 | SHA256 |
| Windows 11 Enterprise (eval. ~90 días) | WS01, WS02 | SHA256 |
| OPNsense | FW01 | `.sha256` + firma |
| Ubuntu Server LTS | SENSOR01, SIEM01 | SHA256SUMS + GPG |
| Kali Linux | KALI01 | SHA256SUMS + GPG |
| Wireshark | Host | Firma del instalador |
| Sysmon | Endpoints (fase posterior) | Firma de Microsoft |

```powershell
Get-FileHash C:\Lab\ISOs\<archivo>.iso -Algorithm SHA256
```

## 📏 Convenciones

- **Equipos:** `ROLnn` (FW01, DC01…), máx. 15 caracteres; nombre de VM = hostname.
- **Usuarios:** `nombre.apellido`; administradores con prefijo `adm-`.
- **Dominio (propuesta):** `astralab.internal` — se evita `.local`.
- **Checkpoints:** `<VM>-F<fase>-<descripción>-<AAAAMMDD>`, máx. 3 por VM, checkpoints automáticos desactivados.
- **Credenciales:** baúl local (KeePassXC) solo para el lab; las contraseñas débiles a propósito se marcan como tales. Nunca se suben al repo.

## ⚠️ Riesgos

| Riesgo | Mitigación |
|---|---|
| SSD de 512 GB compartido con Windows | Medir espacio libre; si hay < ~250 GB, mover `C:\Lab` a otro SSD |
| RAM al límite | Trabajar por perfiles |
| El port mirroring de Hyper-V solo funciona dentro del mismo switch | NIC de captura por switch en SENSOR01, o Suricata en OPNsense (se decide en la Fase 1) |
| Caducan las evaluaciones de Windows | Instalar cada Windows solo cuando toque |

## 🔲 Decisiones pendientes

- [ ] SIEM: Wazuh, Splunk Free o Elastic
- [ ] Windows Server 2022 o 2025
- [ ] Nombre definitivo del dominio
- [ ] ¿Se incluye WS02?
- [ ] Suricata en SENSOR01, en OPNsense o en ambos
- [ ] Espacio libre real del SSD

## ✅ Checklist de salida

- [ ] Virtualización habilitada y Hyper-V activo
- [ ] Espacio en disco medido
- [ ] Carpetas y rutas de Hyper-V configuradas
- [ ] Cuatro switches privados creados
- [ ] ISOs descargadas y hashes verificados
- [ ] Baúl de contraseñas creado
- [ ] Repositorio con estructura y `.gitignore`
- [ ] Primera entrada de bitácora escrita

---

<sub>`.gitignore` recomendado: `*.iso` `*.vhdx` `*.avhdx` `*.pcap` `*.pcapng` `*.kdbx`</sub>
