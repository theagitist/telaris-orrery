# telaris-orrery

Host-side **programmatic provisioner and queue worker** for the Telaris
self-service instance plane. Each Telaris instance is a rootless-Podman pod
(app + web + cron) on its own `telaris_<label>` database on the shared managed
PostgreSQL cluster, reached at `<label>.<wildcard-base>`. The Orrery stands one
up (or tears it down, suspends, resumes) idempotently and reversibly, and drains
the provisioning job queue the Pluriverse website fills.

Single source of truth for the design: the Academia vault note
`Projects/Telaris/Architecture/Self-service instances plan.md` and the memories
`project_telaris_self_service_phase1` / `_phase2` / `_phase3`.

## Usage

```sh
bin/telaris-orrery provision <label> --operator-email=E [opts]
bin/telaris-orrery deprovision <label> [--yes]
bin/telaris-orrery suspend <label>
bin/telaris-orrery resume <label>
bin/telaris-orrery work [--max=N]      # drain the Pluriverse provisioning_jobs queue
bin/telaris-orrery list
bin/telaris-orrery inspect <label>
```

provision opts: `--operator-first=`, `--operator-last=`, `--site-name=`,
`--site-tagline=`, `--locale=en|es|pt|fr`, `--password=<plain>` (omit for a
no-password magic-link operator), `--quota=N|infinite` (recorded only for now),
`--no-federate` (skip Pluriverse auto-trust), `--force` (re-run an existing label
idempotently), `--seed-script=PATH` (dev: `podman cp` a fresh `provision-init`
into the container instead of using the in-image copy).

## What `provision` does

1. validate the label (`[a-z0-9-]`, 2-31 chars, reserved-name list)
2. allocate a free loopback port (18080-18999)
3. create the `telaris_<label>` DB + owning role + `unaccent` on the managed cluster
4. write the per-instance DB password keyfile under `~/apps/keys/`
5. lay out the bind-mount tree under `${TELARIS_INSTANCES_ROOT}/<label>/`
6. render the `.kube` YAML (host-owned `conf/`) + the Quadlet unit, daemon-reload, start
7. seed schema + operator programmatically (`bin/provision-init` inside the container)
8. write the host nginx route + reload, record the federation keyid, write the registry
9. unless `--no-federate`: register + publish the instance in the Pluriverse
   federation registry (auto-trust), through the bridge (see below)

`deprovision` archives a `pg_dump` + a file tarball to `${TELARIS_BACKUPS_ROOT}/<label>/`
FIRST, then tears everything down (data loss is not acceptable), and retracts the
Pluriverse federation row so the instance stops being advertised.

## The Pluriverse bridge (no DB credentials in the Orrery)

The Orrery holds **no database credentials to the Pluriverse and no PII master
key**. Everything it needs to read or write in the Pluriverse-owned tables (the
federation `instances` registry and the `provisioning_jobs` queue) it does by
invoking a single committed dispatcher in the Pluriverse checkout **as www-data**:

```
sudo -n -u www-data /usr/bin/php <pluriverse>/bin/pluriverse-bridge.php <func>   < <json-args>
```

This is the entire trust surface between the two programs. www-data owns that
schema and the PII key; the Orrery only names whitelisted functions
(`register_pending`, `publish_key`, `retract`, `claim_job`, `finish_job`,
`fail_job`, `set_request_status`). The flow always runs more-privileged ->
less-privileged: a compromised Orrery can call only those functions, never run
arbitrary SQL. **This requires passwordless `sudo -u www-data` for the Orrery
user** (see Requirements).

## Requirements

