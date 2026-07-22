# 🔧 Operating Zabbix — Practical Scenarios

The main [README](README.md) documents what's built and verified in this
specific lab. This doc is different: it's how I'd actually approach common
Zabbix operational situations day to day — the "how do you work with it"
questions, not the theory. Grounded in the production alert-lifecycle
ownership across 50+ Fortune 500 environments this lab is meant to extend.

---

## Basic

### "A host shows a red exclamation mark / no data"

- Check `zabbix_agentd.log` on the host first (or `zabbix_get` from the
  server to isolate: is the agent unreachable, or is the *item* the problem?).
- `zabbix_get -s <host> -k agent.ping` from the server — if that fails, it's
  network/agent, not Zabbix configuration. Firewall (port 10050), agent
  service down, or `Server=`/`ServerActive=` misconfigured in
  `zabbix_agentd.conf` are the usual three.
- If `agent.ping` works but a specific item still shows no data: check the
  item's own error message in **Latest Data** — it's almost always stated
  plainly there (permission denied, key doesn't exist, wrong parameters).

### "An item is showing 'Not supported'"

- Read the actual unsupported reason on the item (Latest Data → item →
  hover/click the red icon) before guessing — it tells you directly (e.g.
  wrong key syntax, missing permission, module not loaded).
- Reproduce it manually with `zabbix_get -s <host> -k '<the exact key>'`
  outside of Zabbix — isolates whether it's the agent, the key, or a
  permissions issue on whatever the key is trying to read.
- Common causes in practice: a typo'd key parameter, a monitored path/file
  that doesn't exist on that particular host, or an agent running an older
  version than the key requires.

### "I need to onboard a new host quickly"

- Link it to an existing template rather than hand-building items — that's
  the entire point of templates: config should be inherited and consistent
  across the estate, not recreated per host.
- After linking: confirm in Latest Data that items are actually populating
  (a host that "exists" in Zabbix but isn't collecting is a monitoring gap
  that looks fine on a dashboard and isn't).
- At scale (10s–100s of hosts), this is what host groups + template linking
  via the API is for, not clicking through the UI per host — see the
  onboarding-at-scale question below.

---

## Intermediate

### "We're getting too many alerts / alert fatigue"

- First question is always: is the *threshold* wrong, or is the *system*
  actually unstable? Tuning a bad threshold up just to silence noise hides
  a real problem if the system genuinely is flapping.
- If the metric legitimately hovers near the threshold: use **hysteresis**
  — separate problem and recovery expressions (e.g. problem at >90%,
  recovery at <80%) so it doesn't re-fire every time the value crosses one
  line back and forth.
- If it's noisy because of normal transient spikes: a short-duration
  average (`avg(/host/key,5m)>threshold`) instead of `last()` smooths those
  out without hiding a sustained problem.
- Trigger dependencies matter here too — if a host is unreachable, you
  want the "host down" alert, not twenty child-service alerts firing
  independently for the same root cause.

### "How do you onboard hosts at scale instead of one at a time?"

- Via the API (`host.create`, `hostgroup.get`, `template.get`) driven by
  a CMDB/inventory source, or via Ansible/Salt calling that same API —
  not clicking through the UI per host. (See this lab's own automation
  work: [ansible-zabbix-baseline](https://github.com/dineshravichandiran/ansible-zabbix-baseline)
  onboards a host into monitoring as part of a broader baseline playbook.)
- Always dry-run first (`host.get` to confirm what already exists) before
  a bulk `host.create`/`host.massupdate` — bulk operations without a
  pre-check are how you accidentally duplicate or overwrite hosts across
  an entire estate.
- Bulk changes go out in small batches with validation between batches, not
  one enormous API call against production.

### "A trigger fired but there's nothing actually wrong"

- Check what data the trigger actually evaluated at that timestamp — a
  single bad/spiked data point (agent hiccup, brief network blip) can fire
  a `last()`-based trigger that a `avg()`/`min()` over a short window
  wouldn't have.
- Check for a recent template or item change — a threshold that was fine
  last month may not fit if the underlying workload changed (more traffic,
  new deployment, seasonal load).
- If it's genuinely a one-off and the threshold is otherwise sound, that's
  not a reason to weaken the trigger — it's a reason to look at whether a
  **nodata()** or dependency guard should suppress it during known
  transient conditions instead.

---

## Advanced

### "How do you scale monitoring across multiple sites/regions?"

- Zabbix proxies — deploy one per site/region so hosts report locally and
  the proxy batches data back to the central server. This also means a
  site losing connectivity to the central server doesn't lose monitoring
  during the outage — the proxy buffers and forwards once reconnected.
- Sizing the buffer (`ProxyLocalBuffer`/`ProxyOfflineBuffer`) matters: too
  small and you lose history during a longer outage; understanding how long
  an outage can realistically last for that site drives the number, not a
  default guess.
- Centralize template and trigger management at the server level so every
  site inherits the same monitoring standard — proxies shouldn't become an
  excuse for each site to diverge on what "monitored" means.

### "How do you validate monitoring is actually working, not just configured?"

- Configuration existing isn't the same as monitoring working — validate
  the full loop: item actually populating in Latest Data with fresh
  timestamps, the trigger actually evaluating (not just enabled),
  and the notification actually reaching the right people (test-fire it,
  don't assume the action config is correct because it saved without
  error).
- For a new customer go-live specifically: this is the exact discipline
  from production — set up monitoring *before* go-live, then explicitly
  confirm each layer (collection → trigger → escalation → notification)
  is live and enabled rather than trusting that "it's configured" means
  "it will page someone."

### "How do you handle secrets (DB passwords, API tokens) in Zabbix configs?"

- Zabbix's **Secret text** macro type — value isn't readable back through
  the UI and doesn't appear in `configuration.export`, unlike a plain-text
  user macro which leaks through exports, backups, and screenshots.
- Scope macros as narrowly as possible (host-level over global where
  practical) so a leak or misconfiguration has the smallest possible blast
  radius.

### "What's your actual process for a bulk configuration change?"

This is the one that separates "knows the commands" from "has actually done
this against something that matters":

1. `*.get` first — confirm exactly what you're about to change and how many
   objects it touches. Never `massupdate` against an assumption.
2. `configuration.export` as a rollback point before touching anything.
3. Change a small batch, verify the result, then continue — not the whole
   estate in one call.
4. Re-validate after — same checks as the "is monitoring actually working"
   question above, because a bulk change succeeding without an API error
   doesn't mean every host ended up in the state you intended.
