# Zabbix Hands-On Gap Closer — Do It, Then Say It
**Dinesh Ravichandiran** · Do these before the interview · Each task unlocks a specific answer

> **How this works:** Left = what you do. Right = what you can then honestly say. Do the task, take 3 lines of notes, and the answer stops being recited and starts being remembered. Interviewers hear that difference instantly.
>
> **Time budget:** Tier 1 = 3 hours (do these no matter what). Tier 2 = 3 hours. Tier 3 = if you have a full day.

---

## SETUP (20 min)

```bash
git clone https://github.com/zabbix/zabbix-docker.git
cd zabbix-docker
docker compose -f docker-compose_v3_alpine_pgsql_latest.yaml up -d
# Frontend: http://localhost:8080   Login: Admin / zabbix
docker compose ps                 # confirm server, db, web, agent up
docker compose logs -f zabbix-server | head -50
```

Add a second agent to have something to monitor:
```bash
docker run -d --name agent2 --network zabbix-docker_zbx_net_backend \
  -e ZBX_SERVER_HOST="zabbix-server" -e ZBX_HOSTNAME="lab-host-02" \
  zabbix/zabbix-agent2:alpine-latest
```

**Notes to keep:** version deployed, DB backend, how long it took, anything that broke.

---

# TIER 1 — DO THESE (3 hours, highest interview probability)

### 1.1 Add a host + link a template (15 min)
**Do:** Data collection → Hosts → Create host → name `lab-host-02`, agent interface, link **Linux by Zabbix agent** → wait 2 min → Monitoring → Latest data → confirm values flowing.

**Then say:**
> "When I add a host I link a template rather than building items by hand — config should be inherited, not hand-crafted, so it stays consistent across the estate. After linking I check Latest data to confirm items are actually supported and fresh, because a host that exists but isn't collecting is a blind spot."

### 1.2 Break an item on purpose → fix it (20 min) ⭐ **highest value**
**Do:** Edit any item, change the key to `system.cpu.utilXXX` → wait → item goes **unsupported** → open it, read the error → test from CLI:
```bash
docker exec -it zabbix-server zabbix_get -s agent2 -k system.cpu.utilXXX   # fails
docker exec -it zabbix-server zabbix_get -s agent2 -k system.cpu.util      # works
```
→ fix the key → watch it go supported.

**Then say:**
> "I've deliberately broken items in my lab to practise this. The workflow is: read the unsupported reason on the item first — it usually tells you exactly what's wrong — then reproduce it outside Zabbix with `zabbix_get` to isolate whether it's the agent, the key, or permissions. I never disable an unsupported item; unknown isn't healthy, it's a blind spot with a green icon."

### 1.3 Write a trigger + a nodata trigger (20 min)
**Do:** Create trigger `last(/lab-host-02/system.cpu.util)>20` severity Warning → generate load:
```bash
docker exec -it agent2 sh -c "yes > /dev/null &"
```
→ watch it fire → kill it → watch recovery. Then add:
`nodata(/lab-host-02/agent.ping,5m)=1` → stop the agent container → watch it fire.

**Then say:**
> "Two kinds of trigger matter to me: threshold triggers and freshness triggers. `nodata()` is the one people forget — if the agent dies, your CPU trigger goes quiet and everything looks fine. I've tested that in my lab by stopping the agent: without a nodata trigger, silence looks identical to health."

### 1.4 Add hysteresis (10 min)
**Do:** Set Problem `min(/lab-host-02/system.cpu.util,3m)>20`, Recovery expression `max(/lab-host-02/system.cpu.util,3m)<10`. Generate borderline load and watch it not flap.

**Then say:**
> "I use a recovery expression rather than the same threshold both ways. If problem and recovery are the same number, a metric sitting on the line flaps and pages people repeatedly. Separate thresholds solve it — I've set this up and watched the difference."

### 1.5 Trigger dependency (20 min) ⭐
**Do:** Create "Host unreachable" trigger (`nodata(agent.ping,3m)=1`) and a "Service down" trigger. On the service trigger → Dependencies → add the host trigger. Stop the agent container → only the parent fires.