- **OS user:** a dedicated non-root user that owns the rootless-Podman containers
  (here, the same user that runs the box's app stack). It must have **linger
  enabled** (`loginctl enable-linger <user>`) so its `systemctl --user` services
  and the worker timer run without an interactive login.
- **Rootless Podman 4.x + `crun`**, plus `uidmap`, `slirp4netns`, `catatonit`
  (the `--no-install-recommends podman` install omits these). Docker is not used.
- **PHP 8.3 CLI** with `ext-pdo_pgsql`, `ext-sodium` (the handoff unseal),
  `ext-json`, `ext-mbstring`.
- **PostgreSQL v18 client** (`pg_dump`) for deprovision archives; the box uses
  `/usr/lib/postgresql/18/bin/pg_dump` from the PGDG repo.
- **Passwordless sudo for two narrow things:**
  1. `sudo -u www-data /usr/bin/php <pluriverse>/bin/pluriverse-bridge.php …` —
     the bridge (every Pluriverse-DB op). The worker timer relies on this, so it
     must be NOPASSWD and must not require a tty.
  2. `systemctl reload nginx` (+ a read-only `nginx -t`) — the only OS-root op,
     granted by `/etc/sudoers.d/telaris-orrery`. Container-chowned files (uid
     100032 in the userns) are handled with `podman unshare`, never sudo.
- **The managed-PG admin creds** at `~/apps/keys/managed-polivoxia-pg1-database`
  (creates per-instance DB + role) and the cluster CA at
  `/etc/ssl/polivoxia-pg1-ca.crt`.
- **The operator-handoff key** `/etc/telaris-orrery/handoff.key` (owner
  `<orrery-user>:www-data`, mode `0640`, 32 bytes / 64 hex). The Pluriverse seals
  the operator email/name into each queued job with it; the worker unseals it.
  It is a narrow capability and is NOT the PII master key.
- **The shared origin TLS cert** for the instance tier at
  `/etc/ssl/telaris-instance-origin.{crt,key}` (any of an Origin CA cert, an LE
  wildcard, or a per-host cert works). If absent, instances are served HTTP-only.
- **The app + web OCI images** built locally (`localhost/telaris-app:dev`,
  `localhost/telaris-web:dev`) or pulled from GHCR.

## Host bootstrap (one-time)

```sh
# 1. linger + the bridge sudoers (NOPASSWD, no-tty) for the Orrery user:
sudo loginctl enable-linger <orrery-user>
printf '%s\n' \
  '<orrery-user> ALL=(www-data) NOPASSWD: /usr/bin/php /var/www/www.telaris.ca/bin/pluriverse-bridge.php *' \
  | sudo tee /etc/sudoers.d/telaris-orrery-bridge
sudo chmod 0440 /etc/sudoers.d/telaris-orrery-bridge

# 2. the nginx route dir, included from a root-owned conf.d shim, + the
#    nginx-reload sudoers line:
sudo install -d -o <orrery-user> -g <orrery-user> /etc/nginx/telaris-orrery
echo 'include /etc/nginx/telaris-orrery/*.conf;' \
  | sudo tee /etc/nginx/conf.d/00-telaris-orrery.conf
#    /etc/sudoers.d/telaris-orrery: <orrery-user> ALL=(root) NOPASSWD: /usr/sbin/nginx -s reload, /usr/sbin/nginx -t

# 3. the handoff key (shared with the Pluriverse app):
sudo install -d -o root -g root /etc/telaris-orrery
openssl rand -hex 32 | sudo tee /etc/telaris-orrery/handoff.key >/dev/null
sudo chown <orrery-user>:www-data /etc/telaris-orrery/handoff.key
sudo chmod 0640 /etc/telaris-orrery/handoff.key

# 4. install the worker timer (drains the queue every minute):
cp units/telaris-orrery-worker.{service,timer} ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now telaris-orrery-worker.timer
```

> `systemctl --user` from a non-login shell needs `XDG_RUNTIME_DIR=/run/user/$(id -u)`
> and `DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u)/bus` exported.

The blanket box-wide passwordless sudo present on this host satisfies (1); the
explicit `sudoers.d` drop-in above is what you would ship to a box that does not
already grant the Orrery user passwordless sudo.

## Configuration

Everything is env-overridable; the defaults match this box. Key knobs:
`TELARIS_INSTANCES_ROOT`, `TELARIS_BACKUPS_ROOT`, `TELARIS_IMAGE_APP/WEB`,
`TELARIS_WILDCARD_BASE`, `TELARIS_PORT_MIN/MAX`, `TELARIS_PG_ADMIN_CREDS`,
`TELARIS_DB_SSL_CA`, `TELARIS_INSTANCE_TLS_CERT/_KEY`, `TELARIS_PLURIVERSE_APP_DIR`
(the Pluriverse checkout the bridge lives in), `TELARIS_ORRERY_HANDOFF_KEY`,
`TELARIS_ORRERY_WORKER_MAX_JOBS/ATTEMPTS`.

## Design notes

- Every external process runs via an **argument array** (`proc_open`, no shell), and
  every DB existence check is **parameterized SQL**: the only values that reach DDL
  are a charset-validated identifier and a hex password.
- The registry is a local JSON (`registry.json`, mode 600) used for the operator
  CLI (`list`/`inspect`); the authoritative federation record is the Pluriverse
  `instances` table, written only through the bridge.
- The worker (`work`) claims jobs `FOR UPDATE SKIP LOCKED` (via the bridge), runs
  the fixed provisioner sequence in-process, and records success/backoff/give-up.

## License

GNU AGPL-3.0-or-later (`LICENSE`).
