# VulnChecker — Infraestructura y Endpoints

## Estado actual

| Componente | Estado | Detalle |
|---|---|---|
| Wazuh Manager | ✅ Running | Contabo 217.216.48.103 |
| Wazuh Indexer | ✅ Running | 2.763 vulnerabilidades indexadas |
| Wazuh Agent | ✅ Active | digitalocean-vulnchecker (ID: 001) |
| Backend API | ✅ UP | DigitalOcean 68.183.110.20 |
| PostgreSQL | ✅ Running | DigitalOcean (interno) |
| SSH Tunnel | ✅ Funcional | DigitalOcean → Contabo :9200 |

---

## Arquitectura

```
Wazuh Agent (68.183.110.20)
  └── reporta eventos → Wazuh Manager (217.216.48.103:1514)
                           └── indexa → OpenSearch :9200

VulnChecker Backend (68.183.110.20:8080)
  └── SSH tunnel → sepul@217.216.48.103:22
                     └── → OpenSearch localhost:9200
                               └── 2.763 vulnerabilidades
```

---

## Guía de configuración — Máquina 1 (Contabo / Wazuh)

**IP:** `217.216.48.103` | **Usuario:** `sepul`

### 1. Requisitos previos

- VPS con Ubuntu 22.04+
- Docker y Docker Compose instalados
- Puerto 22, 4430, 1514, 1515, 55000 abiertos

### 2. Clonar el repositorio oficial de Wazuh Docker

```bash
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker/single-node
```

### 3. Generar certificados SSL

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

### 4. Levantar el stack de Wazuh

```bash
docker compose up -d
```

Esto levanta tres contenedores:
- `single-node-wazuh.indexer` — OpenSearch en puerto `9200`
- `single-node-wazuh.manager` — Wazuh Manager en puertos `1514`, `1515`, `55000`
- `single-node-wazuh.dashboard` — Dashboard en puerto `4430`

### 5. Verificar que los servicios están activos

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

### 6. Habilitar autenticación SSH por password

El backend necesita conectarse via password para abrir el SSH tunnel:

```bash
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

### 7. Crear password para el usuario SSH

```bash
sudo passwd sepul
```

### 8. Verificar el Wazuh Indexer

```bash
curl -sk -u admin:admin https://localhost:9200
# Debe responder: { "name": "wazuh.indexer", "cluster_name": "wazuh-cluster" }
```

### Servicios resultantes

| Contenedor | Puerto expuesto |
|---|---|
| `single-node-wazuh.indexer` | `0.0.0.0:9200` |
| `single-node-wazuh.manager` | `0.0.0.0:1514-1515`, `0.0.0.0:55000` |
| `single-node-wazuh.dashboard` | `0.0.0.0:4430` |

### Accesos

| Servicio | URL | Credenciales |
|---|---|---|
| Wazuh Dashboard | `https://217.216.48.103:4430` | admin / admin |
| Wazuh Indexer API | `https://217.216.48.103:9200` | admin / admin |

---

## Guía de configuración — Máquina 2 (DigitalOcean / VulnChecker)

**IP:** `68.183.110.20` | **Usuario:** `root`

### 1. Crear el Droplet en DigitalOcean

- Plan: Basic / Regular — **2GB RAM / 1 vCPU / 50GB SSD** (mínimo recomendado)
- OS: Ubuntu 24.04 LTS
- Auth: Password
- Región: la más cercana al VPS de Wazuh

### 2. Instalar Dokploy

```bash
curl -sSL https://dokploy.com/install.sh | sh
```

Dokploy queda accesible en `http://68.183.110.20:3000`. Registrarse en el primer acceso.

### 3. Crear la base de datos en Dokploy

En Dokploy → **New Service → Database → PostgreSQL**:

| Campo | Valor |
|---|---|
| Database Name | `vulncheck` |
| User | `admin` |
| Password | `admin123` |

Dokploy asigna un **Internal Host** (ej. `bilan-vulncheck-qg6jib`) para uso interno entre contenedores.

### 4. Crear el Application del backend en Dokploy

En Dokploy → **New Service → Application**:

| Campo | Valor |
|---|---|
| Provider | GitHub |
| Repository | `andressep95/DevSecOps---Lab-VulnChecker` |
| Branch | `main` |
| Build Path | `/vulncheckerbackend` |
| Docker File | `./Dockerfile` |
| Docker Context Path | `./vulncheckerbackend` |
| Port | `8080` |

### 5. Configurar variables de entorno del backend

En la pestaña **Environment** del Application:

```env
DB_USERNAME=admin
DB_PASSWORD=admin123
DB_O_LOCALHOST=bilan-vulncheck-qg6jib
SPRING_DATASOURCE_URL=jdbc:postgresql://bilan-vulncheck-qg6jib:5432/vulncheck
SPRING_DATASOURCE_USERNAME=admin
SPRING_DATASOURCE_PASSWORD=admin123
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_FLYWAY_ENABLED=true
```

> Reemplazar `bilan-vulncheck-qg6jib` con el Internal Host real que asignó Dokploy a tu PostgreSQL.

### 6. Configurar dominio en Dokploy

En la pestaña **Domains** del Application:

| Campo | Valor |
|---|---|
| Host | `68.183.110.20.nip.io` |
| Path | `/` |
| Port | `8080` |
| HTTPS | No |

> `nip.io` es un servicio DNS gratuito que resuelve `<ip>.nip.io` a esa misma IP. No requiere dominio propio.

