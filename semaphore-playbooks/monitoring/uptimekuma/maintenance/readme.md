# Uptime Kuma Setup With Maintenance Eindows

Optional add-on to the PatchMon/Semaphore/n8n orchestrator. See [`Jan's tutorial - part 3`](https://bachelor-tech.com/automate-patching-with-semaphore-ui-patchmon-n8n/part-3-set-up-uptimekuma-for-monitoring-for-maintenance-windows-during-patching#step-3-encrypt-uptime-kum). Off by default; nothing
here is required for patching to keep working. This puts the affected host's
monitors into an Uptime Kuma maintenance window (across however many Kuma
instances you run) for the duration of a patch run, so a reboot or service
restart never fires a false "down" alert or wakes up the infra auto-remedy
workflow.

## Why no custom sidecar service

Uptime Kuma's maintenance windows are only writable over its Socket.IO API, not a
stable public REST API — so this can't be a plain `curl`/`ansible.builtin.uri`
call like the existing Proxmox tag read/writes. Rather than building and hosting
a custom microservice to bridge that, this uses
[`lucasheld/ansible-uptime-kuma`](https://github.com/lucasheld/ansible-uptime-kuma),
an actively maintained Ansible collection built on the same author's
[`uptime-kuma-api`](https://github.com/lucasheld/uptime-kuma-api) Python
library. Ansible talks to it as native modules — no new always-on service to
deploy, monitor, or keep patched.

## 1. Install dependencies (on the Semaphore control host)

```bash
pip install uptime-kuma-api jmespath
ansible-galaxy collection install lucasheld.uptime_kuma
```

`jmespath` is needed for the `json_query` filter the playbook uses to match
monitors by tag. Collection version compatibility (from the project's own docs):

| Uptime Kuma    | Collection    |
|----------------|---------------|
| 1.21.3 – 1.23.2| 1.0.0 – 1.2.0 |
| 1.17.0 – 1.21.2| 0.1.0 – 0.14.0|

Check your running Kuma version and pin the collection accordingly if needed.

## 2. Create a dedicated Kuma user on each instance

On each of your 3 instances, create a login just for this automation (not your
own admin account) — least privilege, easy to rotate/revoke independently.
Give it whatever role lets it read monitors and manage maintenance windows.

## 3. Tag the monitors you want protected

In every Kuma instance, tag each monitor that represents something running on a
given host with that host's exact hostname — the same string PatchMon reports
and Ansible/Semaphore uses as `inventory_hostname`. Case matters as written
(the playbook's tag match is currently exact; adjust
`uptimekuma_maintenance_one_instance.yml`'s `json_query` if you want
case-insensitive matching).

Example for `web1`, per your description: the ping monitor in uptimekuma1, plus
`web1-syncthing`, `web1-docker-monitor-termix`, and
`web1-docker-monitor-vaultwarden` — all tagged `web1`, regardless of which of
your 3 Kuma instances each one lives in. A host with nothing tagged on a given
instance is simply skipped there (not an error).

## 4. Store credentials in vaulted group_vars

Copy `group_vars_example/uptimekuma_vault.yml.example` into your real vaulted
group_vars (e.g. `group_vars/all/vault.yml`) and fill in real values, then
`ansible-vault encrypt` it. Keyed by **instance name** (3 entries total, not one
per monitored host) — the `name` fields must exactly match the `name` fields
you'll put in n8n's `UPTIMEKUMA_INSTANCES` in step 6.

## 5. Create the new Semaphore Task Template

Same shape as your existing "PatchMon Tag Host" template:
- Playbook: `uptimekuma_maintenance.yml`
- Connection: local (never touches the target host over SSH)
- Ansible prompts: enable both **Limit** and **Environment/Extra Variables**
  (n8n supplies `limit` = hostname, and `environment` = the
  `uptimekuma_action`/`uptimekuma_duration_minutes`/`uptimekuma_instances` JSON,
  exactly like the existing Tag Host template already does for
  `patchmon_tag_action`).

Note its numeric template ID for step 6.

## 6. Fill in the orchestrator's Global Config

Open the updated `PatchMon Auto-Patch Orchestrator.json` after import and edit
the **Global Config** node:

- `SEMAPHORE_TEMPLATE_UPTIMEKUMA_MAINTENANCE` — the template ID from step 5.
- `UPTIMEKUMA_INSTANCES` — already pre-filled with your 3 instances from this
  conversation:
  ```json
  [
    {"name": "uptimekuma1", "url": "http://192.168.8.60:3001"},
    {"name": "uptimekuma2", "url": "http://192.168.6.60:3001"},
    {"name": "uptimekuma3", "url": "http://hetzner-witness.bachelor-tech.com:3001"}
  ]
  ```
  **Double-check the uptimekuma2 URL** — the value you sent had a copy/paste
  rendering artifact (`[192.168.6.60](http://192.168.8.60:3001/):3001`), so I've
  used `http://192.168.6.60:3001` as the clean read of it. Fix it here if that's
  wrong. This list is the *only* place you manage which Kuma servers get called —
  add a 4th instance here, add its matching `uptimekuma_credentials` entry in
  vaulted group_vars, done.
- `UPTIMEKUMA_DURATION_BUFFER_MINUTES` — defaults to 10. The Kuma-side window
  auto-expires after `POLL_TIMEOUT_MINUTES_PATCH + this value` minutes even if
  the whole patch run crashes and the Stop task never fires — your dead-man's
  switch instead of a tag that can get stuck.
- `UPTIMEKUMA_MAINTENANCE_ENABLED` — set to `true` only once steps 1–5 above are
  actually done. Leave it `false` while you're setting this up; every gate in
  the workflow defaults to routing straight past the feature when it's off.

## 7. Smoke-test before trusting it live

Run the new Semaphore template by hand once (Limit = some real hostname,
Environment = `{"uptimekuma_action": "start", "uptimekuma_duration_minutes": 5,
"uptimekuma_instances": [...]}`) against a throwaway tag before flipping
`UPTIMEKUMA_MAINTENANCE_ENABLED` to `true` in the live orchestrator. In
particular verify:

- the monitor's `tags` shape coming back from `monitor_info` actually has a
  `name` key the way `uptimekuma_maintenance_one_instance.yml` assumes
- the `maintenance` module accepts `strategy: single` with just `dateRange`
  (no `timeRange`) for an immediate one-off window
- `state: absent` keyed on `title:` correctly deletes without needing the
  numeric maintenance `id`

All three are drafted from the collection's public wiki docs, not confirmed
against your exact Kuma + collection version — same caveat already flagged
throughout the rest of this project's Semaphore integration. If any of them
don't hold, the fix is local to
`uptimekuma_maintenance_one_instance.yml`; nothing else changes.

## Files in this delivery

- `uptimekuma_maintenance.yml` — entry playbook, loops over configured
  instances.
- `uptimekuma_maintenance_one_instance.yml` — per-instance login /
  monitor lookup / maintenance toggle, included by the above.
- `group_vars_example/uptimekuma_vault.yml.example` — credentials template.
- `PatchMon AutoPatch Orchestrator.json` — your orchestrator with the new gated
  Start/Stop nodes wired in. Back up your current one before overwriting.
