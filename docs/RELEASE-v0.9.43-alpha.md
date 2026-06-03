# v0.9.43-alpha — CloudTAK Dispatcher plugin + webadmin LDAP spiral deploy hardening

**Date:** 2026-06-03
**Type:** New feature + deploy reliability fix — drop-in via Update Now.
**Status:** RELEASED to `main` 2026-06-03. Validated on Azure (public IP + LE cert, clean fresh deploy end-to-end).

---

## TL;DR

Two headline areas:

1. **CloudTAK Dispatcher plugin** — a Computer-Aided Dispatch panel inside CloudTAK, deployed from its own public repo (like the ping plugin). Works **standalone** (no TAK-CAD server plugin required): incidents are server-backed in CloudTAK's own Postgres and drawn on the map as native CoT, with responder notification over mission thread + direct message. Auto-upgrades to full TAK-CAD mode when the TAK-CAD server plugin is detected.
2. **webadmin LDAP spiral deploy hardening** — fresh TAK Server installs on boxes that can't reach FQDN/Caddy routing (no public IP / no LE cert / slow disk) were dead-ending at the final `webadmin` LDAP-bind verification with `exceeded stage recursion depth` — a flow spiral, not a bad password. The deploy now repairs the flow before retrying, no longer destroys the webadmin user on a spiral verdict, and no longer aborts the whole deploy on a verify miss.

---

## What's new

### 1. CloudTAK Dispatcher plugin (`tak-dispatcher`)

**Source repo:** `https://github.com/takwerx/cloudtak-dispatcher-plugin` (public)
**Catalog entry:** `CLOUDTAK_PLUGINS` in `app.py` (~line 15300)

A dispatcher panel inside CloudTAK, installed from the CloudTAK → Plugins page. Deployed like the ping plugin: the installer clones the public repo, copies `plugin/` into `web/plugins/` and `server/*.ts` into CloudTAK's `api/routes/` (the server files must NOT land under `web/plugins/` or CloudTAK's `vue-tsc` web build would type-check server-only imports).

**Standalone mode (no TAK-CAD server plugin needed):**
- Incidents/Events are **server-backed in CloudTAK's own Postgres** via a CloudTAK API route (`plugin-dispatcher.ts`) — shared across all dispatchers on that CloudTAK, not per-browser.
- Incidents are drawn on the map as **native CloudTAK CoT**, optionally written into a chosen **DataSync feed/mission** so every client (ATAK / iTAK / TAK Aware / WinTAK) sees them.
- Incident markers use the Incident Management usericon via a **foldered `iconsetpath` with no marker-color** so they render correctly on every field client (white marker-color washed the icon out on ATAK/TAK Aware; a flat color-less key renders everywhere).
- **Responder assignment:** multi-select contacts picker (vehicles + personnel), assignment notes, and notification of assigned responders over the **mission thread + direct message** on incident create/note.
- Uses the signed-in **CloudTAK callsign** as the dispatcher identity (no "who are you" prompt).
- Theme-aware (adapts to CloudTAK light/dark).

