# VulnChecker — Infraestructura y Endpoints

## Estado actual

| Componente | Estado | Detalle |
|---|---|---|
| Wazuh Manager | ✅ Running | Contabo 217.216.48.103 |
| Wazuh Indexer | ✅ Running | 2.763 vulnerabilidades indexadas |
| Wazuh Agent | ✅ Active | digitalocean-vulnchecker (ID: 001) |
| Backend API | ✅ UP | DigitalOcean 68.183.110.20 (HTTPS) |
| PostgreSQL | ✅ Running | DigitalOcean (interno, Dokploy) |
| SSH Tunnel | ✅ Funcional | DigitalOcean → Contabo :9200 |
| Frontend | ✅ Running | Vercel (vulnchecker-frontend.vercel.app) |

---

## URLs de servicios

| Servicio | URL | Notas |
|---|---|---|
| Frontend (Vercel) | `https://vulnchecker-frontend.vercel.app` | SPA React |
| Backend API | `https://68.183.110.20.nip.io/api` | Spring Boot |
| Backend Health | `https://68.183.110.20.nip.io/actuator/health` | `{"status":"UP"}` |
| Dokploy UI | `http://68.183.110.20:3000` | Panel de despliegue |
| Wazuh Dashboard | `https://217.216.48.103:4430` | admin / admin |
| Wazuh Indexer API | `https://217.216.48.103:9200` | admin / admin |
| PostgreSQL (externo) | `postgresql://admin:admin123@68.183.110.20:5432/vulncheck` | Cliente SQL externo |
| PostgreSQL (interno) | `postgresql://admin:admin123@bilan-vulncheck-qg6jib:5432/vulncheck` | Entre contenedores Dokploy |

---

## Credenciales de acceso — resumen rápido

### Máquina 1 — Contabo (Wazuh)

| Acceso | Usuario | Contraseña |
|---|---|---|
| SSH | `sepul` | `Sepul1995` |
| Wazuh Dashboard | `admin` | `admin` |
| Wazuh Indexer API | `admin` | `admin` |

### Máquina 2 — DigitalOcean (Backend)

| Acceso | Usuario | Contraseña / Valor |
|---|---|---|
| SSH / root | `root` | (password del droplet) |
| Dokploy UI | — | `http://68.183.110.20:3000` (primer acceso crea cuenta) |
| PostgreSQL | `admin` | `admin123` |
| Base de datos | `vulncheck` | — |

**PostgreSQL — Conexión interna (entre contenedores Dokploy):**
```
postgresql://admin:admin123@bilan-vulncheck-qg6jib:5432/vulncheck
```

**PostgreSQL — Conexión externa (desde internet / cliente SQL):**
```
postgresql://admin:admin123@68.183.110.20:5432/vulncheck
```

### Vercel (Frontend)

| Acceso | Valor |
|---|---|
| URL producción | `https://vulnchecker-frontend.vercel.app` |
| Repositorio | `andressep95/DevSecOps---Lab-VulnChecker` |
| Root Directory | `frontend` |
| Variable de entorno | `VITE_API_URL=https://68.183.110.20.nip.io` |

### Aplicación VulnChecker (credenciales de prueba)

| Campo | Valor |
|---|---|
| Usuario (email prefix) | `admin.seguridad` |
| Dominio (auto-añadido) | `@usach.cl` |
| Email completo | `admin.seguridad@usach.cl` |
| Contraseña | `admin123` |

### Perfil de credencial de infraestructura (para el Consumer)

Crear en **Settings → Credenciales** dentro de la app:

| Campo | Valor |
|---|---|
| Nombre | `Wazuh-Contabo` (o cualquier nombre) |
| SSH User | `sepul` |
| SSH Password | `Sepul1995` |
| Wazuh User | `admin` |
| Wazuh Password | `admin` |

---

## Arquitectura

```
[Vercel] vulnchecker-frontend.vercel.app
  └── HTTPS → [DigitalOcean] https://68.183.110.20.nip.io/api
                  └── Spring Boot :8080
                        └── SSH tunnel (JSch) → sepul@217.216.48.103:22
                                                   └── OpenSearch :9200
                                                         └── 2.763 vulnerabilidades

[Wazuh Agent] (68.183.110.20)
  └── reporta eventos → [Wazuh Manager] 217.216.48.103:1514
                             └── indexa → OpenSearch :9200
```

---

## Guía de configuración — Máquina 1 (Contabo / Wazuh)

**IP:** `217.216.48.103` | **Usuario SSH:** `sepul` | **Password SSH:** `Sepul1995`

### 1. Requisitos previos

- VPS con Ubuntu 22.04+
- Docker y Docker Compose instalados
- Puertos abiertos: 22, 4430, 1514, 1515, 55000, 9200

### 2. Clonar el repositorio oficial de Wazuh Docker

```bash
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker/single-node
```

### 3. Generar certificados SSL

Wazuh requiere certificados TLS para la comunicación interna entre Manager, Indexer y Dashboard.

**3.1 — Descargar el script**

```bash
cd ~/wazuh-docker/single-node
curl -sO https://packages.wazuh.com/5.0/wazuh-certs-tool.sh
chmod +x wazuh-certs-tool.sh
```