**Then say:**
> "Dependencies are my first tool against alert cascades. I set this up in my lab — a service trigger depending on a host-down trigger — then killed the agent, and only the parent alert fired instead of both. At scale that's the difference between one actionable page and fifty."

### 1.6 LLD filter (30 min) ⭐⭐ **they will ask this**
**Do:** Host → Discovery rules → "Mounted filesystem discovery" → Filters tab. Look at what it discovers first (all the `/proc`, `/sys`, overlay junk). Add:
```
{#FSTYPE}  matches       ^(ext4|xfs|btrfs)$
{#FSNAME}  does not match ^(/dev|/sys|/run|/proc|/var/lib/docker|/snap)
```
→ wait for the next discovery cycle → count items before vs after.

**Then say:**
> "I've tuned this exactly. Out of the box, filesystem discovery picks up tmpfs, overlays and container mounts, and you get item churn plus noise. The fix is LLD filters on `{#FSTYPE}` and `{#FSNAME}`, plus the lifetime setting for lost resources — not disabling discovery, and not stretching the interval to hide it. In my lab that cut discovered items by more than half."

### 1.7 Action with severity-based escalation (30 min) ⭐
**Do:** Alerts → Actions → Trigger actions → Create. Condition: Severity ≥ Warning. Operations:
- Step 1 (0 min) → send to "Zabbix administrators"
- Step 2 (start 2, duration 60s) → send to a second user group
Add a Recovery operation. Fire the trigger, watch the escalation steps run.

**Then say:**
> "I've built escalation chains: severity-aligned routing with staged steps and delays, plus recovery operations so the closure notification goes to the same people. The fix for 'everyone gets everything' isn't muting Warning — it's routing by severity and service ownership, with a runbook link in the message body so the person paged knows what to do."

### 1.8 Maintenance window (20 min)
**Do:** Data collection → Maintenance → Create. Try **with data collection** first, scope to your host, fire a trigger → observe suppression. Then repeat **without data collection** → note the data gap in Latest data.

**Then say:**
> "The trap is 'with' versus 'without data collection'. 'Without' stops collection entirely, so you get a real hole in your history — and if something breaks during the window you have nothing to investigate with afterwards. I default to 'with data collection', scope tightly by host group and tags, and always set an end time. An open-ended maintenance window is how real incidents get hidden."

---

# TIER 2 — DO IF YOU HAVE TIME (3 hours)

### 2.1 Dependent items + JSONPath preprocessing (40 min) ⭐
**Do:** Create a master **HTTP agent** item hitting any JSON endpoint (e.g. `https://api.github.com/repos/zabbix/zabbix`), interval 1m. Then create 2–3 **Dependent items** with preprocessing → JSONPath: `$.stargazers_count`, `$.open_issues_count`. Watch one poll feed several metrics.

**Then say:**
> "I use dependent items where one payload contains many metrics — a master item polls once, and dependent items extract fields with JSONPath preprocessing. That turns N polls into one. I've built this in my lab with an HTTP agent master item. When preprocessing queues grow, the things I'd check are preprocessing manager load, master item interval, payload size, JSONPath efficiency, and dependent item count."

### 2.2 Preprocessing: discard unchanged with heartbeat (15 min)
**Do:** On a slow-changing item, add preprocessing step → **Discard unchanged with heartbeat** → 1h. Observe fewer values written.

**Then say:**
> "For slow-changing metrics I use 'discard unchanged with heartbeat' — it only writes when the value changes, but the heartbeat guarantees a value at least hourly so `nodata` triggers don't false-fire. It's one of the cheapest ways to cut database write volume without losing signal."

### 2.3 User macro override (20 min)
**Do:** Set a template-level macro `{$CPU.UTIL.CRIT}` = 90. Use it in a trigger: `last(/host/system.cpu.util)>{$CPU.UTIL.CRIT}`. Then override it on **one host** = 50. Confirm only that host changed.

