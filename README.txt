# awx-backup_and_restore
Backup and restore database of AWX
----
this is a draft how-to-guide, I am going to separate and upload the scripts.
----


AWX – Backup and restore plan
Read in:
English

The goal:
Backup and restore the awx: We wil focus only on the database of awx. This is the most important in our case.
Please keep in your mind: you have to ensure if you are able to restore/redeploy the awx-deployment (like containers via docker compose file),too.

BACKUP:
Create the necessary paths:
/srv/awx/backup : the “backup of environment” will be placed here,
/srv/awx/backup/sql: the backup of database will be stored here

Create the necessary scripts:
notice: feel free to edit based on you espectations (for example: add or remove the desired directory). We will keep the backup(s) 15 days.
vi /usr/local/bin/awx-envbackup.sh :
#!/bin/bash
export today=`date +%Y%m%d`

##delete old backups
export deleteday=`/bin/date -d ‘-15 days’ +%Y%m%d`
rm -f /srv/awx/backup/awx-env-$deleteday.tar.gz

##create a new backup
tar -czvpf /srv/awx/backup/awx-env-$today.tar.gz –exclude=/srv/awx/backup –exclude=/srv/awx/uploads –exclude=/srv/awx/pgdocker –exclude /srv/awx/monitoring /srv/awx

vi /usr/local/bin/awx-pgsqlbackup.sh :
#!/bin/bash
export today=`date +%Y%m%d`

export deleteday=`/bin/date -d ‘-15 days’ +%Y%m%d`

##delete old backups
rm -f /srv/awx/backup/sql/awxenv-pgdumpall_$deleteday.sql.gz
rm -f /srv/awx/backup/sql/awxoschemaonly_$deleteday.sql.gz
rm -f /srv/awx/backup/sql/awxrolesonly_$deleteday.sql.gz
rm -f /srv/awx/backup/sql/awxtablespaces_$deleteday.sql.gz

##create a new backup
# example: pg_dumpall > /tmp/pgalldump_$today.dump.out
docker exec corkscrew2508_postgres_1 pg_dumpall -U awx |gzip -9 > /srv/awx/backup/sql/awxenv-pgdumpall_$today.sql.gz
docker exec corkscrew2508_postgres_1 pg_dumpall -U awx –schema-only |gzip -9 > /srv/awx/backup/sql/awxoschemaonly_$today.sql.gz
docker exec corkscrew2508_postgres_1 pg_dumpall -U awx –roles-only |gzip -9 > /srv/awx/backup/sql/awxrolesonly_$today.sql.gz
docker exec corkscrew2508_postgres_1 pg_dumpall -U awx –tablespaces-only |gzip -9 > /srv/awx/backup/sql/awxtablespaces_$today.sql.gz

Test/validate the record(s) of crontab (=You can run,manually): crontab -l | grep -v ‘^#’ | cut -f 6- -d ‘ ‘ | while read CMD; do eval $CMD; done

crontab:
Edit the crontab, for example, run this command as desired user (for exmample: as root) : crontab -e
01 20 * * * /usr/local/bin/awx-envbackup.sh
01 22 * * * /usr/local/bin/awx-pgsqlbackup.sh

RESTORE:
notes:
Password reset: if needed, you can reset the awx user’s password: ALTER USER user_name WITH PASSWORD ‘new_password’;

So turn to the recovery.
on host’s level: (like docker node’s shell): psql -U awx postgres -h localhost
prevent the user login via webportal (UI): stop the web container: docker stop build_image_web_1
jump to psql container and drop the (corrupt) database:
docker exec -it awx_postgres_1 bash
postgres=# drop database awx;

If the datased is used you get similar error message: database “awx” is being accessed by other users. There are 4 other sessions using the database
Stop the clients’ task(s)/job(s) of psql: there is a chance if you can’t restore the database (from backup) because the dabase is used by Customer/Colleague or technical user (like scheduled task/job/script/psql client). In this case you have to ensure if nobody is able to connect the database (meanwhile you restore the database) and you have to stop the tasks/jobs which are running at the moment. :
first prevent the new possible connections:
jump to container / or you can use native psql client on host’s level: docker exec -it awx_postgres bash
or run psql client on host’s level and connect to database: psql -U awx -d awx
alter database awx allow_connections = off;
enable extended mode: \x on
secondly: check the what kind of processes are running currently – by Customer/Colleague/technical user:
for example:
SELECT * FROM pg_stat_activity WHERE datname = ‘awx’; and
select pid as process_id,usename as username,datname as database_name,client_addr as client_address,application_name,backend_start,state,state_change from pg_stat_activity;
and stop these process(es): SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = ‘awx’; or SELECT pg_terminate_backend(PUT_PID_HERE);

If everything went well, now, you can drop the older/corrupted/actual database: postgres=# drop database awx;
Create the database and quit from sql shell:
create database awx;
CREATE DATABASE
\q
Exit from container and jump to the directory where the uncompressed sql backup is stored and run this command: docker exec -i awx_postgres psql -U awx < awxenv-pgdumpall_20201027.sql
Connect to database and ensureif everything is ok:
postgres=# \du
List of roles
Role name | Attributes | Member of
———————-+————————————————+————————————————————–
awx | Superuser, Create role, Create DB, Replication | {}

postgres=# \l
List of databases
Name | Owner | Encoding | Collate | Ctype | Access privileges
———–+——-+———-+————+————+——————-
awx | awx | UTF8 | en_US.utf8 | en_US.utf8 |
postgres | awx | UTF8 | en_US.utf8 | en_US.utf8 |
postgres=# \c awx;
awx=# \dt List of relations
Schema | Name | Type | Owner
——–+———————————————————–+——-+——-
public | auth_group | table | awx
…..

Don’t forget:
to enable the connection again: alter database awx allow_connections = on;
to start the web container which has been stopped at the beginning of this recovery process: docker start build_image_web_1
