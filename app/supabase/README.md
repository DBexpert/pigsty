# Supabase

[Supabase](https://supabase.com/), The open-source Firebase alternative based on PostgreSQL.

Pigsty allow you to self-host **supabase** with existing managed HA postgres cluster, and launch the stateless part of supabase with docker-compose.

> Notice: Supabase is [GA](https://supabase.com/ga) since 2024.04.15

> Setup Tutorial: https://pigsty.io/docs/kernel/supabase

-----------------------

## Quick Start

To run supabase with existing postgres instance, prepare the [database](#database) with [`supabase.yml`](https://github.com/Vonng/pigsty/blob/master/conf/sample/supabase.yml)

then launch the [stateless part](#stateless-part) with the [`docker-compose`](docker-compose.yml) file:

```bash
cd app/supabase; make up    # https://supabase.com/docs/guides/self-hosting/docker
```

Then you can access the supabase studio dashboard via `http://<admin_ip>:8000` by default, the default dashboard username is `supabase` and password is `pigsty`.

You can also configure the `infra_portal` to expose the WebUI to the public through Nginx and SSL.



-----------------------

## Database

Supabase require certain PostgreSQL extensions, schemas, and roles to work, which can be pre-configured by Pigsty: [`supabase.yml`](https://github.com/Vonng/pigsty/blob/master/conf/sample/supabase.yml).

The following example will configure the default `pg-meta` cluster as underlying postgres for supabase:

```yaml
# supabase example cluster: pg-meta, this cluster needs to be migrated with ~/pigsty/app/supabase/migration.sql :
pg-meta:
  hosts: { 10.10.10.10: { pg_seq: 1, pg_role: primary } }
  vars:
    pg_cluster: pg-meta
    pg_version: 15
    pg_users:
      # supabase roles: anon, authenticated, dashboard_user
      - { name: anon           ,login: false }
      - { name: authenticated  ,login: false }
      - { name: dashboard_user ,login: false ,replication: true ,createdb: true ,createrole: true }
      - { name: service_role   ,login: false ,bypassrls: true }
      # supabase users: please use the same password
      - { name: supabase_admin             ,password: 'DBUser.Supa' ,pgbouncer: true ,inherit: true   ,superuser: true ,replication: true ,createdb: true ,createrole: true ,bypassrls: true }
      - { name: authenticator              ,password: 'DBUser.Supa' ,pgbouncer: true ,inherit: false  ,roles: [ authenticated ,anon ,service_role ] }
      - { name: supabase_auth_admin        ,password: 'DBUser.Supa' ,pgbouncer: true ,inherit: false  ,createrole: true }
      - { name: supabase_storage_admin     ,password: 'DBUser.Supa' ,pgbouncer: true ,inherit: false  ,createrole: true ,roles: [ authenticated ,anon ,service_role ] }
      - { name: supabase_functions_admin   ,password: 'DBUser.Supa' ,pgbouncer: true ,inherit: false  ,createrole: true }
      - { name: supabase_replication_admin ,password: 'DBUser.Supa' ,replication: true }
      - { name: supabase_read_only_user    ,password: 'DBUser.Supa' ,bypassrls: true ,roles: [ pg_read_all_data ] }
    pg_databases:
      - { name: meta ,baseline: cmdb.sql ,comment: pigsty meta database ,schemas: [ pigsty ]} # the pigsty cmdb, optional
      - name: supa
        baseline: supa.sql    # the init-scripts: https://github.com/supabase/postgres/tree/develop/migrations/db/init-scripts
        owner: supabase_admin
        comment: supabase postgres database
        schemas: [ extensions ,auth ,realtime ,storage ,graphql_public ,supabase_functions ,_analytics ,_realtime ]
        extensions:
          - { name: pgcrypto  ,schema: extensions  } # 1.3   : cryptographic functions
          - { name: pg_net    ,schema: extensions  } # 0.9.2 : send async HTTP requests
          - { name: pgjwt     ,schema: extensions  } # 0.2.0 : JSON Web Token API for Postgresql
          - { name: uuid-ossp ,schema: extensions  } # 1.1   : generate universally unique identifiers (UUIDs)
          - { name: pgsodium        }                # 3.1.8 : pgsodium is a modern cryptography library for Postgres.
          - { name: supabase_vault  }                # 0.2.8 : supabase vault extension
          - { name: pg_graphql      }                # 1.3.0 : pg_graphql: GraphQL support
          - { name: index_advisor   }                # 0.2.0 : index_advisor Query Index Advisor
    pg_hba_rules:
      - { user: all ,db: supa ,addr: intra       ,auth: pwd ,title: 'allow supa database access from intranet'}
      - { user: all ,db: supa ,addr: 172.0.0.0/8 ,auth: pwd ,title: 'allow supa database access from docker network'}
    pg_extensions: [ 'supa-stack' ]
    # - pg_repack wal2json pgvector pg_cron pg_sodium pg_graphql pg_jsonschema wrappers vault pgjwt pg_net pgsql_http supautils index_advisor
    pg_libs: 'pg_net, pg_cron, pg_stat_statements, auto_explain'    # add pg_net to shared_preload_libraries
```

To serve a self-hosting supabase, you have to perform schema migration on the bootstrapped cluster `pg-meta`, which is achieved by the  [`files/supa.sql`](https://github.com/Vonng/pigsty/blob/master/files/supa.sql) baseline in `pg_databases[supa]`,
and a migration script [`migration.sql`](migration.sql),  which is gathered from [supabase/postgres/migrations/db/migrations](https://github.com/supabase/postgres/tree/develop/migrations/db/migrations) in chronological order and slightly modified to fit Pigsty.

The latest migration file is synced with [20240606060239](https://github.com/supabase/postgres/blob/develop/migrations/db/migrations/20240606060239_grant_predefined_roles_to_postgres.sql), to run it on the provisioned postgres cluster `pg-meta`:

```bash
# adjust the connection string if necessary
PGURL=postgres://supabase_admin:DBUser.Supa@10.10.10.10:5432/supa
psql ${PGURL} -v ON_ERROR_STOP=1 --no-psqlrc -f ~/pigsty/app/supabase/migration.sql
```

The database is now ready for supabase!

![](https://pigsty.io/img/pigsty/supa.jpg)


-----------------------

## Stateless Part

Supabase stateless part is managed by `docker-compose`, the [`docker-compose`](docker-compose.yml) file we use here is a simplified version of [github.com/supabase/docker/docker-compose.yml](https://github.com/supabase/supabase/blob/master/docker/docker-compose.yml).

Everything you need to care about is in the [`.env`](.env) file, which contains important settings for supabase. It is already configured to use the `pg-meta`.`supa` database by default, You have to change that according to your actual deployment. 

```bash
############
# Secrets - YOU MUST CHANGE THESE BEFORE GOING INTO PRODUCTION
############
# you have to change the JWT_SECRET to a random string with at least 32 characters long
# and issue new ANON_KEY/SERVICE_ROLE_KEY JWT with that new secret, check the tutorial:
# https://supabase.com/docs/guides/self-hosting/docker#securing-your-services
JWT_SECRET=your-super-secret-jwt-token-with-at-least-32-characters-long
ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyAgCiAgICAicm9sZSI6ICJhbm9uIiwKICAgICJpc3MiOiAic3VwYWJhc2UtZGVtbyIsCiAgICAiaWF0IjogMTY0MTc2OTIwMCwKICAgICJleHAiOiAxNzk5NTM1NjAwCn0.dc_X5iR_VP_qT0zsiyj_I_OZ2T9FtRU2BBNWN8Bu4GE
SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyAgCiAgICAicm9sZSI6ICJzZXJ2aWNlX3JvbGUiLAogICAgImlzcyI6ICJzdXBhYmFzZS1kZW1vIiwKICAgICJpYXQiOiAxNjQxNzY5MjAwLAogICAgImV4cCI6IDE3OTk1MzU2MDAKfQ.DaYlNEoUrrEn2Ig7tqibS-PHK5vgusbcbo7X36XVt4Q

############
# Dashboard - Credentials for the Supabase Studio WebUI
############
DASHBOARD_USERNAME=supabase         # change to your own username
DASHBOARD_PASSWORD=pigsty           # change to your own password

############
# Database - You can change these to any PostgreSQL database that has logical replication enabled.
############
POSTGRES_HOST=10.10.10.10           # change to Pigsty managed PostgreSQL cluster/instance VIP/IP/Hostname
POSTGRES_PORT=5432                  # you can use other service port such as 5433, 5436, 6432, etc...
POSTGRES_DB=supa                    # change to supabase database name, `supa` by default in pigsty
POSTGRES_PASSWORD=DBUser.Supa       # supabase dbsu password (shared by multiple supabase biz users)
```

Usually you'll have to change these parameters accordingly. Here we'll use fixed username, password and IP:Port database connect string for simplicity.

The postgres username is fixed as `supabase_admin` and the password is `DBUser.Supa`, change that according to your [`supabase.yml`](https://github.com/Vonng/pigsty/blob/master/conf/sample/supabase.yml#L43)
And the supabase studio WebUI credential is managed by `DASHBOARD_USERNAME` and `DASHBOARD_PASSWORD`, which is `supabase` and `pigsty` by default.

The official tutorial: [Self-Hosting with Docker](https://supabase.com/docs/guides/self-hosting/docker) just have all the details you need.

> ### Hint
>
> You can use the [Primary Service](https://github.com/Vonng/pigsty/blob/master/docs/PGSQL-SVC.md#primary-service) of that cluster through DNS/VIP and other service ports, or whatever access method you like.
>
> You can also configure `supabase.storage` service to use the MinIO service managed by pigsty, too

Once configured, you can launch the stateless part with `docker-compose` or `make up` shortcut:

```bash
cd ~/pigsty/app/supabase; make up    #  = docker compose up
```



-----------------------

## Expose Service

The supabase studio dashboard is exposed on port `8000` by default, you can add this service to the `infra_portal` to expose it to the public through Nginx and SSL. 

```yaml
    infra_portal:                     # domain names and upstream servers
      # ...
      supa         : { domain: supa.pigsty ,endpoint: "10.10.10.10:8000", websocket: true }
```

To expose the service, you can run the `infra.yml` playbook with the `nginx` tag:

```bash
./infra.yml -t nginx
```

Make suare `supa.pigsty` or your own domain is resolvable to the `infra_portal` server, and you can access the supabase studio dashboard via `https://supa.pigsty`.
