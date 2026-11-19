# pisetup plan

Status: plan only, nothing is built. Written 2026-11-19.

`pisetup` is **venue-neutral**: it works for any venue running the Pi ecosystem (a church, a school hall, a theatre), not for one site. It sets up a whole Pi ecosystem install: it finds the compatible devices on the network and guides their setup, installs and configures the `pi*` apps, and (later) guides building the venue's 3D model. It has two front ends, a **CLI** and a **web setup UI**, over **one shared core**, so both do exactly the same things.

## 1. Rules this plan follows

- **Public repo, no site data, no single venue.** pisetup holds no IPs, host names, credentials or venue details, and nothing in the core assumes one venue, one church or one device set. Everything site-specific is read from a *site profile* that each venue keeps in its own private repo or folder (section 4). The first venue's profile lives in private `jsilvanus/keskuskirkko`, but that is only an example of where a profile can come from.
- **Venue differences are data or plugins, never core code.** Different gear (another switcher, another DSP, no AMX at all) means a profile that lists other devices and, where needed, another probe plugin. Church-specific content (hymns, liturgy tooling) is not pisetup's concern.
- **Reuse, do not reinvent.** The app catalog (ports, health paths, env port variables) comes from piecosystem's registry (`ecosystem.manual.yaml` plus each repo's `package.json` `pi` block, merged by `scripts/generate-ecosystem.mjs`). Health checks reuse the logic of `scripts/ecosystem-health.mjs`. Service units are the generic `deploy/*.service` files each repo ships (user units, `%h/apps/<repo>`, `EnvironmentFile`). Backup and restore of config is `tools/pi-backup` (see the kirkonpi SD backup plan), not a new mechanism. The site profile schema is piecosystem's `docs/site-profile-spec.md` (`venue-profile.yaml`).
- **Every app must work without pigateway.** pisetup installs apps independently; pigateway is a normal catalog entry, never a prerequisite.
- **Deploy from `main` only.** Apps are git checkouts of `main` under `~/apps/<repo>`. pisetup never checks out a feature branch.
- **Least privilege, no silent changes.** Read-only by default. Anything that writes to a device or to the Pi is shown as a plan first and applied only on confirmation.
- Stack: Node 24, ESM, pnpm, few dependencies, OpenAPI + CLI like the other `pi*` repos.

## 2. Architecture

```
            CLI (pisetup ...)          Web UI (pisetup serve)
                   \                        /
                    +---- core (library) ---+
                    |  profile  | catalog   |
                    |  discover | checks    |
                    |  plan     | apply     |
                    +-----------+-----------+
                          |           |
              devices on the LAN    the Pi (local or over ssh)
```

- **core** (`src/core`, ESM library, no I/O at import time). The only place that knows how to do anything. Small typed API: `loadProfile`, `loadCatalog`, `discover`, `check`, `plan`, `apply`.
- **CLI** (`bin/pisetup`). A thin wrapper; every command maps to one core call and can print JSON (`--json`) for scripts and for testing the web UI.
- **Web UI** (`pisetup serve`). A small local HTTP server (Fastify, like the sibling apps) with an OpenAPI description, serving a static UI that calls the same core. It binds to localhost by default; LAN access needs an explicit flag plus a PIN, following picontrol's PIN rule. The UI has no logic of its own: wizard pages render core results (devices, checks, plan diff) and send "apply this step" back.
- **Runs where.** On the operator PC (finds devices, reaches the Pi over ssh) or on the Pi itself (local mode). Same code; a *target* setting picks local or ssh.
- **Step model.** Everything is a *step*: `{ id, title, check(), plan(), apply(), verify() }`. `check` is read-only and safe to rerun, `plan` returns what would change, `apply` does it, `verify` proves it worked. Steps are idempotent. Both front ends only list, run and report steps, which keeps them identical.
- **State.** A local state file (outside the repo, `PISETUP_STATE_DIR`) records which steps passed and when. It is a cache, never the source of truth; `check` always re-reads reality.

## 3. What it does

### 3.1 Discover devices
Read-only network discovery on the subnet from the profile, cheapest first:
1. **Expected devices from the profile**: probe each known address with the device's own protocol check. Most reliable, no scanning.
2. **Active scan** (opt-in, `pisetup discover --scan`): ping/ARP sweep of the profile subnet plus targeted port probes, rate-limited. Finds devices the profile does not list and suggests adding them.
3. **Passive hints**: neighbour table, mDNS/SSDP where a device announces itself, switch MAC table via pipoe's SNMP read when pipoe is available.

