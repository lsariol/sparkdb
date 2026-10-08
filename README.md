# sparkdb

**The one Postgres server for every project on the `spark` server.**

sparkdb is a plain, pinned Postgres 16.4 container. Each project gets its own database and roles on it; nothing else runs here. Lighthouse deploys it as data infrastructure: first, alone, and only after a backup.

- **One server, many databases:** every project's data, each in its own database with its own roles.
- **Private:** reachable from containers on the `spark` network as `sparkdb:5432`, and from the server itself on `127.0.0.1:5000`. Nothing else on the network can connect.
- **Backed up before every deploy:** Lighthouse dumps every database before it changes sparkdb.
- **Its data outlives the container:** it lives in `/srv/server/storage/sparkdb/data` on the server.

> **Documentation:** [DOCUMENTATION.md](DOCUMENTATION.md) covers configuration, databases and roles, deploying, backups and restores, and troubleshooting. Changes per release: [CHANGELOG.md](CHANGELOG.md).

---

## How projects use it

A project connects as its own login role, with a connection string from Cove:

```yaml
environment:
  - DATABASE_URL=${MYAPP_DATABASE_URL}     # postgres://myapp_app:...@sparkdb:5432/myapp_db
```

Its roles and database are created once by hand; its tables come from its own migrations. [DOCUMENTATION.md §4](DOCUMENTATION.md#4-databases-and-roles).

---

## Quick start

Lighthouse deploys sparkdb when a version is tagged. By hand, on the server:

```bash
sudo mkdir -p /srv/server/storage/sparkdb/data
docker network inspect spark >/dev/null 2>&1 || docker network create spark
SPARKDB_DATABASE_PASSWORD=<admin password> docker compose up -d
docker exec sparkdb pg_isready          # "accepting connections"
```

The password only matters the first time, when the data folder is empty: Postgres stores it then and ignores the setting afterwards.

To connect from the PC with pgAdmin: an SSH tunnel to the server, then `127.0.0.1:5000`.