**TAK-CAD mode (auto-detected):** when the TAK-CAD server plugin is present on TAK Server, the panel upgrades to the full CAD feature set (incident/vehicle/personnel CRUD against the server plugin's `/Marti/api/plugins/.../submit` API) via the optional `plugin-takcad.ts` proxy route, using the fully-qualified class-name path and CloudTAK's `std()` auth.

### 2. webadmin LDAP spiral — deploy hardening

Root cause (see `docs/HANDOFF-LDAP-AUTHENTIK.md`): `exceeded stage recursion depth` is a flow-execution spiral, **not** a credential failure. `webadmin` is a fresh bind with no cached session, so it hits the spiral cold while the LDAP outpost is on internal direct routing — the routing exposed on boxes with no public IP / no LE cert (Starlink/CGNAT) and worsened by slow disk (Proxmox/bare-metal). Cloud boxes (AWS/Azure/SSDNodes) get a public IP + LE cert → FQDN routing → never spiral, which is why this surfaced only in the field.

Three fixes:
- **Deploy repairs the flow before giving up.** When webadmin sync attempt 1 fails, `run_takserver_deploy()` now runs `_ensure_ldap_flow_authentication_none()` (clears `password_stage` on the identification stage, fixes the 3 `ldap-*` stage bindings, force-recreates the outpost — same as the "Connect TAK Server to LDAP" button) before retrying. Idempotent on healthy boxes (attempt 1 succeeds, repair never runs).
- **Verify no longer destroys webadmin on a spiral.** A spiral surfaces to `ldapsearch` as LDAP 49 "invalid credentials". `_test_ldap_bind_dn_verdict` returned `'fail'` on that alone, which triggered a destructive DELETE+recreate of webadmin that wiped the cached session that would otherwise succeed — locking the box into a cold-spiral loop. It now checks the outpost log for the spiral marker first and returns `'inconclusive'` (no destructive action) when present. A genuine bad password has no spiral marker and still fails.
- **A verify miss no longer aborts the deploy.** Previously a webadmin-verify miss set `error` + `return`, skipping Guard Dog, the LE-cert step, and the completion banner. The install itself is fine, so the deploy now completes with a clear warning pointing the operator at "Connect TAK Server to LDAP" + the 8446 login test.

---

## CloudTAK plugin install/update robustness

Supporting fixes that make plugin install survive a CloudTAK rebuild/update:
- Plugin install **copies** `local_path` source instead of symlinking (Vite/`import.meta.glob` bundles the copy; symlink broke on rebuild).
- CloudTAK **Update restores plugins** that a `git checkout` would otherwise wipe from `web/plugins/`.
- CloudTAK **version check uses the container ID**, not a hardcoded container name.
- CloudTAK **Update no longer hard-fails** when the GitHub tag fetch fails (network).
- Plugins page **guards `p.repo`** for local-path plugins (no crash on entries without a `repo`).

---

## Changes (high level)

| Area | Change |
|------|--------|
| `app.py` `CLOUDTAK_PLUGINS` (~15300) | `tak-dispatcher` catalog entry: public `repo`, `plugin_subpath`/`server_subpath`, requires CloudTAK 13.2+ |
| `app.py` `_run_cloudtak_plugin_action` / `_detect_cloudtak_plugins` (~16091/16052) | copy-not-symlink, `server_subpath` route copy, restore-after-update, container-ID version check, GitHub-tag-fetch tolerance |
| `app.py` `run_takserver_deploy()` (~46064) | repair LDAP flow on webadmin sync spiral before retry; verify miss is non-fatal |
| `app.py` `_test_ldap_bind_dn_verdict` (~39198) | spiral marker downgrades ldapsearch LDAP-49 from `'fail'` to `'inconclusive'` (no destructive recreate) |
| `cloudtak-dispatcher-plugin` (separate public repo) | full Vue plugin + CloudTAK server routes (`plugin-dispatcher.ts` standalone store, `plugin-takcad.ts` optional proxy) |
| `app.py` `VERSION` | `0.9.42-alpha` → `0.9.43-alpha` |

---

## Validation

- **Azure (public IP + LE cert):** clean fresh deploy end-to-end — `✓ webadmin synced to Authentik` on attempt 1 (repair correctly did NOT fire), `proactive routing: outpost already on FQDN`, LE cert installed on 8446, Guard Dog installed, `✓ DEPLOYMENT COMPLETE`. Confirms no regression on the happy path.
- **Heal path** (spiral → repair → retry → synced) targets the field-reported no-public-IP / slow-disk class; manual remediation on an already-deployed box is the "Connect TAK Server to LDAP" button.

---

## Scope discipline — what is NOT in this release

- TAK-CAD **server** plugin is still an external prerequisite for full CAD mode (standalone mode needs nothing extra).
- No change to the LE/Caddy cert flow for CGNAT/no-public-IP boxes — those operators should use self-signed mode or a DNS-01 challenge (tracked separately).
- No TAK Server-side fixes for the known CAD-mode gaps (`incidentstatus` not persisted on update, HTTP DELETE bridge) — those are upstream RTX items.