Each device type is a small **probe plugin** (`src/probes/<type>.js`): what to try, what a positive looks like, which pi* app owns it. Initial set, from the first venue's equipment; a venue without a given device type simply omits it from its profile, and new types are added as plugins (including third-party ones loaded from a plugin directory):

| Device type | Probe | Owning app |
|---|---|---|
| PTZ cameras | HTTP CGI identity, VISCA-over-IP reachable | picam / pigateway |
| ATEM switcher | UDP 9910 handshake | piswitch / pigateway |
| Video matrix (for example Atlona) | its own control API, or via a control system such as AMX where the venue routes through one | pigateway, venue control system |
| Control system (for example AMX), optional | control port reachable | pigateway |
| Soundweb London | TCP 1023 | pisound |
| PoE switch | SNMP read | pipoe |
| Projectors | projector control port | pimatrix / pigateway |
| Tally lights | UDP/HTTP | pitally |
| Encoder | HTTP API | pigateway / piencoder |
| MediaMTX | RTSP 8554, API | pimatrix |
| Operator/assisting PC services | fffleet, STT, ollama health | piecosystem deploy stacks |

Probes only identify; they never change anything. Credentials a probe needs (CGI, SNMP community) come from the profile's secret references (section 4), never from the repo.

### 3.2 Guide device setup
For each discovered or expected device the wizard shows a checklist of `check` steps, for example: reachable, firmware known, address matches the profile, PoE port budget (via pipoe), device-side settings the apps need (FreeD target, camera stream settings, ATEM address). Where a setting can be changed safely through the device API, the step offers an `apply`. Where it cannot (AMX program upload, a front-panel menu), the step is a **guided manual step**: clear instruction, then a `verify` that confirms it was done. Nothing destructive (reset, firmware flash, PoE power cycling) runs without a per-action confirmation, and the tally lamp's reset endpoint is never called.

### 3.3 Install and configure the apps
From the catalog and the profile, pisetup builds a **role-based plan**: which apps run on which host (Pi, operator PC, assisting PC, VPS), following piecosystem's `deployment-plan` and `deploy/` stacks. Per app, per host:
1. **Prerequisites**: Node 24, pnpm, ffmpeg, Docker (operator PC), checked and reported. Installing system packages is a plan item that needs confirmation.
2. **Checkout**: clone or fast-forward `~/apps/<repo>` on `main`, install dependencies, build. Refuses a dirty tree or a non-main branch.
3. **Config**: generate the app's env file from a template plus the profile (ports, device addresses, which app talks to which, API keys). Secrets are generated locally (per-app keys, least privilege) or taken from a secret reference, written with mode 600, and never printed or logged. Port clashes are detected up front from the catalog (clashes are known on shared hosts).
4. **Service**: install the repo's generic user unit (`deploy/<app>.service`) into `~/.config/systemd/user`, `daemon-reload`, enable, start, linger on.
5. **Verify**: wait for the health endpoint (same check as `ecosystem-health`), then run the checks from piecosystem's `inter-service-matrix` (A reaches B with the key it was given). Apps that claim to work without pigateway get an explicit "without pigateway" check.

Updates reuse the same steps (`pisetup update <app>`: fast-forward main, rebuild, restart, verify, roll back to the previous commit if verify fails). Config backup before changes uses `tools/pi-backup` when installed.

### 3.4 Placeholder: church 3D model
Reserved, not planned yet. A calibration approach is likely coming and will make the scope clearer. Known today, so the hook exists without committing to a design: the model lives in pispace (venue JSON), its editor is church-3d-mapper, and each venue's model data stays in that venue's own private repo. pisetup will later add a "3D model" wizard section that checks pispace and the editor are installed and pointed at the right data, then hands over to the calibration flow. No steps, schema or UI in phases 0 to 4.

## 4. Site profile (how pisetup reads site data)