El archivo `config.yml` ya viene en el repositorio y define los nodos.

**3.2 — Generar todos los certificados**

```bash
bash wazuh-certs-tool.sh -A
```

Esto crea el directorio `wazuh-certificates/` con:

```
wazuh-certificates/
├── root-ca.pem
├── root-ca-key.pem
├── admin.pem
├── admin-key.pem
├── wazuh.indexer.pem
├── wazuh.indexer-key.pem
├── wazuh.manager.pem
├── wazuh.manager-key.pem
├── wazuh.dashboard.pem
└── wazuh.dashboard-key.pem
```

**3.3 — Copiar certificados a cada componente**

```bash
mkdir -p config/wazuh_indexer/certs config/wazuh_manager/certs config/wazuh_dashboard/certs

# Indexer
cp wazuh-certificates/root-ca.pem config/wazuh_indexer/certs/
cp wazuh-certificates/wazuh.indexer.pem config/wazuh_indexer/certs/
cp wazuh-certificates/wazuh.indexer-key.pem config/wazuh_indexer/certs/
cp wazuh-certificates/admin.pem config/wazuh_indexer/certs/
cp wazuh-certificates/admin-key.pem config/wazuh_indexer/certs/

# Manager
cp wazuh-certificates/root-ca.pem config/wazuh_manager/certs/
cp wazuh-certificates/wazuh.manager.pem config/wazuh_manager/certs/
cp wazuh-certificates/wazuh.manager-key.pem config/wazuh_manager/certs/

# Dashboard
cp wazuh-certificates/root-ca.pem config/wazuh_dashboard/certs/
cp wazuh-certificates/wazuh.dashboard.pem config/wazuh_dashboard/certs/
cp wazuh-certificates/wazuh.dashboard-key.pem config/wazuh_dashboard/certs/
```

**3.4 — Asegurar permisos**

```bash
chmod 400 config/wazuh_indexer/certs/* \
           config/wazuh_manager/certs/* \
           config/wazuh_dashboard/certs/*
```

> Sin completar estos pasos el stack no arranca.

### 4. Levantar el stack de Wazuh

```bash
docker compose up -d
```

Contenedores resultantes:
- `single-node-wazuh.indexer` — OpenSearch en puerto `9200`
- `single-node-wazuh.manager` — Wazuh Manager en puertos `1514`, `1515`, `55000`
- `single-node-wazuh.dashboard` — Dashboard en puerto `4430`

### 5. Verificar servicios

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

### 6. Habilitar autenticación SSH por password

El backend necesita conectarse via password para abrir el SSH tunnel:

```bash
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

> En Ubuntu el servicio se llama `ssh`, no `sshd`.

### 7. Crear password para el usuario SSH

```bash
sudo passwd sepul
# Ingresar: Sepul1995
```

### 8. Verificar el Wazuh Indexer

```bash
curl -sk -u admin:admin https://localhost:9200
# Respuesta esperada: { "name": "wazuh.indexer", "cluster_name": "wazuh-cluster" }
```

### Accesos

| Servicio | URL | Credenciales |
|---|---|---|
| Wazuh Dashboard | `https://217.216.48.103:4430` | admin / admin |
| Wazuh Indexer API | `https://217.216.48.103:9200` | admin / admin |

---

## Guía de configuración — Máquina 2 (DigitalOcean / Backend)

**IP:** `68.183.110.20` | **Usuario:** `root`

### 1. Crear el Droplet en DigitalOcean

- Plan: Basic / Regular — **2GB RAM / 1 vCPU / 50GB SSD** (mínimo recomendado para Dokploy)
- OS: Ubuntu 24.04 LTS
- Auth: Password
- Región: la más cercana al VPS de Wazuh

### 2. Instalar Dokploy

```bash
curl -sSL https://dokploy.com/install.sh | bash
```

> Usar `bash`, no `sh` — el script usa sintaxis bash que `sh` no entiende.

Dokploy queda accesible en `http://68.183.110.20:3000`. Registrarse en el primer acceso.

### 3. Crear la base de datos en Dokploy

En Dokploy → **New Service → Database → PostgreSQL**:

| Campo | Valor |
|---|---|
| Database Name | `vulncheck` |
| User | `admin` |
| Password | `admin123` |

Dokploy asigna un **Internal Host** (ej. `bilan-vulncheck-qg6jib`) para comunicación entre contenedores.

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
cors.allowed-origins=https://vulnchecker-frontend.vercel.app
```

> Reemplazar `bilan-vulncheck-qg6jib` con el Internal Host real asignado por Dokploy.
> La variable `cors.allowed-origins` permite que el frontend en Vercel pueda llamar al backend.

### 6. Configurar dominio en Dokploy

En la pestaña **Domains** del Application:

| Campo | Valor |
|---|---|
| Host | `68.183.110.20.nip.io` |
| Path | `/` |
| Port | `8080` |
| HTTPS | **Yes** (Let's Encrypt) |

> `nip.io` resuelve `<ip>.nip.io` a esa misma IP. Traefik gestiona el certificado TLS automáticamente.

### 7. Hacer Deploy

Click en **Deploy**. El build tarda ~3-5 minutos (Maven descarga dependencias).

### 8. Verificar que el backend está activo

```bash
curl https://68.183.110.20.nip.io/actuator/health
# {"status":"UP"}
```

---

## Guía de configuración — Wazuh Agent (en Máquina 2)

El agente se instala en el Droplet de DigitalOcean para monitorear ese servidor y reportar al Manager en Contabo.

### 1. Obtener el comando de instalación desde el Dashboard

En `https://217.216.48.103:4430` → **Deploy new agent**:

