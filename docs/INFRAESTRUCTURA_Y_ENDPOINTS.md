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

## Máquina 1 — Contabo (Wazuh)

**IP:** `217.216.48.103`  
**Usuario SSH:** `sepul`  
**Auth:** llave `~/.ssh/id_ed25519_sepul` + password habilitado

### Servicios

| Contenedor | Descripción | Puerto |
|---|---|---|
| `single-node-wazuh.indexer` | OpenSearch | `9200` |
| `single-node-wazuh.manager` | Wazuh Manager | `1514`, `1515`, `55000` |
| `single-node-wazuh.dashboard` | UI Wazuh | `4430` |

### Accesos

| Servicio | URL | Credenciales |
|---|---|---|
| Wazuh Dashboard | `https://217.216.48.103:4430` | admin / admin |
| Wazuh Indexer API | `https://217.216.48.103:9200` | admin / admin |

### Configuración SSH para tunnel

```bash
# /etc/ssh/sshd_config
PasswordAuthentication yes
```

### Notas Wazuh 5.0

- Índice de vulnerabilidades: `wazuh-states-vulnerabilities` (sin wildcard)
- Campo agent ID: `wazuh.agent.id` (en 4.x era `agent.id`)
- Severidad almacenada con mayúscula: `"Critical"`, `"High"`, `"Medium"`, `"Low"`

---

## Máquina 2 — DigitalOcean (VulnChecker + Wazuh Agent)

**IP:** `68.183.110.20`  
**Usuario SSH:** `root`  
**Dokploy:** `http://68.183.110.20:3000`

### Servicios

| Contenedor | Descripción | Puerto |
|---|---|---|
| `vulnchecker-backend` | Spring Boot API | `8080` (interno) |
| `bilan-vulncheck-qg6jib` | PostgreSQL | `5432` (interno) |
| `dokploy-traefik` | Reverse proxy | `80`, `443` |
| `wazuh-agent` | Agente Wazuh | — (systemd) |

### Variables de entorno del backend

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

### Wazuh Agent

```bash
# Ver estado
sudo systemctl status wazuh-agent

# Reiniciar
sudo systemctl restart wazuh-agent

# Logs
sudo tail -f /var/ossec/logs/ossec.log
```

**Agente registrado:**
- ID: `001`
- Nombre: `digitalocean-vulnchecker`
- OS: Ubuntu 24.04.3 LTS
- Versión: 5.0.0

---

## Flujo completo

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

## URL base y variables

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

# Vulnerabilidades sincronizadas en BD local → {"count":0}
curl "$BASE/count-local"
```

---

### Consultas a Wazuh (via SSH tunnel)

> Todas requieren `--max-time 30` porque el SSH tunnel tarda ~10s en establecerse.

#### Resumen por severidad
```bash
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/summary"
```
**Resultado:** Medium: 1346 | High: 596 | Low: 62 | Critical: 13 | Sin clasificar: 746

---

#### Todas las vulnerabilidades (paginadas)
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
# Valores aceptados: critical | high | medium | low
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
# Compatible con Wazuh 4.x (agent.id) y 5.0 (wazuh.agent.id)
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/agent/001?limit=50"
```
**Resultado:** 2.763 (todas pertenecen al agente 001)

---

#### Por CVE
```bash
curl --max-time 30 -H "$WAZUH_AUTH" \
  "$BASE/$SSH_HOST/$SSH_USER/$SSH_PASS/cve/CVE-2024-52005"
```
**Resultado:** 2 coincidencias para CVE-2024-52005

---

### Sincronización masiva (guarda en BD local)

#### Contar vulnerabilidades remotas
```bash
curl -X POST --max-time 30 \
  -H "Content-Type: application/json" \
  -H "$WAZUH_AUTH" \
  -d '{"ip":"217.216.48.103","infrastructureCredentialId":1}' \
  "$BASE/remote-count"
```

#### Iniciar sincronización
```bash
curl -X POST --max-time 30 \
  -H "Content-Type: application/json" \
  -H "$WAZUH_AUTH" \
  -d '{"ip":"217.216.48.103","infrastructureCredentialId":1}' \
  "$BASE/consume"
```
> La sincronización corre en background. Verificar progreso con `count-local`.

---

## Compatibilidad Wazuh

| Query | Wazuh 4.x | Wazuh 5.0 |
|---|---|---|
| Severidad (capitalize) | ✅ | ✅ |
| Agent por `agent.id` | ✅ | — |
| Agent por `wazuh.agent.id` | — | ✅ |
| Agent (bool/should ambos) | ✅ | ✅ |
| Índice `wazuh-states-vulnerabilities` | ✅ | ✅ |