- **Decision (Juha, 2026-11-19): the profile format lives in piecosystem.** The schema, its version and the documented example are owned by piecosystem (`docs/site-profile-spec.md`); every `pi*` app and pisetup validate against that one definition, so all venues share one format. Real profiles (the data) stay in each venue's own private repo, never in piecosystem or pisetup. If pisetup needs a field the schema lacks, the change goes to piecosystem first. pisetup ships only a generated copy of the schema for offline validation and the synthetic example profiles for tests.
- Each venue has one profile, `venue-profile.yaml` (schema in piecosystem `site-profile-spec.md`, versioned with `schema:`), kept in that venue's own private repo or folder. The first venue's is in private `jsilvanus/keskuskirkko`. pisetup never stores a real profile in this repo.
- **Several venues on one machine** (for example an integrator maintaining a few sites): `pisetup venue add|list|use` keeps named profile sources in the local config, and every command takes `--venue <name>`. State and secrets are kept per venue, so one site's data never mixes with another's.
- **New venue from scratch:** `pisetup venue init` runs the discovery wizard and writes a starter profile for the operator to review and commit to their own repo, so a venue without a profile can still begin.
- **Source**, in order: `--profile <path>` or `PISETUP_PROFILE_PATH` (a local checkout of the venue repo, for example on the operator PC); `PISETUP_PROFILE_URL` (pigateway's `/api/v1/venue/profile` once it is up); or a clone/pull of the private repo with the operator's existing Git credentials. The Pi gets no new GitHub credential; on the Pi the profile arrives by copy from the operator PC.
- The profile is read-only input. pisetup may *propose* additions (a newly discovered device) as a patch file for the operator to commit to the venue repo; it never writes to that repo itself.
- **Secrets** are references in the profile (`secret:<name>`), resolved from an environment variable or a local secret file outside any repo. Resolved values never reach logs, the state file, the web UI or discovery output.
- A **site-less mode** works with a bundled synthetic example profile (invented addresses) for tests and demos.
- Only generic examples live in this repo's docs and tests. A real address in a pisetup PR is a bug.

## 5. Phases

Each phase ends with something usable. CLI and web UI land together per phase, since the web UI only renders core results.

**Phase 0: skeleton.** Multi-venue from the start: profile loading by venue name, per-venue state directory. Repo layout, ESM package, step model, CLI shell, `serve` shell, OpenAPI, synthetic example profile, test harness with fake devices, catalog loader reading piecosystem's generated registry. No device or Pi access.

**Phase 1: read-only status.** `pisetup status` / `check`: profile load and validation, catalog, host prerequisites, app presence and versions, health endpoints, port clashes. The web UI shows the same table. Safe to run anywhere.

**Phase 2: discovery.** Probe plugins for the table in 3.1 (profile-driven first, `--scan` second), discovered-versus-expected diff, proposed profile patch. Web UI device list with status and "why not found" hints.

**Phase 3: app install and update.** Steps for checkout, build, env generation, unit install, verify, rollback. Start with two or three low-risk apps on a scratch host (not the live Pi), then the Pi. Dry-run (`plan`) is the default; `--apply` or the UI's confirm button runs it.

**Phase 4: device setup guidance.** Per-device wizards with safe applies and guided manual steps, PoE budget view, whole-stack inter-service verification, a final "ready for service" report.

**Phase 5 (placeholder): 3D model guidance.** Deferred until the calibration approach is settled; see 3.4.

Later options, not planned: fresh-SD bootstrap (one command on clean Debian that installs pisetup, restores config from `pi-backup`, replays the plan), remote fleet hosts via fffleet, a "what changed since last service" diff.

## 6. Open questions

Defaults are recommended; the plan works with them unless told otherwise.

1. **Where it runs first.** Recommended: operator PC reaching the Pi over ssh (the PC already has key ssh, the Pi needs nothing new). Alternative: on the Pi itself.
2. **Web UI hosting.** Recommended: local on the operator PC. Alternative: also served through the Pi's nginx behind the picontrol PIN.
3. **Writing device settings in phase 4.** Recommended: guide and verify by default, applies only behind a per-action confirm.


## 7. Testing and safety

- `node:test`, no real network. Tests run against at least two synthetic venues with different device sets, so single-venue assumptions fail the build.
- Fake devices only. Probes run against fake UDP/TCP/HTTP servers like the sibling repos.
- Install steps default to dry-run, covered by tests (plan output compared, nothing written).
- Everything is unverified against real devices until run on site; docs say so when a probe or applier is added.
- No tokens, keys or community strings in logs; env files written mode 600; the web UI never shows resolved secrets.
