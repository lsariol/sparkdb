# sparkdb Documentation

The reference for **sparkdb**, the Postgres server behind every project on the `spark` server: how it's set up, how projects get a database, how it's deployed and backed up, and how to restore it.

- [README](README.md): a short overview.
- [Lighthouse](https://github.com/lsariol/lighthouse): deploys sparkdb. Its `DOCUMENTATION.md` §7.2 (`x-lighthouse`, backups) and §11 ("If sparkdb or Cove itself is gone") are the other half of this document.
- [Cove](https://github.com/LSariol/Cove): holds every database connection string.

---

## Contents

1. [Overview](#1-overview)
2. [Quick reference](#2-quick-reference)
3. [Configuration](#3-configuration)
4. [Databases and roles](#4-databases-and-roles)
5. [Deploying](#5-deploying)
6. [Backups and restores](#6-backups-and-restores)
7. [Upgrading Postgres](#7-upgrading-postgres)
8. [Development](#8-development)
9. [Troubleshooting](#9-troubleshooting)
10. [Known limitations](#10-known-limitations)

---

## 1. Overview

| Item | Value |
|---|---|
| Image | `postgres:16.4`, pinned |
| Container | `sparkdb`, compose project `sparkdb` |
| Reached at | `sparkdb:5432` from containers on `spark`; `127.0.0.1:5000` from the server itself |
| Data | `/srv/server/storage/sparkdb/data` on the server, mounted at `/var/lib/postgresql/data` |
| Admin user | `Admin`, default database `AdminDB` |
| Health | `pg_isready`, every 5 s |
| Deployed by | Lighthouse, on version tags: `deploy: releases`, `tier: data`, `backup: postgres` |

sparkdb runs nothing but Postgres. Every project that needs a database gets its own database and roles on it (§4), and creates its own tables with its own migrations.

---

## 2. Quick reference

On the server, over SSH.

| Task | Command |
|---|---|
| Is it up? | `docker exec sparkdb pg_isready` |
| A SQL prompt as the admin | `docker exec -it sparkdb psql -U Admin -d AdminDB` |
| List databases | `docker exec sparkdb psql -U Admin -d AdminDB -c '\l'` |
| List roles | `docker exec sparkdb psql -U Admin -d AdminDB -c '\du'` |
| Back up everything now | `sudo mkdir -p /srv/backups/manual && docker exec sparkdb pg_dumpall -U Admin \| gzip \| sudo tee /srv/backups/manual/sparkdb-$(date +%F).sql.gz > /dev/null` |
| Its log | `docker logs sparkdb` |
| Deploy a new version | Tag a release (`v1.0.1`) and push the tag; Lighthouse does the rest |

From the PC, pgAdmin connects through an SSH tunnel to the server, to host `127.0.0.1`, port `5000`.

---

## 3. Configuration

Everything is in `docker-compose.yaml`.

| Setting | Value | Notes |
|---|---|---|
| `POSTGRES_USER` | `Admin` | The superuser. Only used when the data folder is first created |
| `POSTGRES_DB` | `AdminDB` | Created with the superuser. Only used at first creation |
| `POSTGRES_PASSWORD` | `${SPARKDB_DATABASE_PASSWORD}` | From Cove. Only used at first creation; after that the password stored in the database is the real one |
| Port | `127.0.0.1:5000:5432` | The server only. Containers use `sparkdb:5432` on `spark` |
| Network | `spark` (external) | Shared with Cove, Lighthouse and the projects |

**The three `POSTGRES_` settings only apply to an empty data folder.** Changing them later changes nothing in the running database: change the admin password with `ALTER ROLE "Admin" PASSWORD '...'`, then update `SPARKDB_DATABASE_PASSWORD` in Cove to match.

---

## 4. Databases and roles

Each project that needs a database gets four roles and one database. Creating them is admin work, done once by hand; everything inside the database comes from the project's own goose migrations.

| Role | Logs in | Used for |
|---|---|---|
| `<p>_owner` | No | Owns the database and its schema |
| `<p>_migrator` | Yes | Runs migrations; member of `<p>_owner`, with `SET role = '<p>_owner'`, so what it creates belongs to the owner |
| `<p>_app` | Yes | The running project |
| `<p>_reader` | Yes | Read-only access, e.g. pgAdmin |

The database is owned by `<p>_owner`; `PUBLIC` may not connect, and the three login roles may.

**Passwords live only in Cove**, inside one connection string per login role:

| Role | Cove key |
|---|---|
| `<p>_app` | `<P>_DATABASE_URL` |
| `<p>_migrator` | `<P>_DATABASE_MIGRATOR_URL` |
| `<p>_reader` | `<P>_DATABASE_READER_URL` |

Each is `postgres://<role>:<password>@sparkdb:5432/<p>_db?sslmode=disable`. Use only letters and digits in passwords: `@ : / ? #` break a URL.

**A worked example** of the roles, the database and their grants, as a script that is safe to run again, is Lighthouse's `scripts/db/setup.sql` (Lighthouse `DOCUMENTATION.md` §10.1).

**Renaming a project's database or roles** is instant (`ALTER DATABASE ... RENAME`, `ALTER ROLE ... RENAME`) but has two traps: a database can't be renamed from a session connected to it, and a migrator's `role=` setting is stored as text, so it must be set again afterwards.

---

## 5. Deploying

Lighthouse deploys sparkdb when a version tag is pushed (`deploy: releases`). As `tier: data` it goes before everything else, alone:

1. Lighthouse dumps every database (`backup: postgres`, §6).
2. It fetches `SPARKDB_DATABASE_PASSWORD` from Cove, before Cove's database goes away.
3. It swaps the container. Every project loses its database connection for a few seconds and reconnects.
4. It waits for the healthcheck, and puts the previous version back if sparkdb doesn't come up.

A rollback can't undo a change to the data itself, which is why the image is pinned (§7).

**Releasing:** update `CHANGELOG.md`, tag `vX.Y.Z`, push the tag, then push `main`.

**If the container was removed**, Lighthouse can't bring it back (it needs Cove, which needs sparkdb). Start it by hand from its newest deploy folder:

```bash
cd "$(ls -dt /srv/server/staging/sparkdb/*/ | head -1)" && docker compose -p sparkdb up -d
docker exec sparkdb pg_isready
```

Its data is kept, so no password is needed.

---

## 6. Backups and restores

| Backup | Status |
|---|---|
| Before every sparkdb deploy | Automatic: `pg_dumpall`, gzipped, to `/srv/backups/sparkdb/` (mode 0600, newest 5 kept). If it fails, the deploy stops and nothing changes |
| Scheduled (nightly) | Not set up |
| A copy off the server | Not set up |

A `pg_dumpall` file holds every database and every role, with their password hashes: keep it private.

**Restoring.** A `pg_dumpall` file is a whole-server restore: every role and every database, replayed with `psql`. It belongs on an **empty** server (a new data folder), since replaying it over existing databases fails on what already exists:

```bash
gunzip -c /srv/backups/sparkdb/<file>.sql.gz | docker exec -i sparkdb psql -U Admin -d AdminDB
```

To restore **one** database, a per-database dump is easier: `pg_dump -Fc -d <db>` makes one, `pg_restore -d <db>` restores it into an empty database owned by the project's owner role. Pulling one database out of a `pg_dumpall` file means cutting its section out of the text by hand.

Keep manual backups out of `/srv/backups/sparkdb/`: Lighthouse keeps only the newest 5 files there.

**Cove's database needs its key.** A `cove_db` dump is useless without the vault key it was encrypted with (in the password manager). Restore them together; Cove `DOCUMENTATION.md` §14.

**Practice a restore** every few months on the PC, against a throwaway Postgres container, never against the server.

---

## 7. Upgrading Postgres

- **A patch release** (16.4 → 16.5): change the image tag, release. The data format doesn't change.
- **A major release** (16 → 17): the data folder's format changes, so a new image can't open the old data and a rollback can't undo it. Plan it by hand: dump everything, start the new version on a new, empty data folder, restore, check every project, then release the compose change. Keep the old data folder until everything has run on the new one for a while.

---

## 8. Development

`dev/docker-compose.yaml` (not in git) runs a local copy on the PC as `sparkdb-dev`, with a named volume instead of the server's folder, on `127.0.0.1:5000`. Projects' dev setups connect to `localhost:5000`.

---

## 9. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Projects can't connect after a deploy | Postgres is still starting; Lighthouse waits for the healthcheck. `docker logs sparkdb` |
| `password authentication failed for user "<role>"` | The URL in Cove doesn't match the role's password. Set the role's password from the URL, or the URL from the role |
| `database "<x>" does not exist` | The project's database wasn't created, or was renamed. `\l` lists them |
| A deploy fails at `backup` | `pg_dumpall` failed or `/srv/backups` isn't writable. Nothing changed |
| The container exits at start with a data-format error | A major-version image on old data (§7). Go back to the previous image |
| pgAdmin can't connect | The SSH tunnel isn't set, or the connection uses the server's address instead of `127.0.0.1:5000` |

---

## 10. Known limitations

1. **No scheduled or off-server backups** yet; only the dump before each deploy.
2. **One server for every project:** a sparkdb restart interrupts every project at once.
3. **A rollback can't undo a data-format change** (§7).
4. **The `POSTGRES_` settings only apply to an empty data folder** (§3).