**Then say:**
> "This is exactly how I'd handle 'one host needs a different threshold on a shared template'. You never edit the shared template — that silently changes behaviour for unrelated critical services. You override the macro at host level, because macro precedence is global → template → host, and host wins. If the divergence is structural rather than just a number, then a nested or derived template."

### 2.4 Secret macro (15 min)
**Do:** Host → Macros → add `{$DB.PASSWORD}`, change type to **Secret text**. Save. Try to read it back — you can't. Try `configuration.export` — it's not there.

**Then say:**
> "I've used secret macros. The point is that 'only admins can see it' isn't a control — plaintext macros leak through exports, backups, screenshots and the API. Secret text can't be read back in the UI and doesn't appear in exports. And when you find plaintext credentials in an audit, you don't just hide them — you rotate them, because they must be treated as already compromised, then document the control gap."

### 2.5 RBAC — least privilege (25 min) ⭐
**Do:** Users → User roles → create a role with **Read-only** UI access but **Acknowledge** permission enabled, config write disabled. Create a user group scoped to **one host group only**. Log in as that user in an incognito window — confirm they see only their hosts and can ack but not edit.

**Then say:**
> "I've built this. The key insight is separating operational rights from configuration rights — someone can acknowledge and close events without being able to change templates or actions. Permissions are scoped at host-group level to their application area. When someone escalates for admin, that's the answer: least privilege plus documented approval evidence. 'Temporary Super Admin' is how permanent problems start."

### 2.6 API — get, then a safe change (30 min) ⭐
**Do:**
```bash
# token
curl -s -X POST http://localhost:8080/api_jsonrpc.php \
 -H 'Content-Type: application/json-rpc' \
 -d '{"jsonrpc":"2.0","method":"user.login","params":{"username":"Admin","password":"zabbix"},"id":1}'

# scope check FIRST (dry run)
curl -s -X POST http://localhost:8080/api_jsonrpc.php \
 -H 'Content-Type: application/json-rpc' \
 -d '{"jsonrpc":"2.0","method":"host.get","params":{"output":["hostid","host","status"]},"auth":"TOKEN","id":2}'

# export backup
curl -s -X POST http://localhost:8080/api_jsonrpc.php \
 -H 'Content-Type: application/json-rpc' \
 -d '{"jsonrpc":"2.0","method":"configuration.export","params":{"options":{"hosts":["10084"]},"format":"yaml"},"auth":"TOKEN","id":3}'

# then a change
# host.update / host.massupdate
```

**Then say:**
> "I've used the API for host and item operations. My discipline on bulk changes: dry-run with a `.get` to confirm exactly what's in scope, `configuration.export` as a rollback point, diff the intended change, batch it in small transactions, then validate after with a `.get` plus item support state and latest data. That's how you avoid the classic 'bulk macro update broke SNMP on 200 hosts' problem."

---

# TIER 3 — IF YOU HAVE A FULL DAY

### 3.1 Add a proxy + kill it (60 min) ⭐⭐ **the standout**
**Do:**
```bash
docker run -d --name zbx-proxy --network zabbix-docker_zbx_net_backend \
  -e ZBX_HOSTNAME="proxy-lab" -e ZBX_SERVER_HOST="zabbix-server" \
  -e ZBX_PROXYMODE="0" zabbix/zabbix-proxy-sqlite3:alpine-latest
```
Administration → Proxies → add `proxy-lab` (active). Move `lab-host-02` to it. Confirm data flows. Then:
```bash
docker stop zbx-proxy     # wait 5-10 min
docker start zbx-proxy    # watch buffered data arrive late
```
Watch `zabbix[proxy,"proxy-lab",lastaccess]` and the graph backfill.

**Then say:**
> "I've run a proxy in my lab and deliberately disconnected it. The proxy buffers locally and backfills on reconnect — which is why after an outage some metrics have gaps and some don't. It depends on `ProxyLocalBuffer` and `ProxyOfflineBuffer` sizing, how long it was offline, item type, and collection interval — high-frequency items overflow the buffer first. I monitor proxy health with `zabbix[proxy,name,lastaccess]`. Architecturally, I add a proxy when it's a collection problem — remote sites, network segmentation, poller saturation — and scale the server when it's a processing problem."

