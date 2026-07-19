# Zabbix Monitoring Lab — Platform Deep-Dive
**Dinesh Ravichandiran**

Production work at PTC covers alert tuning, triage, and the day-to-day operational side of Zabbix across 50+ Fortune 500 environments. This self-hosted lab (Docker) exists to go deeper into the platform-engineering side I don't touch daily: discovery rules, preprocessing, escalation logic, RBAC, the API, and proxy architecture. Terraform-based provisioning is the next iteration I'm building toward, extending the same repeatable, version-controlled approach to monitoring-as-code.

Each section below documents a specific capability: what I set up, the exact steps, and what it taught me.

---

## Run it yourself

### GitHub Codespaces (no local Docker required)
This repo ships a `.devcontainer` config with Docker-in-Docker preconfigured. **Code → Codespaces → Create codespace on main** will automatically clone `zabbix/zabbix-docker` and run `make up DB=pgsql` to bring up the server, web UI, and database. Once ready, open forwarded port **80** to reach the Zabbix frontend (`Admin` / `zabbix`, the project's own documented demo login).

> `zabbix-docker` moved to a `make`-based workflow — the old single `docker-compose_*.yaml` files were replaced by `compose.yaml` + `compose_pgsql.yaml` combined via a Makefile. If upstream changes again, run `make help` inside `zabbix-docker` to see current targets.

### Local setup (~20 min)
```bash
git clone https://github.com/zabbix/zabbix-docker.git
cd zabbix-docker
make up DB=pgsql
# Frontend: http://localhost:80   Login: Admin / zabbix
docker ps                         # confirm server, db, web are up
docker compose logs -f zabbix-server | head -50
```

Add a second agent to have something to monitor:
```bash
docker run -d --name agent2 --network zabbix-docker_backend \
  -e ZBX_SERVER_HOST="zabbix-server" -e ZBX_HOSTNAME="lab-host-02" \
  zabbix/zabbix-agent2:alpine-latest
```

---

## Data collection & discovery

### Host + template linking
Linked a host (`lab-host-02`) to the **Linux by Zabbix agent** template rather than building items by hand — config should be inherited, not hand-crafted, so it stays consistent across an estate. Confirmed items were actually supported and fresh in Latest Data; a host that exists but isn't collecting is a blind spot.

### Breaking an unsupported item, on purpose
Changed an item key to `system.cpu.utilXXX` and watched it go **unsupported**, then reproduced the failure outside Zabbix to isolate the cause:
```bash
docker exec -it zabbix-server zabbix_get -s agent2 -k system.cpu.utilXXX   # fails
docker exec -it zabbix-server zabbix_get -s agent2 -k system.cpu.util      # works
```
**Takeaway:** read the unsupported reason on the item first — it usually states exactly what's wrong — then reproduce it with `zabbix_get` to isolate whether the fault is the agent, the key, or permissions. An unsupported item should never be disabled to make it quiet; unknown isn't healthy, it's a blind spot with a green icon.

### LLD filters on filesystem discovery
Out of the box, "Mounted filesystem discovery" picks up `/proc`, `/sys`, overlay mounts, and other container noise. Added filters:
```
{#FSTYPE}  matches       ^(ext4|xfs|btrfs)$
{#FSNAME}  does not match ^(/dev|/sys|/run|/proc|/var/lib/docker|/snap)
```
**Result:** cut discovered filesystem items by more than half. The fix is filtering on `{#FSTYPE}`/`{#FSNAME}` plus the lifetime setting for lost resources — not disabling discovery, and not stretching the interval to hide the churn.

### Dependent items + JSONPath preprocessing
Built a master **HTTP agent** item polling a JSON endpoint (`https://api.github.com/repos/zabbix/zabbix`), then created dependent items extracting fields with JSONPath (`$.stargazers_count`, `$.open_issues_count`) — one poll feeding several metrics instead of one poll per metric. At scale, the things to watch when preprocessing queues grow: preprocessing manager load, master item interval, payload size, JSONPath efficiency, and dependent item count.

### Discard unchanged with heartbeat
On a slow-changing item, added the **discard unchanged with heartbeat** (1h) preprocessing step — it only writes when the value changes, but the heartbeat guarantees a value at least hourly so `nodata()` triggers don't false-fire. One of the cheapest ways to cut database write volume without losing signal.

---

## Triggers & escalation

### Threshold + nodata triggers
Created a threshold trigger (`last(/lab-host-02/system.cpu.util)>20`) and a freshness trigger (`nodata(/lab-host-02/agent.ping,5m)=1`), then stopped the agent container to confirm the nodata trigger actually fires. `nodata()` is the one that's easy to forget — if the agent dies, a CPU threshold trigger goes quiet and everything looks fine. Silence and health look identical without it.

### Hysteresis (separate recovery expression)
Set problem expression `min(/lab-host-02/system.cpu.util,3m)>20` and recovery expression `max(/lab-host-02/system.cpu.util,3m)<10`, then generated borderline load. If problem and recovery use the same threshold, a metric sitting on the line flaps and re-pages repeatedly; separate thresholds solve it.

### Trigger dependencies
Built a "Host unreachable" trigger (`nodata(agent.ping,3m)=1`) and made a "Service down" trigger depend on it. Stopping the agent fired only the parent alert. This is the first line of defense against alert cascades — at scale, the difference between one actionable page and fifty.

### Severity-based escalation actions
Built an action with staged operations — Warning+ severity routes to one group immediately, escalates to a second group after a delay, with a recovery operation so the closure notification reaches the same people. The fix for "everyone gets everything" isn't muting Warning severity; it's routing by severity and service ownership, with a runbook link in the message body.

### Maintenance windows
Tested **with data collection** (suppresses notifications, keeps history) against **without data collection** (stops collection entirely, leaving a real gap in history). Default to "with data collection," scope tightly by host group and tags, and always set an end time — an open-ended maintenance window is how real incidents get hidden.

---

## Security & access

### Secret macros
Added `{$DB.PASSWORD}` as a **Secret text** macro — confirmed it can't be read back in the UI and doesn't appear in `configuration.export`. Plaintext macros leak through exports, backups, screenshots, and the API; "only admins can see it" isn't a real control. Credentials found in plaintext during an audit should be rotated, not just hidden, since they have to be treated as already compromised.

### Least-privilege RBAC
Created a user role with read-only UI access but **Acknowledge** permission enabled and config-write disabled, scoped to a single host group. Logged in as that user to confirm they could see and acknowledge events for their hosts only, with no edit access. Separating operational rights from configuration rights means someone can ack and close events without being able to touch templates or actions — the alternative to escalating a user to "temporary Super Admin," which is how permanent access-control problems start.

---

## API & automation

Used the JSON-RPC API for host/item operations, with a fixed discipline for bulk changes: dry-run with a `.get` call to confirm exactly what's in scope, `configuration.export` as a rollback point, diff the intended change, batch it in small transactions, then validate afterward with another `.get` plus item support state and latest data.

```bash
# auth
curl -s -X POST http://localhost:80/api_jsonrpc.php \
 -H 'Content-Type: application/json-rpc' \
 -d '{"jsonrpc":"2.0","method":"user.login","params":{"username":"Admin","password":"zabbix"},"id":1}'

# scope check first (dry run)
curl -s -X POST http://localhost:80/api_jsonrpc.php \
 -H 'Content-Type: application/json-rpc' \
 -d '{"jsonrpc":"2.0","method":"host.get","params":{"output":["hostid","host","status"]},"auth":"TOKEN","id":2}'

# export as rollback point
curl -s -X POST http://localhost:80/api_jsonrpc.php \
 -H 'Content-Type: application/json-rpc' \
 -d '{"jsonrpc":"2.0","method":"configuration.export","params":{"options":{"hosts":["10084"]},"format":"yaml"},"auth":"TOKEN","id":3}'

# then the change itself: host.update / host.massupdate
```

---

## Proxy architecture

Deployed a Zabbix proxy, moved a host to it, then deliberately took it offline:
```bash
docker run -d --name zbx-proxy --network zabbix-docker_backend \
  -e ZBX_HOSTNAME="proxy-lab" -e ZBX_SERVER_HOST="zabbix-server" \
  -e ZBX_PROXYMODE="0" zabbix/zabbix-proxy-sqlite3:alpine-latest

docker stop zbx-proxy     # wait 5-10 min
docker start zbx-proxy    # watch buffered data arrive late
```
Watched `zabbix[proxy,"proxy-lab",lastaccess]` and the graph backfill on reconnect. The proxy buffers locally and backfills on reconnect, so after an outage some metrics show gaps and some don't — it depends on `ProxyLocalBuffer`/`ProxyOfflineBuffer` sizing, how long it was offline, item type, and collection interval (high-frequency items overflow the buffer first). Architecturally: a proxy solves a *collection* problem (remote sites, network segmentation, poller saturation); scaling the server solves a *processing* problem.

## Retention & housekeeping

Set per-item retention (History = 7d, Trends = 365d) instead of relying on global defaults, based on item value, troubleshooting need, trend usefulness, and compliance need. At real scale, the internal housekeeper isn't the right tool for history/trends — partitioning (TimescaleDB or native) and dropping partitions is instant, where a mass `DELETE` is what spikes the DB at peak polling.

## Self-monitoring

Built a dashboard tracking Zabbix's own health: `zabbix[queue,10m]`, `zabbix[process,poller,avg,busy]`, `zabbix[process,preprocessing manager,avg,busy]`, `zabbix[rcache,buffer,pused]`, `zabbix[vcache,buffer,pused]`. Queue depth is the first signal to check — if it's climbing while server CPU is normal, that points at collection capacity (poller utilization, unreachable pollers starving the pool, timeouts, intervals too tight, proxy capacity) rather than compute.

## Webhook integration

Cloned a webhook media type and pointed it at a test endpoint to inspect the outbound payload. For real ITSM integrations, the part that matters is idempotency: an external key mapping the Zabbix event to the ticket, event lifecycle mapping so recovery closes the right ticket, and retry-safe payloads — otherwise action retries create duplicate tickets and recovery closes the wrong one.

---

## Where this leaves me

Two layers of Zabbix experience: **in production at PTC**, I own the alert lifecycle — building and validating monitoring for Fortune 500 customer go-lives, tuning triggers, managing alert quality, and troubleshooting the collection pipeline (unsupported items, agent/SNMP issues, Linux-level checks) across 200+ servers and 50+ enterprise environments.

**Beyond that**, this lab is where I do the platform-level work production doesn't ask of me: LLD filters, dependent items with JSONPath preprocessing, escalation chains, host-level macro overrides, secret macros, least-privilege roles, API operations with a rollback discipline, and a proxy I've deliberately disconnected to see buffering behavior firsthand.

What I'm still building: designing a Zabbix estate from scratch at large scale, and deeper API/IaC automation (Terraform provisioning is the next piece of this lab).