### 7. Hacer Deploy

Click en **Deploy**. El build tarda ~3-5 minutos (Maven descarga dependencias).

### 8. Verificar que el backend está activo

```bash
curl http://68.183.110.20.nip.io/actuator/health
# {"status":"UP"}
```

---

## Guía de configuración — Wazuh Agent (en Máquina 2)

El agente se instala en el mismo Droplet de DigitalOcean para monitorear ese servidor y reportar al Manager en Contabo.

### 1. Obtener el comando de instalación desde el Dashboard

En `https://217.216.48.103:4430` → **Deploy new agent**:

- Package: **DEB amd64**
- Server address: `217.216.48.103`
- Agent name: `digitalocean-vulnchecker`

### 2. Instalar el agente (en el Droplet de DigitalOcean)

```bash
# Descargar el paquete (Wazuh 5.0 beta)
wget https://packages-staging.xdrsiem.wazuh.info/pre-release/5.x/apt/pool/main/w/wazuh-agent/wazuh-agent_5.0.0-beta1_amd64.deb

# Instalar configurando el Manager
sudo WAZUH_MANAGER='217.216.48.103' \
     WAZUH_AGENT_GROUP='default' \
     WAZUH_AGENT_NAME='digitalocean-vulnchecker' \
     dpkg -i ./wazuh-agent_5.0.0-beta1_amd64.deb
```

### 3. Habilitar e iniciar el agente

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### 4. Verificar que está activo

```bash
sudo systemctl status wazuh-agent
# Active: active (running)
```

### 5. Confirmar registro en el Dashboard de Wazuh

En `https://217.216.48.103:4430` → **Endpoints**: debe aparecer el agente con status **Active**.

### Comandos útiles del agente

```bash
# Estado
sudo systemctl status wazuh-agent

# Reiniciar
sudo systemctl restart wazuh-agent

# Logs en tiempo real
sudo tail -f /var/ossec/logs/ossec.log

# Configuración del agente
sudo cat /var/ossec/etc/ossec.conf
```

### Agente registrado

| Campo | Valor |
|---|---|
| ID | `001` |
| Nombre | `digitalocean-vulnchecker` |
| OS | Ubuntu 24.04.3 LTS |
| Versión | 5.0.0 |
| Grupo | default |
| Vulnerabilidades detectadas | 2.763 |

---

## URL base y variables para pruebas

```bash
BASE="http://68.183.110.20.nip.io/api/vulns"
SSH_HOST="217.216.48.103"
SSH_USER="sepul"
SSH_PASS="Sepul1995"
WAZUH_AUTH="Authorization: Basic $(echo -n 'admin:admin' | base64)"
```

---

## Endpoints

### Sistema

```bash
# Health check → {"status":"UP"}
curl http://68.183.110.20.nip.io/actuator/health

# Vulnerabilidades sincronizadas en BD local
curl "$BASE/count-local"
```

---

### Consultas a Wazuh (via SSH tunnel)

> Usar `--max-time 30` — el SSH tunnel tarda ~10s en establecerse.

#### Resumen por severidad
```bash
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/summary"
```
**Resultado:** Medium: 1.346 | High: 596 | Low: 62 | Critical: 13

---

#### Todas las vulnerabilidades
```bash
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/all"
```
**Resultado:** 2.763 total

---

#### Top N vulnerabilidades
```bash
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/top/50"
```

---

#### Solo críticas
```bash
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/critical"
```
**Resultado:** 13 vulnerabilidades críticas

---

#### Por severidad
```bash
# Valores: critical | high | medium | low
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/severity/high?limit=100"
```

| Severidad | Total |
|---|---|
| Critical | 13 |
| High | 596 |
| Medium | 1.346 |
| Low | 62 |

---

#### Por agente
```bash
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/agent/001?limit=50"
```
**Resultado:** 2.763 (todas del agente 001)

---

#### Por CVE
```bash
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/cve/CVE-2024-52005"
```
**Resultado:** 2 coincidencias

---

### Sincronización masiva

```bash
# Contar vulnerabilidades remotas
curl -X POST --max-time 30 \
  -H "Content-Type: application/json" \
  -H "$WAZUH_AUTH" \
  -d '{"ip":"217.216.48.103","infrastructureCredentialId":1}' \
  "$BASE/remote-count"

# Iniciar sincronización (corre en background)
curl -X POST --max-time 30 \
  -H "Content-Type: application/json" \
  -H "$WAZUH_AUTH" \
  -d '{"ip":"217.216.48.103","infrastructureCredentialId":1}' \
  "$BASE/consume"
```

---

## Compatibilidad Wazuh

| Query | Wazuh 4.x | Wazuh 5.0 |
|---|---|---|
| Severidad (capitalize) | ✅ | ✅ |
| Agent por `agent.id` | ✅ | — |
| Agent por `wazuh.agent.id` | — | ✅ |
| Agent (bool/should ambos) | ✅ | ✅ |
| Índice `wazuh-states-vulnerabilities` | ✅ | ✅ |

### Notas Wazuh 5.0

- Índice: `wazuh-states-vulnerabilities` (sin wildcard, sin sufijo de agente)
- Campo agent ID: `wazuh.agent.id` (en 4.x era `agent.id`)
- Severidad como keyword con mayúscula: `"Critical"`, `"High"`, `"Medium"`, `"Low"`