### 3.2 Housekeeping + retention (30 min)
**Do:** Administration → Housekeeping — read every setting. Then on one item, set History = 7d, Trends = 365d. Then look at DB size:
```bash
docker exec -it zabbix-db psql -U zabbix -c "\dt+ history*"
docker exec -it zabbix-db psql -U zabbix -c "SELECT pg_size_pretty(pg_database_size('zabbix'));"
```

**Then say:**
> "I've tuned retention per item rather than globally — retention tiers based on item value, troubleshooting need, trend usefulness and compliance. And at real scale you don't rely on the internal housekeeper for history and trends; you partition (TimescaleDB or native) and drop partitions, because dropping a partition is instant where a mass DELETE is what spikes your DB at peak polling."

### 3.3 Self-monitoring dashboard (30 min)
**Do:** Build a dashboard with: `zabbix[queue,10m]`, `zabbix[process,poller,avg,busy]`, `zabbix[process,preprocessing manager,avg,busy]`, `zabbix[rcache,buffer,pused]`, `zabbix[vcache,buffer,pused]`.

**Then say:**
> "I monitor Zabbix with Zabbix. Queue depth is my first health signal — if `zabbix[queue,10m]` is climbing while server CPU is normal, that tells me it's collection capacity, not compute: poller utilization, unreachable pollers starving the pool, timeouts, intervals too tight, or proxy capacity. I keep poller and preprocessing busy percentages on the same dashboard so I can tell those apart in seconds."

### 3.4 Webhook media type (30 min)
**Do:** Alerts → Media types → clone the Slack or generic webhook type → point it at `https://webhook.site` (free) → attach to a user → fire a trigger → see the payload arrive.

**Then say:**
> "I've configured webhook media types and watched the payload land. For ITSM integrations the thing that matters is idempotency — an idempotent external key mapping the Zabbix event to the ticket, event lifecycle mapping so recovery closes the right ticket, retry-safe payloads and correlation fields. Otherwise action retries create duplicate tickets and recovery closes the wrong one."

---

# WHAT THIS UNLOCKS — THE HONEST FRAMING

After the lab, when they ask **"how much hands-on Zabbix have you done?"**:

> "Two layers. **In production at PTC** I own the alert lifecycle — building and validating monitoring for Fortune 500 customer go-lives, tuning triggers, managing alert quality, and troubleshooting the collection pipeline: unsupported items, agent and SNMP issues, Linux-level checks across 200+ servers. That's daily work across 50+ enterprise environments, 5,000+ incidents, 99.9% uptime.
>
> **Beyond that**, I run my own Zabbix lab, because there's platform work I want to be able to do without supervision — I've built LLD filters, dependent items with JSONPath preprocessing, escalation chains, host-level macro overrides, secret macros, least-privilege roles, API operations, and I've run a proxy and deliberately disconnected it to see buffering behaviour firsthand.
>
> Where I'd be straight with you: designing a Zabbix estate from scratch at bank scale, and deep API/IaC automation, are areas I'm still building. But I know the shape of the problem and I know how to work it out."

**That last paragraph is the one that wins the room.** It's honest, it's specific, and it tells them you know your own edges — which is exactly what "works with minimal supervision" actually means.

---

# NOTE-TAKING TEMPLATE (3 lines per task — do this)

```
Task:
What I configured:
What surprised me / what broke:
```

Those three lines are what turn "I read about it" into "I did it" in your voice. Specific detail — a number, a thing that broke, a setting you got wrong first — is what makes an interviewer believe you.

---

# PRIORITY IF YOU ONLY GET 3 HOURS

1. **1.2** Break and fix an unsupported item
2. **1.6** LLD filters
3. **1.5** Trigger dependency
4. **1.7** Escalation action
5. **2.5** RBAC least privilege
6. **3.1** Proxy + disconnect (if you can stretch)

Those six cover about 80% of what the Architect and Tech Lead will probe.