- Package: **DEB amd64**
- Server address: `217.216.48.103`
- Agent name: `digitalocean-vulnchecker`

### 2. Instalar el agente

```bash
wget https://packages-staging.xdrsiem.wazuh.info/pre-release/5.x/apt/pool/main/w/wazuh-agent/wazuh-agent_5.0.0-beta1_amd64.deb

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

### 4. Verificar

```bash
sudo systemctl status wazuh-agent
# Active: active (running)
```

### 5. Confirmar registro en el Dashboard

En `https://217.216.48.103:4430` → **Endpoints**: debe aparecer con status **Active**.

### Comandos útiles del agente

```bash
sudo systemctl status wazuh-agent
sudo systemctl restart wazuh-agent
sudo tail -f /var/ossec/logs/ossec.log
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

## Guía de configuración — Máquina 3 (Vercel / Frontend)

El frontend es una SPA React desplegada en Vercel directamente desde el repositorio GitHub.

### 1. Crear proyecto en Vercel

1. Ir a [vercel.com](https://vercel.com) → **New Project**
2. Importar repositorio: `andressep95/DevSecOps---Lab-VulnChecker`
3. Configurar build:

| Campo | Valor |
|---|---|
| Root Directory | `frontend` |
| Framework Preset | Vite (detectado automático) |
| Build Command | `npm run build` |
| Output Directory | `dist` |

### 2. Variables de entorno en Vercel

En **Settings → Environment Variables**:

| Variable | Valor |
|---|---|
| `VITE_API_URL` | `https://68.183.110.20.nip.io` |

> Esta variable se inyecta en el bundle en tiempo de build. Apunta al backend con HTTPS para evitar Mixed Content.

### 3. Hacer Deploy

Click en **Deploy**. Vercel construye con `npm run build` y publica automáticamente.

### 4. Redeploy automático

Cada `git push` a `main` dispara un redeploy automático en Vercel.

### Acceso

| Campo | Valor |
|---|---|
| URL | `https://vulnchecker-frontend.vercel.app` |
| Framework | React + Vite |
| Build env | `.env.production` (VITE_API_URL definido en Vercel dashboard) |

---

## Cómo sincronizar vulnerabilidades (primer uso)

La BD de producción arranca vacía. Para poblarla:

### Paso 1 — Crear perfil de credenciales en la app

1. Ir a `https://vulnchecker-frontend.vercel.app`
2. Login con `admin.seguridad` / `admin123`
3. Ir a **Settings → Credenciales**
4. Crear nuevo perfil:

| Campo | Valor |
|---|---|
| Nombre | `Wazuh-Contabo` |
| SSH User | `sepul` |
| SSH Password | `Sepul1995` |
| Wazuh User | `admin` |
| Wazuh Password | `admin` |

### Paso 2 — Ejecutar sincronización

1. Ir a **Configuración Wazuh** (ícono de base de datos en el menú)
2. Completar el formulario:

| Campo | Valor |
|---|---|
| Dirección IP | `217.216.48.103` |
| Perfil de Credencial | `Wazuh-Contabo` |

3. Click en **"Iniciar Consumo de Datos"**

El proceso corre en background (~5-10 minutos para 2.763 registros). La pantalla muestra `Sincronizando: X / 2763` actualizándose cada 2 segundos.

---

## Variables y endpoints para pruebas con curl

```bash
BASE="https://68.183.110.20.nip.io/api/vulns"
SSH_HOST="217.216.48.103"
SSH_USER="sepul"
SSH_PASS="Sepul1995"
WAZUH_AUTH="Authorization: Basic $(echo -n 'admin:admin' | base64)"
```

### Sistema

```bash
# Health check
curl https://68.183.110.20.nip.io/actuator/health
# {"status":"UP"}

# Vulnerabilidades sincronizadas en BD local
curl "$BASE/count-local"
```

### Consultas a Wazuh (via SSH tunnel)

> Usar `--max-time 30` — el SSH tunnel tarda ~10s en establecerse.

```bash
# Resumen por severidad
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/summary"
# Medium: 1.346 | High: 596 | Low: 62 | Critical: 13

# Todas las vulnerabilidades
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/all"
# 2.763 total

# Top 50
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/top/50"

# Solo críticas
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/critical"
# 13 vulnerabilidades críticas

# Por severidad (critical | high | medium | low)
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/severity/high?limit=100"

# Por agente
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/agent/001?limit=50"

# Por CVE
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/cve/CVE-2024-52005"
# 2 coincidencias
```

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
