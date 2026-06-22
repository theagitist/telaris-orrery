# telaris-orrery

Host-side **programmatic provisioner** for the Telaris self-service instance plane
(self-service-instances plan, Phase 2). Run as the Orrery user (the rootless-Podman
owner). It automates, idempotently and reversibly, everything that was done by hand
to stand up the Phase-1 `fleet-test` pod.

Single source of truth for the design: the Academia vault note
`Projects/Telaris/Architecture/Self-service instances plan.md` ("Phase 2 progress")
and memory `project_telaris_self_service_phase2`.

## Usage

```sh
bin/telaris-orrery provision <label> --operator-email=E [opts]
bin/telaris-orrery deprovision <label> [--yes]
bin/telaris-orrery suspend <label>
bin/telaris-orrery resume <label>
bin/telaris-orrery list
bin/telaris-orrery inspect <label>
```

provision opts: `--operator-first=`, `--operator-last=`, `--site-name=`,
`--site-tagline=`, `--locale=en|es|pt|fr`, `--password=<plain>` (omit for a
no-password magic-link operator), `--quota=N|infinite` (recorded only for now),
`--no-federate`, `--force` (re-run an existing label idempotently),
`--seed-script=PATH` (dev: `podman cp` a fresh `provision-init` into the container
instead of using the in-image copy).

## What `provision` does

1. validate the label (`[a-z0-9-]`, 2-31 chars, reserved-name list)
2. allocate a free loopback port (18080-18999)
3. create the `telaris_<label>` DB + owning role + `unaccent` on managed `polivoxia-pg1`
4. write the per-instance DB password keyfile under `~/apps/keys/`
5. lay out the bind-mount tree under `${TELARIS_INSTANCES_ROOT}/<label>/`
6. render the `.kube` YAML (host-owned `conf/`) + the Quadlet unit, daemon-reload, start
7. seed schema + operator programmatically (`bin/provision-init` inside the container)
8. write the host nginx route + reload, record the federation keyid, write the registry

`deprovision` archives a `pg_dump` + a file tarball to `${TELARIS_BACKUPS_ROOT}/<label>/`
FIRST, then tears everything down (data loss is not acceptable).

## Design notes

- Every external process runs via an **argument array** (`proc_open`, no shell), and
  every DB existence check is **parameterized SQL**: the only values that reach DDL
  are a charset-validated identifier and a hex password. (Implements the plan's Q5
  trust boundary.)
- The only OS-root operation is `systemctl reload nginx` (+ a read-only `nginx -t`),
  via `/etc/sudoers.d/telaris-orrery`. Container-chowned files (uid 100032 in the
  userns) are handled with `podman unshare`, not sudo.
- Registry is a local JSON (`registry.json`, mode 600). The Pluriverse `instances`
  table supersedes it at Phase 3/4.
- All paths/values are env-overridable (`TELARIS_INSTANCES_ROOT`, `TELARIS_BACKUPS_ROOT`,
  `TELARIS_IMAGE_APP/WEB`, `TELARIS_WILDCARD_BASE`, `TELARIS_PORT_MIN/MAX`,
  `TELARIS_PG_ADMIN_CREDS`, `TELARIS_DB_SSL_CA`, ...).

## Host prerequisites (one-time, already done on this box)

- Rootless Podman + `crun`, `uidmap`/`slirp4netns`/`catatonit`, linger on (Phase 1).
- Orrery-owned `/etc/nginx/telaris-orrery/` included via root-owned
  `/etc/nginx/conf.d/00-telaris-orrery.conf`.
- The narrow sudoers line `/etc/sudoers.d/telaris-orrery`.
- The app + web images built (`localhost/telaris-app:dev`, `localhost/telaris-web:dev`).
- The managed-PG admin creds at `~/apps/keys/managed-polivoxia-pg1-database` and the
  CA at `/etc/ssl/polivoxia-pg1-ca.crt`.
