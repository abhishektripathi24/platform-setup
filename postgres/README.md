# PostgreSQL [Master-Slave Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/postgres/images/postgresql-logo.png" width="200" height="200"/>

To know more about postgresql, visit https://www.postgresql.org/

## Overview

From the official docs -

> PostgreSQL is a powerful, open source object-relational database system with over 30 years of active development that has earned it a strong reputation for reliability, feature robustness, and performance.

## Setup
For master-slave setup, refer following [timescale's documentation](../timescale/README.md).

## Monitoring

1. Basic monitoring - [Telegraf](https://docs.influxdata.com/telegraf/v1.13/)
    * Installation -
        * https://docs.influxdata.com/telegraf/v1.13/introduction/installation/
    * Configuration -
        * https://github.com/influxdata/telegraf/tree/master/plugins/inputs/postgresql
        * https://github.com/influxdata/telegraf/tree/master/plugins/inputs/postgresql_extensible
        * https://www.postgresql.org/docs/10/monitoring-stats.html
    * Sample configuration can be found [here](monitoring/telegraf.conf).

2. Performance monitoring - [PgHero](https://github.com/ankane/pghero) 
    * Linux setup guide: https://github.com/ankane/pghero/blob/master/guides/Linux.md
    * A sample pg_hero.yml is available [here](monitoring/sample_pg_hero.yml).

3. Log monitoring - [PgBadger](https://github.com/darold/pgbadger)
    * Installation - 
        ```bash
        sudo apt-get install pgbadger --fix-missing
        which pgbadger
        pgbadger /var/log/postgresql/postgresql-10-main.log
        ```
    * Systemd units for python webserver, pgbadger service and timer are present [here](systemd). 

4. Connection Pooling - [PgBouncer](https://github.com/pgbouncer/pgbouncer)
    * Linux setup guide: https://tech.willandskill.se/how-to-setup-pgbouncer-on-ubuntu-18-04-lts/