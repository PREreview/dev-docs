# PREreview v2 developer documentation

- [PREreview v2 developer documentation](#prereview-v2-developer-documentation)
  - [Service providers](#service-providers)
  - [🖥️ Servers 🖥️](#%f0%9f%96%a5%ef%b8%8f-servers-%f0%9f%96%a5%ef%b8%8f)
    - [PREreview v2 platform](#prereview-v2-platform)
      - [Details](#details)
      - [⬆️🖥️ Start the server](#%e2%ac%86%ef%b8%8f%f0%9f%96%a5%ef%b8%8f-start-the-server)
      - [↩️🖥️ Restart the server](#%e2%86%a9%ef%b8%8f%f0%9f%96%a5%ef%b8%8f-restart-the-server)
      - [⬇️🖥️ Stop the server](#%e2%ac%87%ef%b8%8f%f0%9f%96%a5%ef%b8%8f-stop-the-server)
      - [Renew the TLS certificate](#renew-the-tls-certificate)
    - [CORS proxy server](#cors-proxy-server)
    - [Authentication / security](#authentication--security)
  - [Database](#database)
    - [Overview](#overview)
    - [Connecting to the live database](#connecting-to-the-live-database)
    - [DB: Init](#db-init)
    - [DB: Seed](#db-seed)
    - [DB: Migrations](#db-migrations)
  - [User auth - ORCID integration](#user-auth---orcid-integration)
  - [getpreprints](#getpreprints)

## Service providers

- Digitalocean for servers and database
- ORCID for user authentication
- Mailgun for mail
- Lastpass for secrets (e.g. API client IDs and keys)

## 🖥️ Servers 🖥️

### PREreview v2 platform

#### Details

- Address: https://v2.prereview.org
- Digitalocean droplet: `prereview-live`
- Code: https://github.com/PREreview/prereview-standup

#### ⬆️🖥️ Start the server

```bash
cd ~/prereview-standup
npm run start
```

#### ↩️🖥️ Restart the server

```bash
pm2 restart prereview
```

#### ⬇️🖥️ Stop the server

```bash
cd ~/prereview-standup
npm run stop
```

#### Renew the TLS certificate

> 📝 Note: This is run automatically using a cron job. You shoud not need to run it manually under normal circumstances, but it's documented here in case it's needed.

This only performs the hooks and renewal action if the certificate is close to expiry, so it can be run any time (e.g. during the daily update window).

```bash
certbot renew --pre-hook "pm2 stop prereview" --post-hook "pm2 start prereview"
```

### CORS proxy server

- Address: https://preprint-proxy.prereview.org:8080
- Digitalocean droplet: `prereview-preprint-cors-proxy`
- Code: https://github.com/PREreview/preprint-cors-proxy

### Authentication / security

The servers must be accessed by SSH, and using SSH key auth only. To access the servers you must have your SSH public key added to the SSH authorized keys file `~/.ssh/authorized_keys`.

## Database

### Overview

PostgreSQL with Knex.js as the interface.

Production DB runs on Digitaloean:

- Digitalocean droplet: `prereview-postgres-db-alpha`
- PostgreSQL version: `11`
- Database name: `prereview-beta` (to be migrated after beta) and `prereview`
- Account: `prereview-dev`

The database is available as a direct connection (on port `25060`) or a [pool of 22 nodes](https://cloud.digitalocean.com/databases/prereview-postgres-db-alpha/pools?i=cd22e4) (on port `25061`). Generally we use the pool for the live server and the dedicated connection for bulk updates of preprint metadata.

### Connecting to the live database

To work with the live database you must:

- force SSL is on in your connection settings
- have installed the CA certificate (available from [the overview tab](https://cloud.digitalocean.com/databases/prereview-postgres-db-alpha) for the database on Digitalocean)
- add your IP address to the 'trusted sources' list on the Digitaloccean [db overview page](https://cloud.digitalocean.com/databases/prereview-postgres-db-alpha)

> ⚠️ Warning: You should not connect to the live DB manually - use the scripts ⚠️

### DB: Init

The [init script](https://github.com/PREreview/prereview-standup/blob/master/scripts/initdb.js) (re-)creates [the database tables](https://github.com/PREreview/prereview-standup/tree/master/server/db/tables) from the schemas in `server/db/tables/*/schema.js`.

> ⚠️ Warning: Running the init script drops all the tables if they already exist ⚠️

The scripts are run with `npm run initdb:dev` or `npm run initdb:prod`.

### DB: Seed

Populates the preprint table with data from getpreprints.

Seeding the database is a two-step process:

1. Run getpreprints to create a local hyperdb (`~/.getpreprints/data/database`) containing all the aggregated and cleaned metadata
2. Run `npm run initdb:dev` or `npm run initdeb:prod` to stream the data from the local hyperdb to the PREreview database

### DB: Migrations

If you need to change the database structure, use Knex migrations.

The process is:

1. Generate a migration file with `./node_modules/.bin/knex migrate:make [migration_name]`
2. Populate the migration file, which is now in `./migrations/[migration_name]` code
3. Run the migration with `./node_modules/.bin/knex:up`

You can roll back the last migration with `./node_modules/.bin/knex:down`

> 📝 Note: It's important to keep the migrations directory intact as it maps to the migrations table in the database. The migrations directory is `.gitignore`d, to prevent setup and exploratory migrations from creeping into the codebase. This means that if you create production migrations you **must** manually commit and push them. 

## User auth - ORCID integration

User authentication is done using ORCID.

The [server components](https://github.com/PREreview/prereview-standup/tree/master/server/auth/orcid) require two environment variables:

- `ORCID_CLIENT_ID`
- `ORCID_CLIENT_SECRET`

## getpreprints

The data is seeded using getpreprints, which extracts from archival dumps of Crossref, EuropePMC and ArXiv as well as syncing the most recent entries from these three providers.

To populate a db from dumps, you must download the dumps using getpreprints, then run one or more of the import scripts in `getpreprints/scripts/import` to populate the local hyperdb in `~/.getpreprints/data/database`. Once this is done, you can run the [seed scripts](#seed).
