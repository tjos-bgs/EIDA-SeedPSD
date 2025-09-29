# BGS EIDA SeedPSD — Ansible‑driven Docker deployment

This repository contains an Ansible play plus templates to deploy the **SeedPSD** stack (database, worker and web UI/API) in containers. It checks out the upstream SeedPSD sources, builds images, wires configuration, brings up Postgres first and then the web service. Optional bootstrap steps for a fresh install are provided (commented) along with sample cron jobs.

Upstream project: <https://gricad-gitlab.univ-grenoble-alpes.fr/OSUG/RESIF/seedpsd>

---

## Requirements

Basic requirements to run this script include:
1. Python 3 and Pip
2. Ansible (current version: v2.16.3) installed via Pip (`python3 -m pip install --user ansible`)
3. Docker Compose and Docker (usually done via Docker Desktop, but can use GeerlingGuy in requirements.yml to install Docker plugins for Ansible)

---

## Running the Ansible Script

To run the Ansible script, run the following command from the directory where main.yml is stored:

```yml
ansible-galaxy install -r requirements.yml
ansible-playbook -i hosts.yml main.yml -u <username> -t spsd
```

---

## Stack Summary

- **PostgreSQL 16** with one-time init SQL to create roles, database and RO defaults
- **seedpsd-worker** (RW) image for ingest/admin CLI
- **seedpsd-web** (RO) image exposing the web UI/API on localhost
- Health checks for DB and web
- Idempotent provisioning, with optional first‑run bootstrap tasks for schema + data

---

## Ansible SeedPSD Task

Primary task lives in **`tasks/seedpsd_setup.yml`**:

- Creates base directories and checks out upstream repo (`git` module)
- **Dockerfile workaround**: Removes upstream `Dockerfile` and replaces it with `files/seedpsd/Dockerfile` (checksum issue noted 17/09/2025)
- Renders `.env` files, SQL bootstrap and `docker-compose.yml`
- Brings up **Postgres first**, waits until ready, then starts **seedpsd-web**
- (Commented) **first‑run bootstrap** tasks to create schema, run migrations and seed metadata/data with one-time runs of **seedpsd-worker**

> Do not run **seedpsd-worker** with only data load, as this would load all data in the mounted directory (could take days). Always use data load with a flag such as `--network <network_name>` and/or `--year <year>`.
> As per <https://github.com/EIDA/etc/issues/88>, data has been loaded into the db for years 2024 and 2025 with one-time runs of **seedpsd-worker**

---

## Variables

Variables for SeedPSD that lives in **`vars/vars.yml`**:

| Variable | Purpose |
|---|---|
| `owner_name`, `group_name` | Filesystem ownership for created dirs |
| `seedpsd_dir` | Root deploy dir (e.g. `{{ home_dir }}/seedpsd`) |
| `seedpsd_git_dir` | Checkout dir for upstream repo (e.g. `{{ seedpsd_dir }}/seedpsd-github`) |
| `seedpsd_config_dir` | Config dir (e.g. `{{ seedpsd_dir }}/config`) |
| `seedpsd_initdb_dir` | Init SQL dir (e.g. `{{ seedpsd_dir }}/initdb`) |
| `seedpsd_log_dir` | Optional log dir |
| `seedpsd_lib_dir` | Optional app data dir |
| `seedpsd_db_dir` | Postgres host-bound data dir (e.g. `{{ seedpsd_dir }}/postgres`) |
| `seedpsd_git_repo` | Upstream git URL (see link above) |
| `seedpsd_listen_port` | Host port to bind seedpsd-web to (maps → container `8084`) |
| `seedpsd_db_superuser`, `seedpsd_db_superpass` | Postgres superuser credentials (container env) |
| `seedpsd_db_name` | Application DB name |
| `seedpsd_db_rw_user`, `seedpsd_db_rw_pass` | RW role for worker/ingest |
| `seedpsd_db_ro_user`, `seedpsd_db_ro_pass` | RO role for web UI |
| `gpfs_dir` | Host path to archive data |
| `internal_archive_sds_mnt` | Same path as seen *inside* the containers |
| `seedpsd_fdsn_client` | FDSN client identifier |
| `seedpsd_use_sds` | `true/false` per your storage layout |

---

## Running the play

On success:

- DB starts and is healthy
- Web starts and is healthy
- Web UI/API available on: `http://127.0.0.1:{{ seedpsd_listen_port }}/`

---

## First‑run bootstrap (fresh database)

These tasks are **already authored** in `tasks/seedpsd_setup.yml` and **commented out**. For a new deployment:

1. **Uncomment** the six tasks under *“The next 6 tasks are to be used on fresh setup to initialise data”*.
2. Re‑run the play. It will:
   - `seedpsd-cli admin create` — bootstrap base schema (idempotent)
   - `seedpsd-cli admin migrate --upgrade` — migrate DB to HEAD
   - Kick off **metadata** and **data** initialisation in the background (async) using your `internal_archive_sds_mnt`
   - One-time runs for **data** initialisation for the years 2024 and 2025, running parallel and with 2 max CPUs each (BGS VM has only 4 CPUs)
3. Optionally grant RO privileges across `shared` and `network_*` schemas using the provided (non‑idempotent) `psql` block. Networks for BGS are GB, BN and UR

CLI equivalents (for reference):

```bash
# from {{ seedpsd_dir }}/
docker compose run --rm seedpsd-worker seedpsd-cli admin create
docker compose run --rm seedpsd-worker seedpsd-cli admin migrate --upgrade
# Initialise metadata & data (paths must match inside container)
docker compose run --rm seedpsd-worker \
  seedpsd-cli metadata update --config-path {{ internal_archive_sds_mnt }}
docker compose run --rm seedpsd-worker \
  seedpsd-cli data load --config-path {{ internal_archive_sds_mnt }} --year <year> --parallel --max-cpu 2
```

---

## Docker images & entrypoints

This Dockerfile replaces the old Dockerfile in the checked out Git repo in the Git directory. This workaround is done because the Ansible module `community.docker.docker_compose_v2` doesn't recognise the ADD command in the old Dockerfile and the new Dockerfile replaces it with COPY. Bumped up the version of Python in the Dockerfile from v3.11-slim to v3.13-slim (v3.11-slim has 2 high vulnerabilities). Images are built from the upstream repo with a local **`files/seedpsd/Dockerfile`** that:

- Uses Python 3.13-slim and **uv** for fast locked installs
- Builds as **unprivileged `app`** user at runtime
- Exposes `/app/.venv/bin` first in `PATH`
- Defers command selection to Compose (see below)

---

## docker‑compose services (templated)

- **seedpsd-db**: `postgres:16`, health‑checked via `pg_isready`.
- **seedpsd-worker**: build from upstream repo; mounts GPFS dir to internal `/mnt/archive` with RO permissions; uses **RW** DB user; optionally mounts `lib/`.
- **seedpsd-web**: build from upstream repo; mounts GPFS dir to internal `/mnt/archive` with RO permissions; binds `127.0.0.1:{{ seedpsd_listen_port }} → 8000`; uses **RO** DB user; HTTP health check via `wget`; optionally mounts `lib/`.

---

## Database bootstrap SQL

Rendered to `initdb/00_seedpsd.sql` and executed by the official Postgres entrypoint:

- Creates **RW** and **RO** roles if missing
- Creates the **database** (owner = RW role) if missing
- Grants RO defaults on existing and future objects

> It uses `DO $$` blocks for role creation and `\gexec` for conditional `CREATE DATABASE`.

---

## Optional cron jobs

Examples (commented in tasks):

- Daily **data ingest** at 05:00 (Don't run complete data loads; add flags and run)
- Daily **metadata refresh** at 06:00

These run the worker container with the appropriate `seedpsd-cli` commands.

---

## Troubleshooting

- **Web 502/healthcheck fails**: ensure DB is healthy first; check that `.env.web` points to `seedpsd-db` and correct RO creds
- **Worker cannot connect**: verify RW creds and that `internal_archive_sds_mnt` is reachable *inside* the container (path must exist)
- **Migrations complain**: re‑run `admin create` and then `admin migrate --upgrade`
- **Permissions for RO across schemas**: use the provided `GRANT` block in tasks (non‑idempotent) to extend beyond `public`
- **Dockerfile checksum issue**: this role deliberately replaces the upstream Dockerfile; keep the local `files/seedpsd/Dockerfile` up to date
- **Postgres DB permission errors**: If DB has file permission issues on its mounted DB directory, run the following:

  ```text
  # stop the DB
  docker compose -f {{ seedpsd_dir }}/docker-compose.yml stop seedpsd-db

  # fix perms/ownership as root inside the container’s filesystem
  docker compose -f {{ seedpsd_dir }}/docker-compose.yml run --rm -u root seedpsd-db \
    bash -lc 'chown -R postgres:postgres "$PGDATA"; find "$PGDATA" -type d -exec chmod 700 {} +;'

  # start DB back up
  docker compose -f {{ seedpsd_dir }}/docker-compose.yml up -d seedpsd-db
  ```

## Example URLs

Run some of these URLs to check if SeedPSD works:

- `/eidaws/seedpsd/1/histogram?network=GB&station=WLF1&location=00&channel=HHZ&start=2024-01-01&end=2024-12-31&nodata=404`
- `/eidaws/seedpsd/1/spectrogram?network=GB&station=HEX&location=00&channel=HHZ&start=2024-01-01&end=2024-12-31&nodata=404`
- `/eidaws/seedpsd/1/value?network=GB&station=HEX&location=00&channel=HHZ&type=psd&format=csv&nodata=404&start=2024-01-01&end=2024-12-31`
